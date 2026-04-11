# PPI Wallet Platform — Recreation Prompt

Use this document as a comprehensive prompt to recreate the Paytm PPI (Prepaid Payment Instrument) Wallet App and Admin Dashboard from scratch. It consolidates architecture, product requirements, compliance rules, data models, API contracts, and implementation details from all project documents.

---

## 1. Product Overview

Build an RBI-regulated PPI Wallet system with:
- **Consumer Wallet App** — Mobile-first React SPA for end users (add money, pay merchants, P2P transfer, bank transfer, bill pay, passbook, analytics, rewards, KYC, sub-wallets)
- **Admin Dashboard** — Desktop-first React SPA for operations teams (user management, transaction monitoring, KYC queue, spend analytics, benefits management, AI insights)
- **Backend API** — Express.js REST API with Claude AI integration (chat, summarisation, KYC alerts, load guard)
- **MCP Server** — 39 Claude AI tools for natural language wallet operations

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
| HTTP | Axios with 3-tier fallback (Vite middleware → Express API → client-side mock) |
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

## 3. Design System — Paytm PODS

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
- Amount display: ₹ prefix, Indian locale formatting (1,00,000.00)

---

## 4. RBI Compliance Rules (Non-Negotiable)

### KYC Tiers & Limits

| Rule | Min-KYC | Full-KYC |
|------|---------|----------|
| Balance cap | ₹10,000 | ₹2,00,000 |
| Monthly load cap | ₹10,000 | ₹1,00,000 |
| Annual load cap | ₹1,20,000 | — |
| P2P transfer | PROHIBITED | ₹1,00,000/month |
| Wallet validity | 12 months (then EXPIRED) | Unlimited |
| Cash withdrawal | Not allowed | ₹2,000/txn, ₹10,000/month |

### KYC State Machine
```
UNVERIFIED → MIN_KYC (mobile OTP)
MIN_KYC → FULL_KYC_PENDING (Aadhaar OTP submitted)
FULL_KYC_PENDING → FULL_KYC (approved)
FULL_KYC_PENDING → REJECTED (denied)
Any → SUSPENDED (fraud/compliance)
MIN_KYC → EXPIRED (12-month validity lapsed, 60-day grace)
```

### Wallet Load Guard (3 Rules)
These are enforced on every Add Money operation:

1. **BALANCE_CAP** — Total wallet balance (main + all sub-wallets) must not exceed ₹1,00,000
2. **MONTHLY_LOAD** — Calendar month total loads must not exceed ₹2,00,000
3. **MIN_KYC_CAP** — Min-KYC wallets: total balance must not exceed ₹10,000

### Wallet Lifecycle States
```
ACTIVE → SUSPENDED (fraud/compliance hold)
ACTIVE → DORMANT (12 months no transactions)
DORMANT → CLOSED (24 months, balance to suspense)
MIN_KYC → EXPIRED (12-month validity, 60-day grace)
Any → CLOSED (user-initiated, T+1 balance refund via NEFT/IMPS)
```

### Other Compliance
- CTR (Cash Transaction Report): Mandatory for cash > ₹10 lakh/day
- STR (Suspicious Transaction Report): Filed within 7 days
- EDD (Enhanced Due Diligence): Triggered at ₹50,000/month load
- Data retention: 5 years ledger/admin, 10 years KYC artefacts
- Dispute SLA: 30 days per RBI, auto-close in customer's favour at day 30+1
- All data localised to India

---

## 5. Data Models

### Wallet Balance Response
```typescript
interface WalletBalanceResponse {
  success: boolean;
  wallet_id: string;
  user_id: string;
  balance_paise: string;     // BigInt as string
  held_paise: string;
  available_paise: string;   // balance - held
  kyc_tier: 'MINIMUM' | 'FULL';
  is_active: boolean;
  updated_at: string;        // ISO 8601
}
```

### Ledger Entry (Append-Only)
```typescript
interface LedgerEntry {
  id: string;                // UUID
  entry_type: 'CREDIT' | 'DEBIT' | 'HOLD' | 'HOLD_RELEASE';
  amount_paise: string;
  balance_after_paise: string;
  held_paise_after: string;
  transaction_type: 'ADD_MONEY' | 'MERCHANT_PAY' | 'P2P_TRANSFER' | 'WALLET_TO_BANK' | 'BILL_PAY' | 'REFUND';
  reference_id: string | null;
  description: string;
  idempotency_key: string;
  hold_id: string | null;
  created_at: string;
  payment_source?: string;   // e.g. "UPI - HDFC Bank 7125"
}
```

