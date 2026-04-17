# ADR-005: NCMC isolated balance (no cascade)

**Status:** Accepted
**Date:** 2026-04-11
**Amended:** 2026-04-16 — added direct load from UPI/DC/NB
**Deciders:** Platform team

## Context

NCMC (National Common Mobility Card) is a government-mandated interoperable transit payment protocol. Unlike FASTag (tolls online), NCMC readers on metros/buses are frequently **offline** — they validate against a balance stored on the card itself, synced periodically.

This means:
- A cascade to main wallet during a metro tap is not feasible (no network)
- The NCMC pot must behave like a closed-loop stored-value instrument
- Balance must be capped to limit loss exposure if the card is lost

## Decision

NCMC has its own **₹3,000 balance cap** and is **isolated**:

- Transit payments (Metro, Bus, Local train) deduct **only from NCMC balance**
- **No cascade to main wallet** — if NCMC balance < fare, payment fails
- Non-transit merchant pay cannot use NCMC (hard reject with specific error)
- P2P transfer cannot originate from NCMC
- Load sources (amended 2026-04-16):
  - Main Wallet (deducts from main, adds to NCMC)
  - UPI / Debit Card / Net Banking (direct external load, bypasses main wallet)

## Consequences

### Positive
- Matches real-world NCMC reader behavior (card-local balance)
- Loss exposure capped at ₹3,000 if physical card lost
- User can hold NCMC balance even with ₹0 main (metro still works)
- Direct load enables "one-click NCMC top-up from UPI" without touching main wallet

### Negative
- Users see a fare failure if NCMC is ₹0 even when main has ₹50K — educational burden via in-app info box
- Dual-cap enforcement: NCMC ₹3K + combined RBI ₹1L (see ADR-003)
- Direct load via UPI still counts toward combined RBI cap (aggregated on load)

### Edge cases covered in [edge-cases.md](../edge-cases.md)
- Metro tap when NCMC=₹10 and fare=₹30 → fail, specific error code
- Attempt to P2P from NCMC → hard reject
- Attempt to pay Swiggy from NCMC → hard reject (category mismatch)
- Direct UPI load when NCMC=₹2,900 and amount=₹500 → reject (would exceed ₹3K cap)

## Alternatives considered

| Alternative | Rejected because |
|------------|------------------|
| Cascade to main wallet | Reader offline → cannot query main wallet |
| Shared balance with main | Loss exposure = full wallet if card stolen |
| Separate NCMC RBI cap from wallet cap | Regulatory clarity favors aggregation |
