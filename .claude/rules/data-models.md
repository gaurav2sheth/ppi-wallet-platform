# Data Models & Storage

## Core TypeScript Interfaces (paytm-wallet-app/src/types/api.types.ts)

### LedgerEntry (append-only transaction record)
```
id, entry_type (CREDIT|DEBIT|HOLD|HOLD_RELEASE), amount_paise (string),
balance_after_paise, held_paise_after, transaction_type, reference_id,
description, idempotency_key, hold_id, created_at, payment_source?
```
Transaction types: ADD_MONEY, MERCHANT_PAY, P2P_TRANSFER, WALLET_TO_BANK, BILL_PAY, REFUND

### SubWallet (mock.ts)
```
sub_wallet_id, type, icon, color, label, balance_paise (number),
status, monthly_limit_paise, monthly_loaded_paise, loaded_by,
last_loaded_at, expiry_date, eligible_categories[], transactions[],
max_balance_paise? (NCMC), is_security_deposit? (FASTag),
vehicle_count?, security_deposit_per_vehicle_paise?, security_deposit_used_paise?
```

### SagaResponse
```
saga_id, saga_type, status (STARTED|RUNNING|COMPLETED|COMPENSATING|COMPENSATED|DLQ),
result? { balance_after_paise, entry_id }, error?, steps[], created_at, updated_at
```

## localStorage Keys

| Key | Content |
|-----|---------|
| `__mock_balance` | Main wallet balance in paise (string) |
| `__mock_ledger` | Transaction history (LedgerEntry[]) |
| `__mock_sub_wallets_v2` | Sub-wallet data with NCMC/FASTag fields |
| `wallet_pin` | 4-digit PIN (default: 1234) |
| `wallet_auth` | Login state (walletId, userId, userName, phone) |
| `wallet_budget` | Budget category limits |
| `wallet_payees` | Recent contacts by type (max 20 each) |
| `wallet_notifications` | Alert messages |
| `wallet_rewards` | Scratch cards + cashback |

## Default Seed Data
- Main wallet: ₹236.11 (23611 paise)
- 12 historical transactions spanning 20 days
- 5 sub-wallets: Food ₹1,200, NCMC ₹1,800, FASTag ₹600 (2 vehicles), Gift ₹2,000, Fuel ₹1,500

## Currency Formatting
```typescript
formatPaise(paise: string | number): string
// Returns: ₹X,XX,XXX.XX using en-IN locale
```
Always use `formatPaise()` from `src/utils/format.ts`. Never format manually.
