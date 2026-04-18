# PPI Wallet Platform -- Product Requirements Document v1.1

> **Regulatory numbers in this PRD reference** [`compliance-reference.md`](compliance-reference.md), **not a local copy.** Where specific ₹ amounts appear inline (balance caps, monthly limits, P2P aggregates, etc.), they are summaries of values in that reference doc. If this PRD and the reference doc diverge, the reference doc wins.

---

## 1. Product Overview

Build an RBI-regulated PPI (Prepaid Payment Instrument) Wallet system with:

- **Consumer Wallet App** -- Mobile-first React SPA for end users (add money, pay merchants, P2P transfer, bank transfer, bill pay, passbook, analytics, rewards, KYC, sub-wallets)
- **Admin Dashboard** -- Desktop-first React SPA for operations teams (user management, transaction monitoring, KYC queue, spend analytics, benefits management, AI insights)
- **Backend API** -- Express.js REST API with Claude AI integration (chat, summarisation, KYC alerts, load guard)
- **MCP Server** -- 49 Claude AI tools for natural language wallet operations

All monetary values are stored and transmitted as **integers in paise** (1 INR = 100 paise). Never use floating-point for money.

---

## 2. Tech Stack

### Wallet App

| Layer | Technology |
|-------|-----------|
| Framework | React 19 + TypeScript |
| Bundler | Vite 8 |
| Styling | Tailwind CSS v4 with Paytm PODS design tokens |
| Routing | React Router v7 (HashRouter for GitHub Pages) |
| State | Zustand with localStorage persistence |
| HTTP | Axios with 3-tier fallback (Vite middleware, Express API, client-side mock) |
| Icons | Inline SVGs (no icon library) |

### Admin Dashboard

| Layer | Technology |
|-------|-----------|
| Framework | React 19 + TypeScript |
| UI Library | Ant Design v5 |
| Styling | Ant Design theme tokens + custom CSS |
| Routing | React Router v7 (HashRouter) |
| State | Zustand |

### Backend API

| Layer | Technology |
|-------|-----------|
| Runtime | Node.js + Express.js |
| AI | Anthropic Claude API (claude-sonnet-4-20250514 for chat, claude-haiku-4-5-20251001 for alerts) |
| CORS | Whitelist GitHub Pages + localhost |

### MCP Server

| Layer | Technology |
|-------|-----------|
| Protocol | Model Context Protocol (MCP) via stdio transport |
| Validation | Zod schemas for all tool inputs |
| Data | In-memory mock data (200 users, 500+ transactions) seeded with deterministic PRNG |

---

## 3. Design System -- Paytm PODS

### Colour Palette

| Token | Hex | Usage |
|-------|-----|-------|
| Navy | #002E6E | Primary, headers, CTAs |
| Cyan | #00B9F1 | Links, accents, secondary actions |
| Green | #12B76A | Success, credits, active badges |
| Orange | #F97316 | Warnings, Food wallet |
| Red | #E53935 | Errors, debits, destructive actions |
| Text | #1A1A2E | Body text |
| Muted | #707070 | Secondary text, labels |
| Border | #E8E8E8 | Card borders, dividers |
| Background | #F5F7FA | Page background |

### Component Patterns

- Card-based layouts with rounded corners (xl = 12px)
- Pill-shaped buttons and badges
- Bottom navigation with 4 tabs: Home, Scan & Pay, Passbook, Profile
- Page transitions with `page-enter` animation class
- Mobile viewport: max-width 480px centred on desktop
- Loading states: skeleton pulse animations
- Amount display: rupee prefix, Indian locale formatting (1,00,000.00)

---

## 4. RBI Compliance Rules (Non-Negotiable)

### 4.1 KYC Tiers and Limits

| Rule | Min-KYC | Full-KYC |
|------|---------|----------|
| Balance cap | 10,000 INR | 2,00,000 INR |
| Monthly load cap | 10,000 INR | 1,00,000 INR |
| Annual load cap | 1,20,000 INR | No limit |
| P2P transfer | PROHIBITED | 1,00,000 INR per month |
| Wallet validity | 12 months (then EXPIRED) | Unlimited |
| Cash withdrawal | Not allowed | 2,000 INR per transaction, 10,000 INR per month |

### 4.2 KYC State Machine

The KYC verification follows a strict state machine with the following transitions:

```
UNVERIFIED --> MIN_KYC (mobile OTP verification completed)
MIN_KYC --> FULL_KYC_PENDING (Aadhaar OTP submitted for verification)
FULL_KYC_PENDING --> FULL_KYC (verification approved by admin/compliance)
FULL_KYC_PENDING --> REJECTED (verification denied by admin/compliance)
Any state --> SUSPENDED (fraud detection or compliance hold triggered)
MIN_KYC --> EXPIRED (12-month validity period lapsed, 60-day grace period begins)
```

**State Transition Details:**

- **UNVERIFIED to MIN_KYC:** User completes mobile number verification via OTP. This is the minimum required to use the wallet with restricted limits.
- **MIN_KYC to FULL_KYC_PENDING:** User initiates Aadhaar-based verification by submitting Aadhaar number and completing Aadhaar OTP. The request enters the KYC approval queue.
- **FULL_KYC_PENDING to FULL_KYC:** An operations manager or compliance officer approves the KYC request after document verification. The user gains full wallet capabilities.
- **FULL_KYC_PENDING to REJECTED:** The KYC request is denied. The user remains at MIN_KYC tier and may re-submit.
- **Any to SUSPENDED:** Triggered by fraud detection systems, suspicious transaction reports, or manual compliance holds. The wallet is frozen -- no transactions allowed until review is complete.
- **MIN_KYC to EXPIRED:** If the wallet has been in MIN_KYC state for 12 months without upgrading to Full KYC, the wallet enters EXPIRED state. A 60-day grace period allows the user to complete Full KYC or withdraw remaining balance.

### 4.3 Wallet Load Guard (3 Rules)

These three rules are enforced on **every** Add Money operation, in order. All three must pass for the load to succeed.

#### Rule 1: BALANCE_CAP

- **Description:** Total wallet balance (main wallet + all sub-wallet balances combined) must not exceed 1,00,000 INR after the load.
- **Check:** `(current_main_balance + sum_of_all_sub_wallet_balances + load_amount) <= 1,00,000 INR`
- **Failure message:** "Adding this amount would exceed the RBI maximum wallet balance of 1,00,000 INR."
- **Maximum allowed load:** `1,00,000 INR - current_total_balance`

#### Rule 2: MONTHLY_LOAD

- **Description:** Calendar month total loads (all loads within the current calendar month) must not exceed 2,00,000 INR.
- **Check:** `(total_loads_this_month + load_amount) <= 2,00,000 INR`
- **Failure message:** "Adding this amount would exceed the monthly load limit of 2,00,000 INR."
- **Resets:** On the 1st of each month at 00:00 IST.

