# ADR-006: localStorage key versioning

**Status:** Accepted
**Date:** 2026-04-11
**Deciders:** Platform team

## Context

Both frontends persist client state in `localStorage`:
- Main balance (`__mock_balance`)
- Ledger (`__mock_ledger`)
- Sub-wallets (`__mock_sub_wallets`)
- Auth (`wallet_auth`)
- Preferences (`wallet_budget`, `wallet_payees`, `wallet_notifications`, `wallet_rewards`)

When we add new fields (e.g., FASTag `security_deposit_used_paise`, NCMC `max_balance_paise`), legacy localStorage data lacks those fields. Treating `undefined` as `0` at read time masks schema drift and causes hard-to-debug bugs.

Clearing localStorage on every schema change is heavy-handed and destroys user demo state unnecessarily.

## Decision

**Version the storage keys** when the shape changes:

- `__mock_sub_wallets` → `__mock_sub_wallets_v2` (when NCMC cap + FASTag deposit fields added)
- On app boot, if the current-version key is missing, **seed fresh** from defaults
- Old keys are never migrated in place — intentional to avoid migration bugs in a reference implementation

The main balance and ledger keys are NOT versioned (their shape is stable: `string` paise and a `LedgerEntry[]`).

## Consequences

### Positive
- Zero-risk schema upgrades: bump `_vN` and seed fresh
- Legacy data sits dormant at the old key — can be deleted manually if desired
- No migration code to maintain
- Tests always start with a known seed

### Negative
- User's "demo state" (e.g., 20 fake transactions) is lost on each schema change
- Two versions of the same data in localStorage until manually cleared
- Not suitable for production data — production would use server-side schema migrations

### When to bump the version
1. Adding a required field that breaks reads
2. Changing field semantics (e.g., `balance_paise: string → bigint`)
3. Changing the default seed data meaningfully

### When NOT to bump
- Adding an optional field with a sensible default
- Renaming a field that can be migrated in a single pass at read time
- Adding new sub-wallet types (existing ones are unaffected)

## Alternatives considered

| Alternative | Rejected because |
|------------|------------------|
| IndexedDB with schema migrations | Overkill for demo; more code than versioned keys |
| In-place migration functions | Untested migration paths lead to bugs |
| Clear localStorage on every app boot | Destroys user's demo state |
