# Scope and Limitations

This document enumerates what the PPI Wallet Platform actually is, what it demonstrates, and what would change for production deployment. It exists so reviewers can evaluate the work against the right bar — as a reference implementation and AI-PM capability artifact, not as production wallet infrastructure.

## What this is

An AI-assisted reference implementation built using Claude Code as the primary engineering tool. The goal was to demonstrate:

- A compliant RBI PPI wallet architecture, end to end, covering consumer app, admin operations, MCP tool layer, and autonomous AI agents.
- How a non-engineering PM can drive a non-trivial product build using Claude Code and structured documentation as scaffolding.
- How Claude AI agents can operate safely in an ops-critical financial context (KYC chase, support triage) with clear guardrails and human escalation paths.

## What is real (correctly implemented against RBI PPI Master Directions)

| Layer | Status |
| --- | --- |
| Load Guard rules (BALANCE_CAP, MONTHLY_LOAD, MIN_KYC_CAP) | Enforced at every Add Money call; tier-aware in docs (code currently uses a flat ₹1L BALANCE_CAP; tier-differentiation is a doc-vs-code gap, see row below) |
| Sub-wallet business logic (Food, NCMC, FASTag, Gift, Fuel) | Cascade spend, isolation rules, category eligibility |
| Min-KYC → Full-KYC upgrade flow | UX flow modelled; verification is mocked |
| Monetary arithmetic | BigInt paise internally, string at API boundary |
| RBAC in admin dashboard | 6 roles, 12 permissions, enforced in route guards |
| AI agent orchestration | Intent classification, tool selection, escalation, SLA tracking |
| Idempotency pattern on transaction POSTs | Keys defined in engineering brief |

## What is mocked (would need real integration in production)

| Layer | Current state | Production requirement |
| --- | --- | --- |
| Ledger | In-memory + localStorage | Double-entry ledger in a real DB with WAL / backup |
| Escrow reconciliation | Described in `engineering-brief.md`; not runnable | Live escrow account with daily reconciliation job |
| KYC verification | UI flow only | Aadhaar eKYC, OVD capture, CKYCR lookup |
| Payment rails | Mocked UPI / DC / NB responses | PSP integration (bank partner or PA) |
| Bank settlement | Not implemented | T+1 settlement to user's bank per RBI rules |
| FIU-IND reporting (STR / CTR) | Not implemented | Automated generation and submission |
| Fraud / velocity checks | Not implemented | Device fingerprinting, velocity thresholds, rule engine |
| DPDP Act 2023 consent flows | Not modelled | Consent capture, data fiduciary registration, retention |
| Auth | Hardcoded demo credentials | OAuth2 / OIDC, MFA, token rotation |
| Secrets management | `.env` | HSM or secrets vault (e.g., AWS KMS, HashiCorp Vault) |
| Rate limiting | Not implemented | Per-user and per-endpoint on all mutating calls |
| Transaction commit path | ✅ `processLoad()` atomic validate+commit with idempotency-key dedupe now exists (`mcp/services/wallet-load-guard.js`) | Production: back the in-process lock + in-memory key store with Postgres row locks + Redis key store with TTL |
| Cascade-spend as a discrete service | ✅ Pure `calculateCascadeSpend(input)` function exists (`paytm-wallet-app/src/services/cascade-spend.ts`) with 13 invariant tests | Production: reuse same pure function server-side; wire the returned plan to a real ledger write |
| Tier-differentiated BALANCE_CAP | Code enforces flat ₹1L regardless of KYC tier (`BALANCE_CAP_PAISE = 10000000`); docs describe tier-aware RBI policy | Per-tier caps: ₹2L Full-KYC, ₹10K Min-KYC, with Load Guard reading the user's tier at check time |
| Timezone-aware monthly-load reset | ✅ `getMonthlyLoadTotal({loads, month, timezone})` + timezone-aware `getMonthlyLoadedPaise(userId, 'Asia/Kolkata')` now exist | Production: plumb explicit timezone through all callers; default to IST |
| Concurrency / racing load requests | Single-process in-memory lock in `processLoad()` — sufficient for this reference impl's scale | Postgres `SELECT ... FOR UPDATE` or Redis distributed lock on `(user_id, op_type)`; idempotency key store with TTL (design preserved so the API surface doesn't change) |

## Known anti-patterns deliberately used for demo purposes

