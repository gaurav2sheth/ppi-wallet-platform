# Sub-Wallet System (Corporate Benefits)

5 benefit wallet types, each with distinct rules. Sub-wallet balances count toward the ₹1L RBI cap.

## Food Wallet 🍱
- **Employer-loaded only** — no self-load
- Monthly limit: ₹3,000
- Eligible: Restaurants, Cafes, Food delivery, Swiggy, Zomato, Canteen
- Spending deducts directly from Food wallet balance

## NCMC Transit 🚇
- **Own ₹3,000 BALANCE LIMIT** (max balance at any time, not monthly)
- **Multiple load sources**: Main Wallet, UPI, Debit Card, or Net Banking
  - Main Wallet: deducts from main balance, transfers to NCMC
  - UPI/DC/NB: loads directly into NCMC without affecting main wallet balance
- Transit payments (Metro, Bus, Local train) use **ONLY NCMC balance**
- **No cascade to main wallet** — if NCMC balance insufficient, payment fails
- Interface field: `max_balance_paise: 300000`
- Direct load function: `mockDirectLoadNcmc(amountPaise, paymentSource)` — bypasses main wallet

## FASTag 🛣️ (Security Deposit Model)
- Sub-wallet holds **₹300 security deposit per vehicle**
- FASTag toll transactions **deduct from MAIN WALLET first**
- If main wallet is zero → security deposit is consumed
- **Add Money via UPI, Debit Card, or Net Banking** (payment source selectable):
  1. Refill security deposit first (up to ₹300 × vehicles)
  2. Remainder goes to main wallet
- "New Vehicle" = issue FASTag + ₹300 deposit from main wallet
- Interface fields: `is_security_deposit: true`, `vehicle_count`, `security_deposit_per_vehicle_paise: 30000`, `security_deposit_used_paise`

## Gift Wallet 🎁
- Employer-loaded with expiry date (typically 1 year)
- Self-load from main wallet allowed (no cap)
- Universal merchant eligibility — usable everywhere
- Shows EXPIRED badge when past expiry_date

## Fuel Wallet ⛽
- **Employer-loaded only** — no self-load
- Monthly limit: ₹2,500
- Eligible: HP, Indian Oil, BPCL, Shell stations only

## Cascade Spend Logic (Merchant Pay)
1. Auto-detect merchant category via MCC keyword matching
2. Check specific wallets first (Food/Transit/Fuel) based on category
3. Gift wallet as universal fallback
4. If sub-wallet balance < payment amount → split: sub-wallet + remainder from main wallet

## localStorage Key
`__mock_sub_wallets_v2` — v2 includes NCMC balance cap + FASTag security deposit fields. Increment version to force re-seed on schema changes.
