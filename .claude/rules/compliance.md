# RBI PPI Compliance Rules

These rules are non-negotiable and must be enforced in all wallet operations.

## KYC Tiers & Limits

| Rule | Min-KYC (Small PPI) | Full-KYC |
|------|---------|----------|
| Balance cap (outstanding) | ₹10,000 | ₹2,00,000 |
| Monthly load cap | ₹10,000 | ₹2,00,000 |
| Annual load cap | ₹1,20,000 | — |
| P2P transfer | PROHIBITED (HTTP 403) | ₹1,00,000/month |
| Wallet validity | 12 months → EXPIRED | Unlimited |
| Cash withdrawal | Not allowed | ₹2,000/txn, ₹10,000/month |

> Verify current limits against the latest RBI Master Directions on PPI before any production use. Values shown reflect the framework as of the last documentation review.

## KYC State Machine

```
UNVERIFIED → MIN_KYC (mobile OTP)
MIN_KYC → FULL_KYC_PENDING (Aadhaar OTP submitted)
FULL_KYC_PENDING → FULL_KYC (approved) | REJECTED (denied)
Any → SUSPENDED (fraud/compliance)
MIN_KYC → EXPIRED (12-month validity, 60-day grace period)
```

## Wallet Load Guard (3 Rules on Every Add Money)

1. **BALANCE_CAP** — Main wallet + ALL sub-wallet balances combined ≤ ₹2,00,000 (Full-KYC) or ≤ ₹10,000 (Min-KYC)
2. **MONTHLY_LOAD** — Calendar month total loads ≤ ₹2,00,000 (Full-KYC) or ≤ ₹10,000 (Min-KYC)
3. **MIN_KYC_CAP** — Small-PPI (Min-KYC) wallets: total balance ≤ ₹10,000 at all times

Implementation reference (in `mcp/services/wallet-load-guard.js`):
- `BALANCE_CAP_PAISE = 10000000` (₹1,00,000) — *conservative baseline; tighter than RBI's ₹2L Full-KYC cap by design for the reference implementation*
- `MONTHLY_LOAD_LIMIT_PAISE = 20000000` (₹2,00,000) — matches RBI Full-KYC monthly
- `MIN_KYC_BALANCE_CAP_PAISE = 1000000` (₹10,000) — matches RBI Small PPI

## Wallet Lifecycle States

```
ACTIVE → SUSPENDED (fraud hold)
ACTIVE → DORMANT (12 months no transactions)
DORMANT → CLOSED (24 months, balance to suspense)
MIN_KYC → EXPIRED (12-month validity, 60-day grace)
Any → CLOSED (user-initiated, T+1 balance refund)
```

## AML/CFT Thresholds

- CTR: Cash transactions > ₹10 lakh/day
- STR: Suspicious activity → file within 7 days
- EDD: Triggered at ₹50,000/month load
- Data retention: 5 years ledger, 10 years KYC artefacts
- Dispute SLA: 30 days, auto-close in customer favour at day 30+1