#### Rule 3: MIN_KYC_CAP

- **Description:** For Min-KYC wallets only -- total balance must not exceed 10,000 INR.
- **Check:** If `kyc_tier == 'MINIMUM'`, then `(current_total_balance + load_amount) <= 10,000 INR`
- **Failure message:** "Min-KYC wallets cannot hold more than 10,000 INR. Please upgrade to Full KYC."
- **Applies to:** Only wallets with KYC tier = MINIMUM. Full-KYC wallets are exempt from this rule (Rule 1 applies instead).

**Load Guard API Endpoint:** `POST /api/wallet/validate-load`

- Request body: `{ user_id, amount_paise, kyc_tier }`
- Response: `{ allowed: boolean, violated_rules: string[], max_allowed_paise: number, suggestions: string[] }`
- All blocked load attempts are logged to `GET /api/wallet/load-guard-log` for audit purposes.

### 4.4 Wallet Lifecycle States

```
ACTIVE --> SUSPENDED (fraud/compliance hold)
ACTIVE --> DORMANT (12 months with no transactions)
DORMANT --> CLOSED (24 months of inactivity, balance moved to suspense account)
MIN_KYC --> EXPIRED (12-month validity, 60-day grace period)
Any state --> CLOSED (user-initiated closure, T+1 balance refund via NEFT/IMPS)
```

**Lifecycle State Details:**

- **ACTIVE:** Normal operating state. All transactions permitted based on KYC tier.
- **SUSPENDED:** Wallet frozen due to fraud detection, suspicious transaction report, or manual compliance hold. No transactions permitted. Requires compliance officer review to reactivate.
- **DORMANT:** Automatically triggered after 12 consecutive months with zero transactions. User is notified and can reactivate by performing any transaction.
- **CLOSED (inactivity):** After 24 months of dormancy, the wallet is permanently closed. Any remaining balance is transferred to an RBI-mandated suspense account.
- **EXPIRED (Min-KYC only):** The 12-month validity window for Min-KYC wallets has lapsed. The user has a 60-day grace period to either upgrade to Full KYC or withdraw the remaining balance.
- **CLOSED (user-initiated):** User requests wallet closure. Remaining balance is refunded via NEFT or IMPS on T+1 (next business day).

### 4.5 Other Compliance Requirements

#### Cash Transaction Report (CTR)
- **Trigger:** Mandatory filing for cash transactions exceeding 10,00,000 INR (10 lakh) in a single day.
- **Filed to:** Financial Intelligence Unit -- India (FIU-IND).
- **Timeline:** Filed in monthly batches.

#### Suspicious Transaction Report (STR)
- **Trigger:** Any transaction pattern flagged as suspicious by fraud detection or manual review.
- **Filing deadline:** Must be filed within 7 days of detection.
- **Filed to:** FIU-IND.
- **Internal process:** The `flag_suspicious_transaction` MCP tool flags transactions for review; compliance officers then determine if STR filing is required.

#### Enhanced Due Diligence (EDD)
- **Trigger:** Automatically triggered when a user's monthly load exceeds 50,000 INR.
- **Actions:** Additional identity verification, source-of-funds documentation, enhanced monitoring.

#### Data Retention Policy
- **Ledger and administrative records:** Retained for a minimum of 5 years.
- **KYC artefacts (Aadhaar details, verification records):** Retained for a minimum of 10 years.
- **All data must be localised to India** -- no cross-border data transfer.

#### Dispute Resolution SLA
- **Maximum resolution time:** 30 days per RBI guidelines.
- **Auto-close rule:** If a dispute is not resolved within 30 days, it is automatically closed in the customer's favour on day 30+1.
- **Internal tracking:** Disputes are tracked via `raise_dispute`, `get_dispute_status` MCP tools and the admin transaction detail page.

---

## 5. Data Models

### 5.1 Wallet Balance Response

```typescript
interface WalletBalanceResponse {
  success: boolean;
  wallet_id: string;
  user_id: string;
  balance_paise: string;     // BigInt as string to avoid precision loss
  held_paise: string;        // Amount held for pending transactions
  available_paise: string;   // balance_paise - held_paise (usable balance)
  kyc_tier: 'MINIMUM' | 'FULL';
  is_active: boolean;
  updated_at: string;        // ISO 8601 timestamp
}
```

**Field Details:**

- `balance_paise`: The total wallet balance in paise, represented as a string to safely handle large numbers without floating-point precision issues. All arithmetic should parse this to BigInt or integer before calculations.
- `held_paise`: Funds currently held/locked for pending transactions (e.g., a payment that has been initiated but not yet completed). These funds cannot be used for new transactions.
- `available_paise`: The actual usable balance, calculated as `balance_paise - held_paise`. This is the amount shown to the user as their available balance.
- `kyc_tier`: Determines which compliance rules apply. 'MINIMUM' = Min-KYC (restricted limits), 'FULL' = Full-KYC (higher limits, P2P enabled).
- `is_active`: Whether the wallet is in ACTIVE state. False for SUSPENDED, DORMANT, EXPIRED, or CLOSED wallets.

### 5.2 Ledger Entry (Append-Only)

```typescript
interface LedgerEntry {
  id: string;                // UUID v4 - unique identifier for this entry
  entry_type: 'CREDIT' | 'DEBIT' | 'HOLD' | 'HOLD_RELEASE';
  amount_paise: string;      // Amount of this entry in paise
  balance_after_paise: string;  // Running balance after this entry
  held_paise_after: string;     // Running held amount after this entry
  transaction_type: 'ADD_MONEY' | 'MERCHANT_PAY' | 'P2P_TRANSFER' | 'WALLET_TO_BANK' | 'BILL_PAY' | 'REFUND';
  reference_id: string | null;  // External reference (e.g., UPI ref, merchant txn ID)
  description: string;          // Human-readable description
  idempotency_key: string;      // Client-generated UUID for deduplication
  hold_id: string | null;       // Links HOLD_RELEASE entries to their original HOLD
  created_at: string;           // ISO 8601 timestamp
  payment_source?: string;      // e.g. "UPI - HDFC Bank 7125"
}
```

**Entry Type Details:**

- **CREDIT:** Money added to the wallet (Add Money, refunds, cashback).
- **DEBIT:** Money removed from the wallet (payments, transfers, withdrawals).
- **HOLD:** Funds temporarily locked for a pending transaction. The `held_paise_after` increases.
- **HOLD_RELEASE:** A previously held amount is released (either converted to a DEBIT on success, or returned to available balance on failure). The `hold_id` field links back to the original HOLD entry.

**Append-Only Principle:** The ledger is strictly append-only. Entries are never modified or deleted. Corrections are made by adding new compensating entries (e.g., a CREDIT entry for a refund). This provides a complete audit trail for regulatory compliance.