1. **AI support agent receives balance/transactions from the frontend.** This is a UX trick so the AI matches what the user sees — but in production the server must be the authoritative source. The agent architecture would need to invert: fetch fresh balance from ledger, never trust client state. See [ADR-007](adr/ADR-007-ai-context-sync.md) for the full tradeoff.
2. **Hardcoded admin credentials** (`admin/admin123` etc.) for the admin dashboard. Demo-only; would be replaced with SSO + RBAC-integrated auth.
3. **localStorage-backed Zustand persistence.** Fine for a client-side demo; would be session + server-backed in production with no sensitive state ever written to browser storage.
4. **HashRouter** for GitHub Pages compatibility. Would be BrowserRouter in production behind a proper web server.
5. **Seed data includes real employer names** (Paytm, TCS). Acceptable for internal demos; would be neutralised for any external-facing presentation.

## What would change on the architecture side for production

- Add an OpenAPI 3.1 specification with versioned contract, validated in CI.
- Move from Express single-process to a horizontally scalable API tier behind a load balancer.
- Introduce a message queue (Kafka / SQS) for async event handling — particularly agent triggers and alert dispatch.
- Move agents from in-process cron to a dedicated worker tier with distributed locking.
- Add observability: structured logs, traces (OpenTelemetry), metrics (Prometheus), and SLO dashboards.
- Factor `calculateCascadeSpend`, `validateMerchantSource`, and `commitLoad` into pure functions so they can be unit-tested and shared between client and server.

## What would change on the compliance side for production

- Live legal and compliance review of every Load Guard rule against the then-current RBI circular text, not summarised paraphrases.
- Formal PMLA policy document, officer designation, and STR/CTR workflow.
- Integration with CKYCR for KYC record upload.
- DPDP Act 2023 compliance: consent, purpose limitation, data fiduciary obligations, grievance officer.
- PCI-DSS scope review if card numbers are ever touched (currently out of scope — card-on-file and payment instrument details go via PSP tokenisation).
- Annual information security audit and penetration test against CERT-In guidelines.

## Adversarial test findings (see `edge-cases.md` for the full 68-case catalog)

The adversarial test pass (2026-04-17) surfaced four code gaps. **Three of four are now closed** (2026-04-17 later in the day), leaving one outstanding:

1. ~~**No atomic `processLoad`**~~ ✅ **CLOSED.** `processLoad({userId, amountPaise, idempotencyKey, apiKey})` shipped in `mcp/services/wallet-load-guard.js`. Atomic validate+commit under per-user serialization lock with 24h idempotency-key cache. 10 tests in `mcp/agents/__tests__/process-load-concurrency.test.js`.

2. ~~**No cascade-spend pure function**~~ ✅ **CLOSED.** `calculateCascadeSpend(input) → {splits, status, reason}` shipped in `paytm-wallet-app/src/services/cascade-spend.ts`. Takes immutable input, returns a plan, never writes. 13 tests in `src/services/__tests__/cascade-spend.test.ts` enforce: category-specific priority, Gift fallback rules, expired-Gift exclusion, clean decline with zero partial-debit, no-zero-amount-splits.

3. **No NCMC hard-reject service-boundary tests** — **still open.** The isolation invariants (NCMC cannot P2P, NCMC cannot pay non-transit merchants) are guaranteed by omission in the UI and the mockCheckEligibility function, but not enforced at a `transactions` service layer because no such service exists yet. Mitigation: when a `transactions` service is extracted with explicit source-wallet parameters, add explicit rejection tests. The current eligibility tests exercise the equivalent invariants through `mockCheckEligibility`.

4. ~~**Monthly-load timezone behavior is ambiguous**~~ ✅ **CLOSED.** `getMonthlyLoadTotal({loads, month, timezone})` shipped in `mcp/services/wallet-load-guard.js` using `Intl.DateTimeFormat` for correct IST boundaries. `getMonthlyLoadedPaise()` now delegates with default `'Asia/Kolkata'`. 7 tests exercise the 23:30-IST / 00:30-IST boundary. Render's UTC hosting no longer mis-classifies IST month boundaries.

## How to read the rest of the documentation

Treat the documents in `docs/` as **PM-authored specifications** describing what a production PPI wallet should do. Treat the code in the sibling repos as **a runnable illustration** of those specifications against mock data. The gap between spec and runnable code is enumerated in the tables above.
