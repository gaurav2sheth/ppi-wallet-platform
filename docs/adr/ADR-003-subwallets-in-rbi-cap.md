# ADR-003: Sub-wallet balances count toward the ₹1L RBI cap

**Status:** Accepted
**Date:** 2026-04-11
**Deciders:** Platform team + compliance review

## Context

RBI's PPI master direction caps Full-KYC wallet balance at ₹2,00,000 and Min-KYC at ₹10,000. Our platform adds 5 sub-wallets (Food, NCMC Transit, FASTag, Gift, Fuel). Open question:

- Do sub-wallet balances count toward the RBI cap?
- Or do they sit outside as a separate logical account?

The "outside" interpretation would let a Full-KYC user hold ₹2L main + ₹3K Food + ₹3K NCMC + ₹300 FASTag + ₹90K Gift + ₹2.5K Fuel = ₹2.98L effective, exceeding the cap.

## Decision

**Sub-wallet balances count toward the RBI cap.**

Concretely:
- `BALANCE_CAP` Load Guard rule: `main + Σ(sub_wallets) ≤ ₹1,00,000` per Min-KYC policy at this tier
- `MIN_KYC_CAP` Load Guard rule: for Min-KYC, same but `≤ ₹10,000`
- Balance cap check runs on every Add Money and every sub-wallet load

## Consequences

### Positive
- Single cap ceiling → easier for users to understand and for compliance to audit
- Prevents regulatory arbitrage (stacking sub-wallets to exceed main cap)
- Cap math is simple: `total_system_balance ≤ tier_cap`

### Negative
- Limits benefit-wallet flexibility — employer can't load unlimited Gift without user feeling main cap pressure
- Load Guard must aggregate all balances on every top-up — adds one read per sub-wallet
- Corner case: employer loads Gift when main is near cap → must reject with clear error (see [edge-cases.md](../edge-cases.md))

### Regulatory alignment
- Conservative interpretation matches RBI's intent (prevent wallet stacking)
- If RBI later clarifies differently, we relax — harder to tighten after launch

## Alternatives considered

| Alternative | Rejected because |
|------------|------------------|
| Sub-wallets outside cap | Regulatory risk; arbitrage |
| Per-wallet-type sub-caps only | Too permissive; total could exceed main-cap |
| Only employer-loaded sub-wallets exempt | Inconsistent UX; user can't predict behavior |