**Transaction Types:**

| Type | Direction | Description |
|------|-----------|-------------|
| ADD_MONEY | CREDIT | User loads money into wallet from bank/UPI |
| MERCHANT_PAY | DEBIT | Payment to a merchant (QR code, merchant ID) |
| P2P_TRANSFER | DEBIT (sender) / CREDIT (receiver) | Peer-to-peer transfer between wallet users |
| WALLET_TO_BANK | DEBIT | Withdrawal from wallet to bank account |
| BILL_PAY | DEBIT | Utility bill, recharge, or subscription payment |
| REFUND | CREDIT | Reversal of a previous debit transaction |

### 5.3 Saga Response (Transaction Result)

```typescript
interface SagaResponse {
  saga_id: string;           // Unique identifier for this saga
  saga_type: string;         // e.g., 'ADD_MONEY', 'MERCHANT_PAY'
  status: 'STARTED' | 'RUNNING' | 'COMPLETED' | 'COMPENSATING' | 'COMPENSATED' | 'DLQ';
  result?: {
    balance_after_paise?: string;  // New balance after transaction
    entry_id?: string;             // Ledger entry ID created
  };
  error?: string;            // Error message if failed
  steps: SagaStep[];         // Individual saga steps with status
  created_at: string;        // ISO 8601
  updated_at: string;        // ISO 8601
}
```

**Saga Status Details:**

- **STARTED:** Saga has been initiated but no steps have executed yet.
- **RUNNING:** One or more steps are currently executing.
- **COMPLETED:** All steps executed successfully. The transaction is finalised.
- **COMPENSATING:** A step failed and the system is rolling back previous steps via compensating transactions.
- **COMPENSATED:** All compensating transactions have completed. The system is back to its pre-saga state.
- **DLQ (Dead Letter Queue):** Compensation itself failed. Requires manual intervention by operations team. These are surfaced in the admin dashboard's failed transactions view.

### 5.4 Sub-Wallet Data Model

```typescript
interface SubWallet {
  sub_wallet_id: string;        // Unique identifier
  type: 'FOOD' | 'NCMC TRANSIT' | 'FASTAG' | 'GIFT' | 'FUEL';
  icon: string;                 // Emoji representation
  color: string;                // Hex colour code for UI
  label: string;                // Display name
  balance_paise: number;        // Current balance
  status: 'ACTIVE' | 'FROZEN' | 'EXPIRED';
  monthly_limit_paise: number;  // Maximum monthly load (0 = no limit)
  monthly_loaded_paise: number; // Amount loaded this month
  loaded_by: string;            // Who loads this wallet (e.g., "Employer - TCS")
  last_loaded_at: string;       // ISO 8601 timestamp of last load
  expiry_date: string | null;   // Expiry date for Gift wallets
  eligible_categories: string[]; // Merchant categories eligible for this wallet
  transactions: SubWalletTxn[]; // Transaction history for this sub-wallet

  // NCMC Transit specific fields
  max_balance_paise?: number;   // Maximum balance cap (3,000 INR = 300000 paise)

  // FASTag specific fields
  is_security_deposit?: boolean;               // true for FASTag wallets
  vehicle_count?: number;                       // Number of registered vehicles
  security_deposit_per_vehicle_paise?: number;  // 300 INR = 30000 paise per vehicle
  security_deposit_used_paise?: number;         // How much deposit has been consumed
}
```

```typescript
interface SubWalletTxn {
  id: string;
  type: 'CREDIT' | 'DEBIT';
  amount_paise: number;
  description: string;
  date: string;          // ISO 8601
  category?: string;     // MCC category for the transaction
}
```

### 5.5 MCC Category Mapping

19 spending categories with keyword detection from transaction descriptions:

| Category | Icon | Keywords |
|----------|------|----------|
| Taxi/Ride | Taxi icon | Uber, Ola, taxi, ride, cab |
| Food & Dining | Food icon | Swiggy, Zomato, restaurant, cafe, food |
| Groceries | Cart icon | BigBasket, grocery, supermarket, Zepto |
| Shopping | Shopping icon | Amazon, Flipkart, Myntra, mall |
| Fuel | Fuel icon | Petrol, diesel, HP, Indian Oil, BPCL, Shell |
| Travel | Travel icon | Flight, hotel, MakeMyTrip, IRCTC |
| Entertainment | Entertainment icon | Netflix, PVR, Spotify, movie |
| Health | Health icon | Hospital, pharmacy, Apollo, medical |
| Education | Education icon | School, tuition, Byju's, Unacademy |
| Utilities | Utilities icon | Electricity, water, gas, broadband |
| Money Transfer | Transfer icon | P2P, sent to, received from |
| Refunds | Refund icon | Refund, reversal, cashback |
| Recharge | Recharge icon | Mobile recharge, DTH, data pack |
| Insurance | Insurance icon | LIC, HDFC Life, premium |
| Bank Transfer | Bank icon | NEFT, IMPS, bank transfer |
| Wallet Top-up | Wallet icon | Add money, top-up, loaded |
| Subscription | Subscription icon | Monthly, subscription, renewal |
| Government | Government icon | Tax, challan, e-stamp, passport |
| Others | Default icon | Any unmatched transaction |

---

## 6. Sub-Wallet System (Corporate Benefits)

The sub-wallet system provides 5 distinct benefit wallet types, primarily loaded by employers, each with its own compliance rules, spending restrictions, and operational model.

### 6.1 Food Wallet