### Saga Response (Transaction Result)
```typescript
interface SagaResponse {
  saga_id: string;
  saga_type: string;
  status: 'STARTED' | 'RUNNING' | 'COMPLETED' | 'COMPENSATING' | 'COMPENSATED' | 'DLQ';
  result?: {
    balance_after_paise?: string;
    entry_id?: string;
  };
  error?: string;
  steps: SagaStep[];
  created_at: string;
  updated_at: string;
}
```

### Sub-Wallet
```typescript
interface SubWallet {
  sub_wallet_id: string;
  type: 'FOOD' | 'NCMC TRANSIT' | 'FASTAG' | 'GIFT' | 'FUEL';
  icon: string;              // Emoji
  color: string;             // Hex
  label: string;
  balance_paise: number;
  status: 'ACTIVE' | 'FROZEN' | 'EXPIRED';
  monthly_limit_paise: number;
  monthly_loaded_paise: number;
  loaded_by: string;
  last_loaded_at: string;
  expiry_date: string | null;
  eligible_categories: string[];
  transactions: SubWalletTxn[];
  // NCMC-specific
  max_balance_paise?: number;           // ₹3,000 cap
  // FASTag-specific
  is_security_deposit?: boolean;
  vehicle_count?: number;
  security_deposit_per_vehicle_paise?: number;  // ₹300
  security_deposit_used_paise?: number;
}
```

### MCC Category Mapping
19 spending categories with keyword detection from transaction descriptions:
```
Taxi/Ride (🚕), Food & Dining (🍔), Groceries (🛒), Shopping (🛍️),
Fuel (⛽), Travel (✈️), Entertainment (🎬), Health (🏥),
Education (📚), Utilities (💡), Money Transfer (💸), Refunds (↩️),
Recharge (📱), Insurance (🛡️), Bank Transfer (🏦), Wallet Top-up (💳),
Subscription (🔄), Government (🏛️), Others (💰)
```

---

## 6. Sub-Wallet System (Corporate Benefits)

5 benefit wallet types loaded by employers, each with distinct rules:

### Food Wallet 🍱
- Employer-loaded only (no self-load)
- Monthly limit: ₹3,000
- Eligible: Restaurants, Cafes, Food delivery, Swiggy, Zomato, Canteen
- Spending deducts from Food wallet balance

### NCMC Transit 🚇
- Own **₹3,000 balance limit** (not monthly — max balance at any time)
- Self-load from main wallet allowed (up to ₹3,000 cap)
- Transit payments (Metro, Bus, Local train, Parking) use **only NCMC balance** — not main wallet
- No cascade to main wallet

### FASTag 🛣️ (Security Deposit Model)
- **₹300 security deposit per vehicle** — sub-wallet holds the deposit only
- FASTag toll transactions **deduct from main wallet** first
- If main wallet is zero, **security deposit** is consumed
- On next top-up: **security deposit refilled first** (up to ₹300 per vehicle), then remainder goes to main wallet
- "Add Money" on FASTag page = top-up to main wallet (with deposit refill logic)
- "New Vehicle" = issue FASTag + ₹300 deposit deducted from main wallet
- `is_security_deposit: true`, `vehicle_count`, `security_deposit_per_vehicle_paise: 30000`, `security_deposit_used_paise`

### Gift Wallet 🎁
- Employer-loaded with expiry date (typically 1 year)
- Self-load from main wallet allowed (no cap)
- Usable at all retail merchants (universal eligibility)
- Shows expiry date, EXPIRED badge when past

### Fuel Wallet ⛽
- Employer-loaded only
- Monthly limit: ₹2,500
- Eligible: HP, Indian Oil, BPCL, Shell stations only

### Cascade Spend Logic
For merchant payments, the system auto-detects the best sub-wallet match based on MCC category:
1. Check specific wallets first (Food, Transit, Fuel) based on merchant category keyword matching
2. Gift wallet as fallback (universal eligibility)
3. If sub-wallet balance insufficient: sub-wallet amount + remainder from main wallet (split payment)

### Total Balance = Main Wallet + All Sub-Wallet Balances
The ₹1,00,000 RBI balance cap applies to the **combined total** of main wallet + all sub-wallets.

---

## 7. Wallet App — Pages & Routes

