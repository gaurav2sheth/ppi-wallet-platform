# ADR-004: FASTag as security deposit model

**Status:** Accepted
**Date:** 2026-04-11
**Deciders:** Platform team

## Context

FASTag is not a typical sub-wallet. At toll plazas, the tag must respond with available balance within ~150ms — any deep-link to a hosted wallet API risks timeout. Two common models:

- **Isolated balance** (like NCMC): user loads FASTag explicitly; toll deducts from FASTag.
- **Main-wallet-backed**: toll deducts from main wallet; FASTag is just metadata.

Neither is complete:
- Isolated balance requires users to manage two pots
- Main-wallet-backed fails when main is ₹0 during a toll

## Decision

Use a **security-deposit model**:

- Sub-wallet holds `₹300 × vehicle_count` as a **security deposit**
- **Toll deducts from main wallet FIRST**
- If main wallet is ₹0 → **security deposit is consumed**
- On next Add Money to the FASTag page:
  1. Refill security deposit first (up to `₹300 × vehicles`)
  2. Remainder flows to main wallet
- New vehicle = additional ₹300 deposit from main wallet

Security deposit acts as a safety net for cases when a user genuinely has ₹0 during a toll but still has a funded FASTag on-vehicle.

## Consequences

### Positive
- Tolls almost always succeed (main wallet has money OR deposit is available)
- Users don't need to think about FASTag balance — it "just works"
- Ops can audit deposit shortfalls as a proxy for "user ran low"
- On refill, deposit is made whole before main wallet grows — self-healing

### Negative
- Model is non-obvious — documented via in-app info box and [sub-wallets.md](../../.claude/rules/sub-wallets.md)
- Deposit shortfall creates a ₹300 liability that must be tracked per vehicle
- `security_deposit_used_paise` is a new schema field; legacy entries need migration
- Post-expiry vehicles — when the user removes a vehicle, the deposit must return to main wallet (handled in `mockRemoveFastagVehicle`, but needs an RBI-compliant reconciliation journal in production)

### Edge cases covered in [edge-cases.md](../edge-cases.md)
- Toll when main=₹0 and deposit=₹300 → deposit consumed
- Toll when main=₹0 and deposit=₹0 → hard decline, specific error code
- Add Money ₹500 when deposit_used=₹100 → ₹100 refill + ₹400 to main
- Add Money ₹50 when deposit_used=₹100 → full ₹50 goes to deposit refill, main gets ₹0

## Alternatives considered

| Alternative | Rejected because |
|------------|------------------|
| Isolated balance | Users would forget to top up FASTag separately → toll failures |
| Main-only | No safety net when main=₹0 |
| Credit-line model (auto-debit later) | RBI PPI rules prohibit credit features |