- **Loading:** Employer-loaded only (no self-load by user)
- **Monthly limit:** 3,000 INR per month
- **Eligible merchants/categories:** Restaurants, Cafes, Food delivery services, Swiggy, Zomato, Office canteen
- **Spending behaviour:** Payments at eligible merchants deduct from Food wallet balance first
- **Tax benefit:** Section 17(2)(viii) of the Income Tax Act -- meal vouchers up to 50 INR per meal are non-taxable
- **UI colour:** Orange (#F97316)
- **Icon:** Food tray icon
- **Typical employer load:** Monthly on payroll date

### 6.2 NCMC Transit

- **Loading:** Self-load from main wallet is allowed, subject to balance cap
- **Balance cap:** 3,000 INR maximum balance at any time (not a monthly limit -- this is the absolute maximum the sub-wallet can hold)
- **Self-load validation:** `(current_ncmc_balance + load_amount) <= 3,000 INR`
- **Eligible categories:** Metro, Bus, Local train, Parking
- **Spending behaviour:** Transit payments use ONLY the NCMC balance. There is NO cascade/fallback to the main wallet. If NCMC balance is insufficient, the transaction fails.
- **No cascade rule:** This is a critical distinction from other sub-wallets. NCMC Transit operates as a completely isolated balance for transit payments.
- **UI colour:** Blue (#00B9F1)
- **Icon:** Metro train icon
- **NCMC compliance:** Follows National Common Mobility Card standards per MoHUA guidelines

### 6.3 FASTag (Security Deposit Model)

FASTag operates on a unique security deposit model rather than a traditional balance model:

- **Security deposit:** 300 INR per registered vehicle
- **Deposit source:** Deducted from main wallet when a new vehicle is registered
- **Toll payment priority:**
  1. First, deduct from **main wallet** balance
  2. If main wallet balance is zero/insufficient, deduct from **security deposit**
- **Top-up refill logic:** When the user adds money to their wallet:
  1. **Security deposit is refilled first** (up to 300 INR per vehicle, based on how much deposit was consumed)
  2. **Remainder** goes to main wallet balance
- **"Add Money" on FASTag page:** This tops up the main wallet (with the deposit refill logic applied)
- **"New Vehicle" action:** Issues a new FASTag + deducts 300 INR security deposit from main wallet
- **Data fields:**
  - `is_security_deposit: true` -- identifies this as a deposit-model sub-wallet
  - `vehicle_count` -- number of registered vehicles
  - `security_deposit_per_vehicle_paise: 30000` -- 300 INR per vehicle
  - `security_deposit_used_paise` -- how much of the total deposit pool has been consumed by toll payments
- **UI colour:** Green (#12B76A)
- **Icon:** Highway/road icon

**FASTag Transaction Example:**

1. User has main wallet: 500 INR, FASTag deposit: 300 INR (1 vehicle, unused)
2. Toll of 200 INR: deducted from main wallet. Main = 300 INR, Deposit = 300 INR.
3. Toll of 400 INR: main wallet has 300 INR, so 300 from main + 100 from deposit. Main = 0, Deposit = 200 INR.
4. User adds 1,000 INR: first 100 INR refills deposit back to 300 INR, remaining 900 INR goes to main wallet. Main = 900 INR, Deposit = 300 INR.

### 6.4 Gift Wallet

- **Loading:** Employer-loaded with an expiry date (typically 1 year from load date). Self-load from main wallet is also allowed (no cap).
- **Expiry:** Gift wallet funds expire on the specified expiry date. After expiry, the balance is forfeited and an EXPIRED badge is displayed.
- **Eligible merchants:** Universal -- usable at all retail merchants (no category restriction)
- **Self-load:** Users can transfer funds from main wallet to Gift wallet at any time with no cap
- **UI display:** Shows expiry date prominently. Displays EXPIRED badge in red when past expiry.
- **UI colour:** Pink (#EC4899)
- **Icon:** Gift box icon

### 6.5 Fuel Wallet

- **Loading:** Employer-loaded only (no self-load by user)
- **Monthly limit:** 2,500 INR per month
- **Eligible merchants:** HP (Hindustan Petroleum), Indian Oil, BPCL (Bharat Petroleum), Shell fuel stations only
- **Spending behaviour:** Payments at eligible fuel stations deduct from Fuel wallet balance first
- **Tax benefit:** Fuel allowance may be exempt under specific employer reimbursement policies
- **UI colour:** Amber (#F59E0B)
- **Icon:** Fuel pump icon

### 6.6 Cascade Spend Logic

When a user makes a merchant payment, the system automatically determines the optimal payment source using this cascade logic:

1. **Category matching:** The system checks the merchant's category (based on MCC code or description keywords) against each sub-wallet's `eligible_categories` list.
2. **Specific wallets first:** If the merchant matches a specific sub-wallet (Food, NCMC Transit, or Fuel), that sub-wallet is used first.
   - **Exception:** NCMC Transit does NOT cascade. If NCMC balance is insufficient for a transit payment, the transaction fails entirely.
3. **Gift wallet fallback:** If no specific sub-wallet matches, or if the specific sub-wallet has insufficient balance, the Gift wallet is checked (it has universal eligibility).
4. **Split payment:** If the matched sub-wallet has insufficient balance but is not NCMC:
   - Deduct available sub-wallet balance first
   - Deduct the remainder from main wallet
   - This creates a split payment across two sources
5. **Main wallet final fallback:** If no sub-wallet matches or all are exhausted, the full amount is deducted from the main wallet.

**API endpoint:** `GET /api/wallet/sub-wallets/eligibility?category={category}` returns the best matching sub-wallet for a given merchant category.

**Mock API function:** `mockFindBestSubWallet(category)` implements this logic client-side.

### 6.7 Total Balance Calculation

**Total Balance = Main Wallet Balance + Sum of All Sub-Wallet Balances**

The RBI 1,00,000 INR balance cap (Load Guard Rule 1) applies to this **combined total**, not just the main wallet. This means:

- If main wallet has 80,000 INR and sub-wallets total 15,000 INR, the total is 95,000 INR.
- The maximum additional load allowed is 5,000 INR (to reach the 1,00,000 INR cap).
- Sub-wallet employer loads also count toward this total.

---

## 7. Wallet App -- Pages and Routes

### 7.1 Route Map

| Route | Page Component | Description |
|-------|---------------|-------------|
| `/login` | LoginPage | Phone + OTP login (mock: any 10-digit number, any 6-digit OTP) |
| `/` | HomePage | WalletStrip (collapsible sub-wallet list), Money Transfer grid, Recharge & Bills, Rewards strip, AI Chat, AI Summary |
| `/wallet` | WalletPage | Wallet overview with balance and quick actions |
| `/wallet/detail` | WalletDetailPage | Detailed balance breakdown, recent transactions, quick action buttons |
| `/wallet/add-money` | AddMoneyPage | Amount input, Payment Gateway selection, PIN verification, Result screen |
| `/wallet/sub/:type` | SubWalletDetailPage | Sub-wallet balance, add money (NCMC/FASTag/Gift only), transaction history, AI prompts |
| `/wallet/statement` | WalletStatementPage | Download statement for 1M/3M/6M/1Y/Custom date range via email |
| `/pay` | MerchantPayPage | QR code placeholder, merchant ID input, sub-wallet suggestion, PIN authentication |
| `/send` | SendMoneyPage | P2P transfer, UUID wallet ID validation, Full KYC required check |
| `/transfer-bank` | TransferToBankPage | Bank withdrawal, IFSC code validation, confirm account number (enter twice) |
| `/bill-pay` | BillPayPage | 6 biller categories, biller selection, amount, PIN authentication |
| `/passbook` | PassbookPage | Transaction history with month grouping, type/category filters, MCC icons |
| `/analytics` | SpendAnalyticsPage | Donut chart, category breakdown, top merchants, daily trend line chart |
| `/budget` | BudgetPage | Category-wise spending limits, monthly total cap, progress bars |
| `/kyc` | KycPage | Current KYC tier status, Aadhaar OTP upgrade flow, document upload |
| `/profile` | ProfilePage | User info display, navigation menu to sub-pages, logout button |
| `/transaction/:id` | TransactionDetailPage | Full transaction details including saga steps and timeline |
| `/notifications` | NotificationsPage | In-app alert list with action buttons (e.g., "Complete KYC", "View Transaction") |
| `/rewards` | RewardsPage | Scratch cards with reveal animation, cashback history list |

### 7.2 Home Page -- WalletStrip Component

The WalletStrip is the primary UI element on the home page, designed to match Paytm's real app experience:

#### Collapsed State (Default)
- Displays total balance (main wallet + all sub-wallets combined)
- Full KYC badge (green) or Min-KYC badge (orange) based on KYC tier
- Chevron icon indicating the strip can be expanded
- Tapping anywhere on the strip expands it

#### Expanded State
- Vertical list showing each wallet with its balance:
  - **Main Wallet** -- primary balance in INR
  - **Food Wallet** -- employer-loaded balance with food icon
  - **NCMC Transit** -- transit balance with metro icon
  - **FASTag Wallet** -- shows "(Deposit)" label, security deposit amount
  - **Gift Wallet** -- universal spend balance with gift icon
  - **Fuel Wallet** -- fuel station balance with fuel icon
- Each row is tappable and navigates to the respective sub-wallet detail page (`/wallet/sub/:type`)
- Inactive/zero-balance sub-wallets are shown greyed out

#### Add Money Section (Below WalletStrip)
- Quick-add buttons: +100 INR, +500 INR, +1,000 INR (marked "Popular"), Custom amount
- Auto top-up toggle: "Auto add 2,000 INR when balance below 200 INR"
- Payment source display: "From: HDFC Bank - 7125" (last 4 digits of linked bank account)
- Tapping any quick-add button navigates to `/wallet/add-money` with the amount pre-filled

#### Home Page Additional Sections
- **Money Transfer Grid:** 4 icons -- Pay Merchant, Send Money, Bank Transfer, Check Balance
- **Recharge & Bills:** 6 icons -- Mobile Recharge, DTH, Electricity, Gas, Water, Broadband
- **Rewards Strip:** Horizontal scroll of scratch cards with "You have X rewards" header
- **AI Chat Button:** Floating action button to open Claude AI chat overlay
- **AI Summary Card:** Auto-generated spending summary for the current month

### 7.3 Transaction Flow (Saga Pattern)

All transactions in the wallet app follow the saga pattern with this consistent UX flow:

#### Step 1: Amount Input
- User enters the transaction amount
- Real-time validation against available balance (for debits)
- Load Guard validation (for Add Money -- checks all 3 RBI rules)
- Amount displayed in Indian format with rupee symbol

#### Step 2: PIN Modal
- 4-digit PIN entry overlay (PinModal component)
- PIN is stored in localStorage (key: `wallet_pin`, default: `1234`)
- 3 attempts allowed before temporary lockout
- Biometric option displayed (mock -- always succeeds)
- On correct PIN, the transaction callback is executed

#### Step 3: API Call (Saga Execution)
- `sagaPost()` function sends the transaction to the backend
- Includes `idempotency_key` (UUID v4) to prevent duplicate processing
- 3-tier fallback: tries real API first, falls back to mock on error
- Loading state shown with spinner and "Processing..." message

#### Step 4: Result Screen
- **Success:** Green checkmark animation, new balance displayed, "Done" button returns to home
- **Failure:** Red error icon, error message from saga response, "Retry" button re-executes, "Cancel" button returns to previous page
- Transaction details: amount, recipient/merchant, reference ID, timestamp

### 7.4 Page Detail Specifications

#### LoginPage (`/login`)
- Phone number input with +91 prefix (India only)
- 10-digit validation
- OTP input (6 digits, mock accepts any value)
- Auto-generates wallet_id and user_id on first login
- Stores auth state in localStorage
- Redirects to `/` on success

#### AddMoneyPage (`/wallet/add-money`)
- Amount input with quick-select chips (100, 500, 1000, 2000, 5000)
- Custom amount input with numeric keyboard
- Load Guard validation runs before proceeding
- Payment gateway selection: UPI, Debit Card, Net Banking, Credit Card (mock -- all succeed)
- PIN verification via PinModal
- Success: balance updated, ledger entry created, sync to admin dashboard

#### MerchantPayPage (`/pay`)
- QR code scanner placeholder (visual only in mock)
- Manual merchant ID input field
- Amount input
- Sub-wallet suggestion: system checks `mockFindBestSubWallet()` based on merchant category
- Shows "Pay from: Food Wallet (balance INR)" or "Pay from: Main Wallet"
- Split payment display when sub-wallet balance is partial
- PIN verification, then saga execution

#### SendMoneyPage (`/send`)
- Recipient wallet ID input (UUID format validation)
- Amount input
- **Full KYC check:** If user is Min-KYC, shows upgrade prompt and blocks the transaction
- **Monthly P2P limit check:** 1,00,000 INR per month for Full-KYC users
- Recent payees list (from payees.store, max 20)
- PIN verification, then saga execution

#### TransferToBankPage (`/transfer-bank`)
- Bank account number input (enter twice for confirmation, must match)
- IFSC code input with format validation (4 letters + 0 + 6 alphanumeric)
- Account holder name input
- Amount input with available balance display
- Recent bank accounts list
- PIN verification, then saga execution

#### BillPayPage (`/bill-pay`)
- 6 biller categories: Electricity, Gas, Water, Broadband, DTH, Mobile Postpaid
- Each category shows a list of billers (e.g., Electricity: Tata Power, Adani, BEST)
- Consumer number / account ID input
- Amount input (some billers show "Fetch Bill" to auto-populate amount)
- PIN verification, then saga execution

#### PassbookPage (`/passbook`)
- Transaction list grouped by month (e.g., "April 2026", "March 2026")
- Each entry shows: MCC icon, description, amount (green for credit, red for debit), date
- Filter bar: All, Credits, Debits, Add Money, Payments, Transfers, Bills
- Search input for transaction descriptions
- Infinite scroll / load more pagination
- Tap on any transaction navigates to `/transaction/:id`

#### SpendAnalyticsPage (`/analytics`)
- **Donut chart:** Category-wise spending breakdown for selected period
- **Category cards:** Each category shows total spend, percentage of total, transaction count
- **Top merchants:** Ranked list of merchants by spend amount
- **Daily trends:** Line chart showing daily spending over the selected period
- Period selector: This Week, This Month, Last 3 Months, Last 6 Months, Custom

#### BudgetPage (`/budget`)
- Monthly total budget cap (user-configurable)
- Category-wise limits (e.g., Food: 5,000 INR, Shopping: 3,000 INR)
- Progress bars showing spend vs. limit for each category
- Colour coding: green (under 75%), orange (75-100%), red (exceeded)
- Notification toggle: alert when approaching limit (80%) or exceeded

#### KycPage (`/kyc`)
- Current tier display: Min-KYC (orange badge) or Full-KYC (green badge)
- For Min-KYC users:
  - "Upgrade to Full KYC" prominent CTA
  - Benefits list: higher limits, P2P enabled, no expiry
  - Aadhaar number input (12 digits with validation)
  - Aadhaar OTP verification flow
  - Status tracking: PENDING, APPROVED, REJECTED
- For Full-KYC users:
  - "Verified" status with green checkmark
  - KYC details: verification date, document type
- For EXPIRED users:
  - Warning banner with grace period countdown
  - "Upgrade Now" CTA

#### SubWalletDetailPage (`/wallet/sub/:type`)
- Sub-wallet balance display with type-specific icon and colour
- **Add Money button** (only for NCMC, FASTag, Gift):
  - NCMC: amount input with 3,000 INR cap validation
  - FASTag: "Add Money" tops up main wallet (with deposit refill), "New Vehicle" issues FASTag
  - Gift: amount input with no cap
  - Food/Fuel: no add money button (employer-loaded only)
- Transaction history list for this sub-wallet
- Eligible categories display
- Monthly limit and usage progress (Food, Fuel)
- AI-powered spending prompts (e.g., "You spent 80% of your food budget")

---

## 8. Admin Dashboard -- Pages and RBAC

### 8.1 Routes

| Route | Page Component | Required Permission |
|-------|---------------|-------------------|
| `/login` | LoginPage | None (public) |
| `/` | DashboardPage | All authenticated roles |
| `/users` | UsersPage | users.view |
| `/users/:id` | UserDetailPage | users.view |
| `/transactions` | TransactionsPage | transactions.view |
| `/transactions/:id` | TransactionDetailPage | transactions.view |
| `/kyc` | KycPage | kyc.view |
| `/benefits` | BenefitsPage | dashboard.view |
| `/settings` | SettingsPage | settings.view |

### 8.2 RBAC (Role-Based Access Control) Roles

| Role | Key Permissions |
|------|----------------|
| SUPER_ADMIN | All permissions (full system access) |
| BUSINESS_ADMIN | Dashboard, users (read-only), transactions (read-only), analytics, KYC (read-only) |
| OPS_MANAGER | Dashboard, users (read/write), transactions (read/write), KYC (approve/reject) |
| CS_AGENT | Dashboard, users (read-only), transactions (read-only) |
| COMPLIANCE_OFFICER | Dashboard, users (read-only), transactions (read/write), KYC (read/write), analytics |
| MARKETING_MANAGER | Dashboard, analytics |

### 8.3 Demo Admin Credentials

Three demo roles (Super Admin, Business Admin, CS Agent) use env-var-driven fallback credentials. See `admin-dashboard/.env.example` for the variable names and [`security.md §3`](security.md#3-authentication--authorization) for the production auth gap analysis.

### 8.4 Dashboard Page (`/`)
- **Overview metrics cards:** Total Users, Active Wallets, Total Transaction Volume, Revenue
- **Transaction volume chart:** Daily/weekly/monthly transaction counts
- **KYC funnel:** Visual showing users at each KYC stage
- **Recent alerts:** Suspicious transactions, failed sagas, compliance flags

### 8.5 Users Page (`/users`)
- **Search:** By name, phone number, wallet ID, user ID
- **Filters:** KYC tier, wallet status, registration date range
- **Table columns:** User ID, Name, Phone, KYC Tier, Balance, Status, Last Active
- **Actions:** View detail, suspend user (with reason), export to CSV
- **PII masking toggle:** Controlled by auth store, masks phone numbers and names for non-privileged roles

### 8.6 User Detail Page (`/users/:id`)
- **Profile section:** Name, phone, email, KYC tier, registration date
- **Wallet section:** Main balance, sub-wallet balances, held amounts
- **Transaction history:** Paginated list with filters
- **KYC section:** Current status, verification documents, approval/rejection history
- **Actions:** Suspend/unsuspend, trigger KYC review, view compliance flags

### 8.7 Transactions Page (`/transactions`)
- **Search:** By transaction ID, saga ID, user ID, reference ID
- **Filters:** Type (Add Money, Merchant Pay, P2P, etc.), status, date range, amount range
- **Table columns:** Transaction ID, User, Type, Amount, Status, Date
- **Flagged transactions highlight:** Red background for suspicious flags

### 8.8 KYC Page (`/kyc`)
- **Queue view:** List of FULL_KYC_PENDING applications awaiting review
- **Stats cards:** Pending count, Approved today, Rejected today, Avg processing time
- **Review interface:** Document viewer, user details, approve/reject buttons with reason
- **Expiry alerts:** List of Min-KYC wallets approaching 12-month expiry

### 8.9 Benefits Page (`/benefits`) -- Sub-Wallet Admin

Two-column layout for corporate benefits management:

#### Left Column: BulkLoadPanel
- Select wallet type dropdown (Food, NCMC, FASTag, Gift, Fuel)
- Occasion/reason input (e.g., "Monthly food allowance", "Diwali gift")
- Amount per user input (in INR)
- User selection: All eligible users, specific department, custom list
- Preview panel showing: number of users, total amount, per-user amount
- Execute button with confirmation dialog
- Load history table

#### Right Column: AiBenefitsInsight
- Claude AI-powered insights panel
- Utilisation analysis: "Food wallet utilisation is 62% -- 38% of loaded funds are unused"
- Expiry risk: "1,247 Gift wallets worth 12.4L INR expire in the next 30 days"
- Recommendations: "Consider increasing Food wallet limit from 3,000 to 4,000 INR based on spending patterns"
- Refreshes on page load and on-demand

#### Below: UtilisationDashboard
- **4 metric cards:**
  1. Total Loaded: aggregate amount loaded across all sub-wallets
  2. Total Spent: aggregate amount spent from sub-wallets
  3. Utilisation Rate: (Total Spent / Total Loaded) as percentage
  4. Expiring This Month: count and value of sub-wallets expiring
- **Breakdown table by wallet type:** Type, Total Loaded, Total Spent, Utilisation %, Active Users, Avg Balance

---

## 9. API Endpoints

### 9.1 Express API Server (Production)

| Method | Path | Description |
|--------|------|-------------|
| GET | `/health` | Health check -- returns server status and version |
| POST | `/api/chat` | Claude AI chat endpoint. Body: `{ message, role: 'admin' or 'user', context }` |
| POST | `/api/summarise-transactions` | AI-generated transaction summary. Body: `{ transactions, period }` |
| GET | `/api/kyc-alerts/preview` | Preview at-risk KYC users (approaching expiry, unusual patterns) |
| POST | `/api/kyc-alerts/run` | Execute KYC alerts using Claude Haiku for fast processing |
| POST | `/api/wallet/validate-load` | Load Guard validation (3 RBI rules). Body: `{ user_id, amount_paise, kyc_tier }` |
| GET | `/api/wallet/load-guard-log` | Retrieve blocked load attempts for audit |
| GET | `/api/wallet/sub-wallets/:userId` | Get all sub-wallets for a user |
| POST | `/api/wallet/sub-wallets/load` | Employer loads a sub-wallet. Body: `{ user_id, type, amount_paise, loaded_by }` |
| POST | `/api/wallet/sub-wallets/spend` | Spend from a sub-wallet. Body: `{ user_id, type, amount_paise, merchant }` |
| GET | `/api/wallet/sub-wallets/eligibility` | Check merchant category to sub-wallet match. Query: `?category=food` |
| GET | `/api/wallet/benefits/utilisation` | Aggregate utilisation statistics for admin dashboard |

### 9.2 Wallet App Mock API (Client-Side Fallback)

The mock API in `src/api/mock.ts` provides complete functionality with localStorage persistence:

| Function | Description |
|----------|-------------|
| `mockApi.getBalance()` | Returns wallet balance from localStorage |
| `mockApi.getLedger()` | Returns paginated transaction history |
| `mockApi.sagaSuccess()` | Simulates successful saga with ledger entry + balance update |
| `mockGetSubWallets()` | Returns all sub-wallets with computed totals |
| `mockGetSubWalletDetail(type)` | Returns single sub-wallet by type |
| `mockAddMoneyToSubWallet(type, paise)` | Self-load with type-specific validation (NCMC: balance cap, FASTag: main wallet + deposit refill, Gift: direct add) |
| `mockFastagTransaction(paise, tollName)` | FASTag toll payment (main wallet first, then security deposit fallback) |
| `mockIssueFastag(vehicleNumber)` | Register new vehicle + deduct 300 INR deposit from main wallet |
| `mockFindBestSubWallet(category)` | Returns best matching sub-wallet for a merchant category |
| `mockCheckEligibility(category, type)` | Checks if a category is eligible for a specific sub-wallet type |
| `validateLoadAmount(paise)` | Load Guard enforcement (all 3 RBI rules) |

**Default seed data:**
- Main wallet balance: 236.11 INR (23611 paise)
- 12 historical transactions across various types
- 5 sub-wallets with realistic balances and transaction histories

---

## 10. MCP Tools (39 Tools)

### 10.1 User Tools (12)

| Tool | Description |
|------|-------------|
| `get_wallet_balance` | Retrieve current wallet balance for a user |
| `get_transaction_history` | Get paginated transaction history with filters |
| `flag_suspicious_transaction` | Flag a transaction for compliance review |
| `unflag_transaction` | Remove suspicious flag from a transaction |
| `get_spending_summary` | Aggregated spending summary by category for a period |
| `search_transactions` | Search transactions by keyword, amount range, date range |
| `get_user_profile` | Get user profile details including KYC status |
| `compare_spending` | Compare spending between two periods |
| `detect_recurring_payments` | Identify recurring payment patterns |
| `generate_report` | Generate a detailed report (spending, compliance, etc.) |
| `get_notifications` | Get user notifications and alerts |
| `set_alert_threshold` | Set spending alert thresholds for a user |

### 10.2 Transaction Tools (5)

| Tool | Description |
|------|-------------|
| `add_money` | Load money into wallet (with Load Guard validation) |
| `pay_merchant` | Make a payment to a merchant |
| `transfer_p2p` | Transfer money to another wallet user (Full KYC required) |
| `pay_bill` | Pay a utility bill or recharge |
| `request_refund` | Request refund for a transaction |

### 10.3 Admin Tools (10)

| Tool | Description |
|------|-------------|
| `get_system_stats` | System-wide statistics (users, transactions, volume) |
| `search_users` | Search users by name, phone, ID, KYC status |
| `get_flagged_transactions` | Get all transactions flagged as suspicious |
| `suspend_user` | Suspend a user's wallet (with reason) |
| `get_failed_transactions` | Get transactions in DLQ or failed state |
| `get_kyc_stats` | KYC pipeline statistics (pending, approved, rejected counts) |
| `check_compliance` | Run compliance checks on a user or transaction |
| `compare_users` | Compare metrics between two users |
| `get_peak_usage` | Identify peak usage hours and days |
| `get_monthly_trends` | Monthly trend data for transactions and volume |

### 10.4 KYC Tools (5)

| Tool | Description |
|------|-------------|
| `approve_kyc` | Approve a pending KYC application |
| `reject_kyc` | Reject a pending KYC application (with reason) |
| `request_kyc_upgrade` | Initiate KYC upgrade request for a user |
| `query_kyc_expiry` | Check KYC expiry status and approaching deadlines |
| `generate_kyc_renewal_report` | Generate report of KYC renewals needed |

### 10.5 Support Tools (3)

| Tool | Description |
|------|-------------|
| `raise_dispute` | Raise a dispute for a transaction |
| `get_dispute_status` | Check current status of a dispute |
| `get_refund_status` | Check refund processing status |

### 10.6 Sub-Wallet Tools (4)

| Tool | Description |
|------|-------------|
| `get_sub_wallets` | Get all sub-wallets for a user |
| `load_sub_wallet` | Load money into a specific sub-wallet |
| `get_sub_wallet_transactions` | Get transaction history for a sub-wallet |
| `validate_merchant_eligibility` | Check if a merchant category is eligible for a sub-wallet |

---

## 11. State Management (Zustand Stores)

### 11.1 Wallet App Stores (7)

| Store | Key State | Description |
|-------|-----------|-------------|
| `auth.store` | walletId, userId, userName, phone | Login/logout, sync to admin dashboard |
| `wallet.store` | balance_paise, held_paise, available_paise, kyc_tier | Core wallet state, refreshed after every transaction |
| `pin.store` | pin (4-digit string) | Stored in localStorage, default: "1234" |
| `budget.store` | categoryLimits, monthlyCap, spending | Category-wise budget tracking |
| `payees.store` | recentPayees (by type: P2P, merchant, bank) | Max 20 per type, most recent first |
| `notifications.store` | alerts (title, message, type, actionPath, isRead) | In-app notification management |
| `rewards.store` | scratchCards (amount, isScratched), totalCashback | Scratch card reveal state and cashback total |

### 11.2 Admin Dashboard Stores (6)

| Store | Key State | Description |
|-------|-----------|-------------|
| `auth.store` | adminUser, role, permissions, piiMaskingEnabled | Admin auth with PII masking toggle |
| `users.store` | userList, filters, pagination, searchQuery | User management state |
| `transactions.store` | transactionList, filters, detailView | Transaction monitoring state |
| `kyc.store` | kycStats, queueItems | KYC approval queue |
| `dashboard.store` | overviewMetrics | Dashboard summary data |
| `ui.store` | sidebarCollapsed, theme | UI preferences |

---

## 12. Key Implementation Patterns

### 12.1 Three-Tier API Fallback

```
Tier 1: Vite dev middleware (POST /api/sync) --> shared JSON file
Tier 2: Express API on Render --> Claude AI + MCP data
Tier 3: Client-side mock (localStorage) --> always available
```

The `sagaPost` function in `saga.api.ts` tries the real API first, then falls back to mock on any error. This ensures the app always works, even offline.

### 12.2 Idempotency

Every transaction POST includes a UUID `idempotency_key` generated client-side. The backend deduplicates on this key, ensuring that network retries or double-taps never result in duplicate transactions.

### 12.3 Data Sync

When transactions occur in the wallet app, `sync.ts` pushes events to:
1. **Admin Dashboard (dev mode):** Vite plugin writes to `.shared-data/wallet-events.json`
2. **Render Backend (prod):** `POST /api/wallet/transact` updates MCP mock data

### 12.4 localStorage Keys

| Key | Content |
|-----|---------|
| `__mock_balance` | Main wallet balance in paise |
| `__mock_ledger` | Transaction history array (JSON) |
| `__mock_sub_wallets_v2` | Sub-wallet data (v2 includes NCMC cap + FASTag deposit model) |
| `wallet_pin` | 4-digit PIN string |
| `wallet_auth` | Login state (walletId, userId, phone) |
| `wallet_budget` | Budget limits and spending data |
| `wallet_payees` | Recent contacts by type |
| `wallet_notifications` | Alert messages array |
| `wallet_rewards` | Scratch cards + cashback total |

### 12.5 Currency Formatting

```typescript
function formatPaise(paise: string | number): string {
  const num = Number(paise) / 100;
  return '\u20B9' + num.toLocaleString('en-IN', {
    minimumFractionDigits: 2,
    maximumFractionDigits: 2
  });
}
```

All amounts are stored as integers in paise. Display formatting always uses `toLocaleString('en-IN')` for Indian number formatting (e.g., 1,00,000.00).

---

## 13. Deployment

| Target | Platform | URL |
|--------|----------|-----|
| Wallet App | GitHub Pages | gaurav2sheth.github.io/ppi-wallet-app |
| Admin Dashboard | GitHub Pages | gaurav2sheth.github.io/ppi-wallet-admin |
| Backend API | Render | ppi-wallet-api.onrender.com |
| MCP Server | Local (Claude Desktop) | stdio transport |

### Build and Deploy Wallet App

```bash
VITE_API_URL=https://ppi-wallet-api.onrender.com npm run build
npx gh-pages -d dist
```

---

## 14. Competitive Context

Benchmarked against PhonePe Wallet (~600M users), Amazon Pay (~90M), Airtel Payments Bank (~80M).

Key differentiators for this PPI wallet:

1. **Deep Conversational AI** -- 49 MCP tools for natural language wallet operations (no competitor offers this)
2. **Corporate Benefits Sub-Wallets** -- 5 wallet types with distinct compliance models (NCMC balance cap, FASTag security deposit)
3. **Proactive Financial Wellness** -- Budget manager, spend analytics, AI-powered insights
4. **Wallet Load Guard** -- Real-time RBI compliance validation with AI-powered contextual suggestions

---

## 15. Acceptance Criteria

### 15.1 RBI Compliance Acceptance Criteria

- [ ] Min-KYC wallet balance never exceeds 10,000 INR
- [ ] Full-KYC wallet balance never exceeds 2,00,000 INR
- [ ] Combined balance (main + sub-wallets) never exceeds 1,00,000 INR (Load Guard Rule 1)
- [ ] Monthly load total never exceeds 2,00,000 INR (Load Guard Rule 2)
- [ ] Min-KYC wallets cannot perform P2P transfers
- [ ] Full-KYC P2P transfers capped at 1,00,000 INR per month
- [ ] KYC state transitions follow the defined state machine exactly
- [ ] Min-KYC wallets expire after 12 months with 60-day grace period
- [ ] Dormant wallets flagged after 12 months of inactivity
- [ ] Dormant wallets closed after 24 months with balance to suspense
- [ ] Dispute auto-closes in customer favour after 30 days
- [ ] All blocked load attempts are logged for audit

### 15.2 Sub-Wallet Acceptance Criteria

- [ ] Food wallet only accepts employer loads (no self-load)
- [ ] Food wallet monthly limit is 3,000 INR
- [ ] NCMC Transit balance never exceeds 3,000 INR
- [ ] NCMC Transit does not cascade to main wallet (transaction fails if insufficient)
- [ ] FASTag security deposit is 300 INR per vehicle
- [ ] FASTag toll deducts from main wallet first, then security deposit
- [ ] FASTag top-up refills security deposit first, then main wallet
- [ ] Gift wallet shows expiry date and EXPIRED badge when past
- [ ] Gift wallet allows self-load with no cap
- [ ] Fuel wallet only accepts employer loads
- [ ] Fuel wallet monthly limit is 2,500 INR
- [ ] Fuel wallet only works at HP, Indian Oil, BPCL, Shell
- [ ] Cascade spend logic correctly matches merchant category to sub-wallet
- [ ] Split payment works when sub-wallet has partial balance

### 15.3 Transaction Flow Acceptance Criteria

- [ ] All transactions follow Amount -> PIN -> Execute -> Result flow
- [ ] PIN modal requires correct 4-digit PIN before executing
- [ ] Idempotency keys prevent duplicate transactions
- [ ] Failed sagas show error with retry option
- [ ] Successful transactions update balance and ledger immediately
- [ ] Transaction sync pushes events to admin dashboard

### 15.4 Wallet App UI Acceptance Criteria

- [ ] WalletStrip collapses and expands correctly
- [ ] Total balance includes all sub-wallet balances
- [ ] Quick-add buttons pre-fill amount on Add Money page
- [ ] Passbook groups transactions by month
- [ ] Analytics donut chart reflects actual spending categories
- [ ] Budget progress bars update in real-time
- [ ] KYC page shows correct tier and upgrade flow
- [ ] All amounts formatted in Indian locale with rupee symbol

### 15.5 Admin Dashboard Acceptance Criteria

- [ ] RBAC restricts page access based on role permissions
- [ ] PII masking toggle hides sensitive user data
- [ ] KYC queue shows pending applications with approve/reject actions
- [ ] Benefits page allows bulk sub-wallet loading
- [ ] Utilisation dashboard shows accurate metrics per wallet type
- [ ] AI insights panel generates relevant recommendations