| Route | Page | Description |
|-------|------|-------------|
| `/login` | LoginPage | Phone + OTP login (mock: any 10-digit number) |
| `/` | HomePage | WalletStrip (collapsible sub-wallet list), Money Transfer grid, Recharge & Bills, Rewards strip, AI Chat, AI Summary |
| `/wallet` | WalletPage | Wallet overview |
| `/wallet/detail` | WalletDetailPage | Detailed balance, recent transactions, quick actions |
| `/wallet/add-money` | AddMoneyPage | Amount input → Payment Gateway → PIN → Result |
| `/wallet/sub/:type` | SubWalletDetailPage | Sub-wallet balance, add money (NCMC/FASTag/Gift), transaction history, AI prompts |
| `/wallet/statement` | WalletStatementPage | Download statement (1M/3M/6M/1Y/Custom) via email |
| `/pay` | MerchantPayPage | QR placeholder, merchant ID, sub-wallet suggestion, PIN auth |
| `/send` | SendMoneyPage | P2P transfer, UUID wallet ID validation, Full KYC required |
| `/transfer-bank` | TransferToBankPage | Bank withdrawal, IFSC validation, confirm account number |
| `/bill-pay` | BillPayPage | 6 categories, biller selection, PIN auth |
| `/passbook` | PassbookPage | Transaction history with month grouping, type filters, MCC icons |
| `/analytics` | SpendAnalyticsPage | Donut chart, category breakdown, top merchants, daily trends |
| `/budget` | BudgetPage | Category-wise spending limits, monthly cap |
| `/kyc` | KycPage | Tier status, Aadhaar OTP upgrade flow |
| `/profile` | ProfilePage | User info, navigation menu, logout |
| `/transaction/:id` | TransactionDetailPage | Full transaction details with saga steps |
| `/notifications` | NotificationsPage | In-app alerts with action buttons |
| `/rewards` | RewardsPage | Scratch cards, cashback history |

### Home Page — WalletStrip Component
The WalletStrip is the primary UI element, designed to match Paytm's real app:
- **Collapsed**: Shows total balance (main + sub-wallets), Full KYC badge, chevron to expand
- **Expanded**: Vertical list — Main Wallet ₹X, Food Wallet ₹X, NCMC Transit ₹X, FASTag Wallet ₹X (Deposit), Gift Wallet ₹X, Fuel Wallet ₹X — each tappable to navigate to detail page
- Below: Add Money section with quick-add buttons (+₹100, +₹500, +₹1,000 Popular, Custom)
- Auto top-up toggle: "Auto add ₹2,000 when balance below ₹200"
- Payment source: "From: HDFC Bank - 7125"

### Transaction Flow (Saga Pattern)
All transactions follow: Amount Input → PIN Modal → API Call (saga) → Result Screen (success/failure with retry)

PIN is a 4-digit code stored in localStorage (default: 1234). PinModal component handles verification before executing the transaction callback.

---

## 8. Admin Dashboard — Pages & RBAC

### Routes
| Route | Page | Permission |
|-------|------|-----------|
| `/login` | LoginPage | — |
| `/` | DashboardPage | All roles |
| `/users` | UsersPage | users.view |
| `/users/:id` | UserDetailPage | users.view |
| `/transactions` | TransactionsPage | transactions.view |
| `/transactions/:id` | TransactionDetailPage | transactions.view |
| `/kyc` | KycPage | kyc.view |
| `/benefits` | BenefitsPage | dashboard.view |
| `/settings` | SettingsPage | settings.view |

### RBAC Roles
| Role | Key Permissions |
|------|----------------|
| SUPER_ADMIN | All permissions |
| BUSINESS_ADMIN | Dashboard, users (read), transactions (read), analytics, KYC (read) |
| OPS_MANAGER | Dashboard, users, transactions, KYC (approve/reject) |
| CS_AGENT | Dashboard, users (read), transactions (read) |
| COMPLIANCE_OFFICER | Dashboard, users (read), transactions, KYC, analytics |
| MARKETING_MANAGER | Dashboard, analytics |

### Mock Admin Credentials
- Super Admin: admin / admin123
- Business Admin: business / admin123
- CS Agent: support / admin123

### Benefits Page (Sub-Wallet Admin)
Two-column layout:
- **Left**: BulkLoadPanel — select wallet type, occasion, amount per user, preview, execute bulk load
- **Right**: AiBenefitsInsight — Claude-powered insights on utilisation, expiry risk, recommendations
- **Below**: UtilisationDashboard — 4 metric cards (Total Loaded, Total Spent, Utilisation Rate, Expiring This Month) + breakdown table by wallet type

