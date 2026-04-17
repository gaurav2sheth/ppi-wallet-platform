# ADR-011: Monetary arithmetic in paise with BigInt

**Status:** Accepted
**Date:** 2026-01 (approximate — baseline platform convention, promoted to ADR 2026-04-17)
**Deciders:** Platform team

## Context

Wallet amounts must be precise. INR values expressed as floating-point numbers (e.g. `99.99`) suffer from IEEE-754 representation errors — `0.1 + 0.2 !== 0.3` — which in a financial system compounds over millions of transactions into real money lost or created. The industry-standard pattern is to store the smallest currency unit as an integer; for INR this is paise (1 INR = 100 paise).

A secondary consideration is scale. Large corporate payouts or aggregated settlements can exceed JavaScript's `Number.MAX_SAFE_INTEGER` (2^53 − 1 paise ≈ ₹9 × 10^13, or roughly ₹90 trillion). While a single user will never hit this, aggregate queries (total monthly load across all users, employer bulk loads, escrow settlement totals) can.

## Decision

All monetary values are stored as **integers in paise**, using JavaScript `BigInt` for internal arithmetic inside the MCP / server layer. Values are serialised as decimal strings at API boundaries. Display formatting converts back to rupees with two decimal places using `en-IN` locale formatting via a shared `formatPaise()` helper.

Concretely:
- Mock layer in `paytm-wallet-app/src/api/mock.ts`: `Number` paise for single-user math (always < `MAX_SAFE_INTEGER` in practice), strings at the localStorage boundary.
- Server / MCP in `mcp/mock-data.js`: `BigInt` for aggregates, strings across the wire.
- API responses always use strings for paise values to prevent JSON number coercion.
- Display via `formatPaise(paise: string | number): string` returning `"₹X,XX,XXX.XX"`.

## Consequences

### Positive
- No floating-point precision errors anywhere in the stack.
- No silent overflow on large aggregations (server-side BigInt).
- Matches the pattern used by most production payment systems (Stripe, Razorpay, PayTM internals).
- API contract is self-documenting — any `*_paise` field is an integer string.

### Negative
- BigInt cannot be `JSON.stringify`'d natively — requires explicit `.toString()` at serialisation boundaries. This is a discipline, not a feature.
- Test fixtures and mock data must use paise, which reads awkwardly (`balance: 20000000` rather than `balance: 200000.00`). Mitigated by a `rupeesToPaise()` helper in test utilities.
- Frontend display code must never do arithmetic on displayed values — only on underlying paise integers. Enforced by code review since TypeScript can't express "this `string` is formatted output."
- Mixed `Number` paise (wallet app) and `BigInt` paise (MCP) means the boundary must convert explicitly — a common bug site.

## Alternatives considered

- **Float with rounding at boundaries.** Rejected: precision errors still accumulate in intermediate calculations. Even with rounding at the final step, interest accrual or cascade-spend math can drift mid-computation.
- **Decimal string throughout.** Rejected: requires a decimal arithmetic library (e.g. `decimal.js`), adds a dependency, and slows arithmetic-heavy code paths like batch settlement calculations.
- **Number (integer paise, no BigInt).** Rejected on overflow grounds for aggregate queries — the `get_system_stats` tool sums balances across 200 seed users and easily crosses safe-integer territory when scaled.
- **Storage in rupees with fixed-point arithmetic.** Rejected: inherits all the floating-point risks with extra boilerplate.

## References

- Platform `CLAUDE.md` — "Conventions" section: "All monetary values: integers in paise (1 INR = 100 paise). Never use floats for money."
- MCP repo `CLAUDE.md`: "All balances use BigInt arithmetic internally, serialised as strings at API boundary."
- [ADR-003](ADR-003-subwallets-in-rbi-cap.md) — cap math depends on precise paise arithmetic at the aggregate level.
- [ADR-007](ADR-007-ai-context-sync.md) — context-sync's `balance_paise` field documented as a string.
- `paytm-wallet-app/src/utils/format.ts` — `formatPaise()` implementation.