---

## 9. API Endpoints

### Express API Server (Production)

| Method | Path | Description |
|--------|------|-------------|
| GET | /health | Health check |
| POST | /api/chat | Claude AI chat (role: admin/user) |
| POST | /api/summarise-transactions | AI transaction summary |
| GET | /api/kyc-alerts/preview | Preview at-risk KYC users |
| POST | /api/kyc-alerts/run | Execute KYC alerts (Claude Haiku) |
| POST | /api/wallet/validate-load | Load Guard validation (3 RBI rules) |
| GET | /api/wallet/load-guard-log | Blocked load attempts |
| GET | /api/wallet/sub-wallets/:userId | Get all sub-wallets |
| POST | /api/wallet/sub-wallets/load | Employer loads sub-wallet |
| POST | /api/wallet/sub-wallets/spend | Spend from sub-wallet |
| GET | /api/wallet/sub-wallets/eligibility | Check merchant → sub-wallet match |
| GET | /api/wallet/benefits/utilisation | Aggregate utilisation stats |

### Wallet App Mock API (Client-Side Fallback)
The mock API in `src/api/mock.ts` provides complete functionality with localStorage persistence:

- `mockApi.getBalance()` — Wallet balance
- `mockApi.getLedger()` — Paginated transaction history
- `mockApi.sagaSuccess()` — Simulates successful saga with ledger entry + balance update
- `mockGetSubWallets()` — All sub-wallets with totals
- `mockGetSubWalletDetail(type)` — Single sub-wallet
- `mockAddMoneyToSubWallet(type, paise)` — Self-load (NCMC: balance cap, FASTag: main wallet + deposit refill, Gift: direct add)
- `mockFastagTransaction(paise, tollName)` — FASTag toll (main wallet → security deposit fallback)
- `mockIssueFastag(vehicleNumber)` — New vehicle + ₹300 deposit
- `mockFindBestSubWallet(category)` — Best sub-wallet for merchant category
- `mockCheckEligibility(category, type)` — Category eligibility check
- `validateLoadAmount(paise)` — Load Guard (3 RBI rules)

Default seed data: ₹236.11 main wallet balance, 12 historical transactions, 5 sub-wallets with realistic transactions.

---

## 10. MCP Tools (39 Tools)

### User Tools (12)
`get_wallet_balance`, `get_transaction_history`, `flag_suspicious_transaction`, `unflag_transaction`, `get_spending_summary`, `search_transactions`, `get_user_profile`, `compare_spending`, `detect_recurring_payments`, `generate_report`, `get_notifications`, `set_alert_threshold`

### Transaction Tools (5)
`add_money`, `pay_merchant`, `transfer_p2p`, `pay_bill`, `request_refund`

### Admin Tools (10)
`get_system_stats`, `search_users`, `get_flagged_transactions`, `suspend_user`, `get_failed_transactions`, `get_kyc_stats`, `check_compliance`, `compare_users`, `get_peak_usage`, `get_monthly_trends`

### KYC Tools (5)
`approve_kyc`, `reject_kyc`, `request_kyc_upgrade`, `query_kyc_expiry`, `generate_kyc_renewal_report`

### Support Tools (3)
`raise_dispute`, `get_dispute_status`, `get_refund_status`

### Sub-Wallet Tools (4)
`get_sub_wallets`, `load_sub_wallet`, `get_sub_wallet_transactions`, `validate_merchant_eligibility`

---

## 11. State Management (Zustand Stores)

### Wallet App Stores (7)
1. **auth.store** — walletId, userId, userName, phone, login/logout, sync to admin dashboard
2. **wallet.store** — balance_paise, held_paise, available_paise, kyc_tier
3. **pin.store** — 4-digit PIN in localStorage (default: 1234)
4. **budget.store** — category limits, monthly cap, spending tracking
5. **payees.store** — recent payees by type (P2P, merchant, bank), max 20 each
6. **notifications.store** — alerts with title, message, type, action path, read status
7. **rewards.store** — scratch cards (amount, isScratched), total cashback

### Admin Dashboard Stores (6)
1. **auth.store** — admin user, role, permissions, PII masking toggle
2. **users.store** — user list, filters, pagination, search
3. **transactions.store** — transaction list, filters, detail view
4. **kyc.store** — KYC stats, queue items
5. **dashboard.store** — overview metrics
6. **ui.store** — sidebar collapsed, theme

---

## 12. Key Implementation Patterns

### 3-Tier API Fallback
```
1. Vite dev middleware (POST /api/sync) → shared JSON file
2. Express API (Render) → Claude AI + MCP data
3. Client-side mock (localStorage) → always available
```
The `sagaPost` function in `saga.api.ts` tries the real API first, then falls back to mock on any error. This ensures the app always works, even offline.

### Idempotency
Every transaction POST includes a UUID `idempotency_key` generated client-side. The backend deduplicates on this key.

### Data Sync
When transactions occur in the wallet app, `sync.ts` pushes events to:
1. Admin Dashboard (dev mode): Vite plugin → `.shared-data/wallet-events.json`
2. Render Backend (prod): `POST /api/wallet/transact` → updates MCP mock data

### localStorage Keys
| Key | Content |
|-----|---------|
| `__mock_balance` | Main wallet balance in paise |
| `__mock_ledger` | Transaction history array |
| `__mock_sub_wallets_v2` | Sub-wallet data (v2 = NCMC cap + FASTag deposit model) |
| `wallet_pin` | 4-digit PIN |
| `wallet_auth` | Login state |
| `wallet_budget` | Budget limits |
| `wallet_payees` | Recent contacts |
| `wallet_notifications` | Alert messages |
| `wallet_rewards` | Scratch cards + cashback |

### Currency Formatting
```typescript
function formatPaise(paise: string | number): string {
  const num = Number(paise) / 100;
  return '₹' + num.toLocaleString('en-IN', { minimumFractionDigits: 2, maximumFractionDigits: 2 });
}
```

---

## 13. Deployment

| Target | Platform | URL |
|--------|----------|-----|
| Wallet App | GitHub Pages | gaurav2sheth.github.io/ppi-wallet-app |
| Admin Dashboard | GitHub Pages | gaurav2sheth.github.io/ppi-wallet-admin |
| Backend API | Render | ppi-wallet-api.onrender.com |
| MCP Server | Local (Claude Desktop) | stdio transport |

Build & deploy wallet app:
```bash
VITE_API_URL=https://ppi-wallet-api.onrender.com npm run build
npx gh-pages -d dist
```

---

## 14. Competitive Context

Benchmarked against PhonePe Wallet (~600M users), Amazon Pay (~90M), Airtel Payments Bank (~80M).

Key differentiators for this PPI wallet:
1. **Deep Conversational AI** — 39 MCP tools for natural language wallet operations (no competitor does this)
2. **Corporate Benefits Sub-Wallets** — 5 wallet types with distinct compliance models (NCMC balance cap, FASTag security deposit)
3. **Proactive Financial Wellness** — Budget manager, spend analytics, AI insights
4. **Wallet Load Guard** — Real-time RBI compliance validation with AI-powered contextual suggestions

---

## 15. Reproduction Steps

To recreate this system from scratch:

1. **Set up 4 repos**: wallet-app (React+Vite), admin-dashboard (React+Vite+AntDesign), mcp (Node.js MCP server), api-server (Express.js)

2. **Build the mock data layer first** (`mock.ts`): localStorage-persisted wallet balance, ledger, sub-wallets with seed data. This lets the entire UI work without any backend.

3. **Build core wallet pages**: Login → Home (WalletStrip) → Add Money (with Payment Gateway) → Passbook. These form the critical path.

4. **Add transaction flows**: Merchant Pay, Send Money (P2P), Bank Transfer, Bill Pay — all following the saga pattern (amount → PIN → execute → result).

5. **Add sub-wallet system**: 5 types with distinct models. NCMC (balance cap), FASTag (security deposit), Food/Fuel (employer-only), Gift (universal). Collapsible vertical list in WalletStrip.

6. **Add compliance layer**: Load Guard (3 RBI rules), KYC state machine, P2P restrictions for Min-KYC.

7. **Build admin dashboard**: Dashboard metrics, User Management (search/filter/detail), Transaction Monitoring, KYC Queue, Benefits Management (bulk load + utilisation).

8. **Add AI layer**: MCP server with 39 tools, Express API for Claude chat/summarisation, KYC expiry alerts.

9. **Wire up data sync**: Wallet app → admin dashboard (dev) and → Render backend (prod).

10. **Deploy**: GitHub Pages for both frontends, Render for API server.
