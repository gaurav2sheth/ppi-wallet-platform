# PPI Wallet Platform -- System Architecture

## 1. Overview

The PPI Wallet Platform is an RBI-regulated Prepaid Payment Instrument (PPI) wallet system built as a multi-repo workspace. It consists of a consumer-facing mobile-first wallet app, an admin operations dashboard, a Model Context Protocol (MCP) server exposing 39 AI tools for Claude, and an Express.js REST API server that integrates Claude AI for chat, summarisation, and KYC alerts.

The platform enforces Indian regulatory compliance (RBI PPI Master Directions) at every layer, from wallet load validation to KYC lifecycle management and AML/CFT thresholds.

---

## 2. Ecosystem & Repository Structure

| Repo / Directory | Tech Stack | Port | Purpose |
|---|---|---|---|
| `paytm-wallet-app/` | React 19 + Vite 8 + Tailwind CSS v4 + Zustand + Axios | 5173 | Consumer wallet UI (mobile-first SPA) |
| `admin-dashboard/` | React 19 + Vite 8 + Ant Design v5 + Zustand | 5174 | Admin operations dashboard (desktop) |
| `mcp/` | Node.js + `@modelcontextprotocol/sdk` + Zod | stdio | 49 Claude AI tools via MCP protocol (8 categories) |
| `api-server/` | Express.js + `@anthropic-ai/sdk` + CORS | 3001 | REST API (chat, support/KYC agents, load guard, sub-wallets — 28 routes) |
| `ppi-wallet-api-deploy/` | Express.js | Render | Production deploy mirror of api-server |

### Git Repositories (4 Separate Repos)

```
paytm-wallet-app/       -> github.com/gaurav2sheth/ppi-wallet-app
admin-dashboard/         -> github.com/gaurav2sheth/ppi-wallet-admin-dashboard
mcp/                     -> github.com/gaurav2sheth/ppi-wallet-mcp
api-server/              -> (local only, deployed via ppi-wallet-api-deploy)
ppi-wallet-api-deploy/   -> github.com/gaurav2sheth/ppi-wallet-api-deploy
```

### Live URLs

| Target | URL |
|---|---|
| Wallet App | https://gaurav2sheth.github.io/ppi-wallet-app |
| Admin Dashboard | https://gaurav2sheth.github.io/ppi-wallet-admin |
| API | https://ppi-wallet-api.onrender.com/health |

---

## 3. High-Level Architecture Diagram

```
+---------------------+        +------------------------+
|  Wallet App (React)  |        |  Admin Dashboard (React)|
|  Port 5173           |        |  Port 5174              |
|  Tailwind + Zustand  |        |  Ant Design + Zustand   |
+----------+----------+        +-----------+------------+
           |                                |
           |  3-Tier API Fallback           |  Fetches from Render API
           |  (Vite MW -> Express -> Mock)  |  with mock fallback
           v                                v
+----------------------------------------------------------+
|                Express.js API Server (Port 3001)          |
|  /api/chat, /api/summarise-transactions,                  |
|  /api/kyc-alerts/*, /api/wallet/validate-load,            |
|  /api/wallet/sub-wallets/*, /api/wallet/benefits/*        |
+----+----------------------------+------------------------+
     |                            |
     v                            v
+--------------------+   +------------------------+
|  MCP Server (stdio) |   |  Claude AI (Anthropic)  |
|  39 tools, Zod      |   |  Sonnet 4 + Haiku 4.5   |
|  wallet-mcp-server.js|   +------------------------+
+----+----------------+
     |
     v
+--------------------+
|  Mock Data Layer    |
|  mock-data.js       |
|  200 users, 500+    |
|  transactions       |
|  BigInt paise math  |
+--------------------+
```

---

## 4. API Server Architecture

### 4.1 Express.js REST Endpoints (12 Total)

| Method | Path | Description |
|---|---|---|
| GET | `/health` | Health check (18 tools, v1.0.0) |
| POST | `/api/chat` | Claude AI chat (query param: `role=admin\|user`) |
| POST | `/api/summarise-transactions` | AI transaction summary (Claude Sonnet) |
| GET | `/api/kyc-alerts/preview` | Preview at-risk KYC users |
| POST | `/api/kyc-alerts/run` | Execute KYC alerts (Claude Haiku generates messages) |
| POST | `/api/wallet/validate-load` | Load Guard: validate against 3 RBI rules |
| GET | `/api/wallet/load-guard-log` | Blocked load attempts log |
| GET | `/api/wallet/sub-wallets/:userId` | All sub-wallets for a user |
| POST | `/api/wallet/sub-wallets/load` | Employer loads a sub-wallet |
| POST | `/api/wallet/sub-wallets/spend` | Spend from sub-wallet (with eligibility check) |
| GET | `/api/wallet/sub-wallets/eligibility` | Check merchant category vs sub-wallet type |
| GET | `/api/wallet/benefits/utilisation` | Aggregate utilisation stats for admin |

### 4.2 CORS Whitelist

```javascript
origin: [
  'https://gaurav2sheth.github.io',  // production (both apps)
  'http://localhost:5173',             // wallet app dev
  'http://localhost:5174',             // admin dashboard dev
]
```

### 4.3 Chat Handler -- Agentic Tool Loop

The chat handler (`mcp/chat-handler.js`) implements an agentic loop that:

1. Receives a natural language question from the user or admin.
2. Sends it to Claude Sonnet 4 with all 30 tool definitions and a role-specific system prompt.
3. If Claude responds with `tool_use` blocks, executes each tool call against the mock data layer.
4. Pushes tool results back as `tool_result` messages.
5. Loops up to 5 iterations until Claude returns a final text response (`stop_reason: 'end_turn'`).

The system prompt is differentiated by role:
- **User role**: Friendly personal finance assistant, assumes `user_001`, uses INR formatting.
- **Admin role**: Data-driven financial analyst, receives `<app_context>` block with dashboard-visible users and transactions to avoid unnecessary tool calls.

### 4.4 Transaction Summarisation

`POST /api/summarise-transactions` accepts an array of transaction objects, formats them into numbered lines (date, type, amount, description), and sends to Claude Sonnet 4 for a 3-5 line plain English summary. The system prompt is differentiated by user vs admin role.

---

## 5. MCP Server Architecture

### 5.1 Overview

The MCP server (`mcp/wallet-mcp-server.js`) exposes 39 tools to Claude Desktop via the Model Context Protocol stdio transport. It uses `@modelcontextprotocol/sdk` for the server framework and Zod for input schema validation.

### 5.2 Claude Desktop Configuration

```json
{
  "mcpServers": {
    "ppi-wallet": {
      "command": "/usr/local/bin/node",
      "args": ["/Users/GauravSheth/Documents/PPI_Wallet/mcp/wallet-mcp-server.js"],
      "env": {
        "DATABASE_URL": "postgresql://ppsl_user:change_me@localhost:5432/ppsl_wallet_dev"
      }
    }
  }
}
```

### 5.3 Tool Categories (39 Total)

#### User Tools (12)

| Tool | Description |
|---|---|
| `get_wallet_balance` | Balance, status, optional runway estimation (avg daily spend, days remaining) |
| `get_transaction_history` | Filtered transactions with pagination (entry_type, transaction_type, limit, offset) |
| `flag_suspicious_transaction` | Mark a transaction as suspicious with reason |
| `unflag_transaction` | Remove suspicious flag with reason |
| `get_spending_summary` | Spending breakdown by category or merchant (top N) |
| `search_transactions` | Search by keyword, merchant name, or amount range |
| `get_user_profile` | Full profile with KYC tier, limits, activity stats, optional limit utilization |
| `compare_spending` | Compare spending between two consecutive time periods |
| `detect_recurring_payments` | Identify subscription/recurring merchant patterns |
| `generate_report` | Comprehensive report: summary, detailed, or risk-focused |
| `get_notifications` | User notifications (unread filter, pagination) |
| `set_alert_threshold` | Set low balance, high transaction, daily spend alerts |

#### Transaction Tools (5)

| Tool | Description |
|---|---|
| `add_money` | Load wallet (validates KYC balance limits), source: UPI/Debit Card/NEFT |
| `pay_merchant` | Merchant payment (checks balance) |
| `transfer_p2p` | P2P transfer (validates balance + P2P limits) |
| `pay_bill` | Bill payment (electricity, phone, etc.) |
| `request_refund` | Initiate refund for a debit transaction |

#### Admin Tools (10)

| Tool | Description |
|---|---|
| `get_system_stats` | Platform-wide statistics |
| `search_users` | Search users by various criteria |
| `get_flagged_transactions` | All flagged/suspicious transactions |
| `suspend_user` | Suspend a user account |
| `get_failed_transactions` | Failed transaction log |
| `get_kyc_stats` | KYC pipeline statistics |
| `check_compliance` | RBI PPI compliance check with optional 0-100 risk score |
| `compare_users` | Side-by-side user comparison (balance, spending, income) |
| `get_peak_usage` | Platform-wide peak usage analysis (hours, days, types) |
| `get_monthly_trends` | Month-over-month trends (volumes, active users, growth) |

#### KYC Tools (5)

| Tool | Description |
|---|---|
| `approve_kyc` | Admin: Approve pending Full KYC application |
| `reject_kyc` | Admin: Reject pending Full KYC with reason |
| `request_kyc_upgrade` | Initiate MIN_KYC to FULL_KYC upgrade |
| `query_kyc_expiry` | Flexible query with date ranges, urgency bands, balance thresholds |
| `generate_kyc_renewal_report` | Comprehensive compliance report with executive summary |

#### Support Tools (3)

| Tool | Description |
|---|---|
| `raise_dispute` | File a dispute (types: failed_transaction, unauthorized, wrong_amount, merchant_issue, other) |
| `get_dispute_status` | Dispute status or list all disputes |
| `get_refund_status` | Refund status or list all refunds |

#### Sub-Wallet Tools (4)

| Tool | Description |
|---|---|
| `get_sub_wallets` | List sub-wallets with balances |
| `load_sub_wallet` | Employer loads a sub-wallet |
| `get_sub_wallet_transactions` | Sub-wallet transaction history |
| `validate_merchant_eligibility` | Check merchant category vs sub-wallet type |

### 5.4 Mock Data Layer (`mock-data.js`)

- **200 users** with wallets (seeded with deterministic PRNG, seed 42)
- **500+ transactions** spanning multiple days
- **Sub-wallets** for users with 5 types (seeded with PRNG seed 500)
- **Employers**: `employer_001` (Paytm), `employer_002` (TCS)
- All balances use **BigInt arithmetic** internally, serialised as strings at API boundary
- **41 exported functions** for data access and mutation

---

## 6. AI-Powered Services

### 6.1 Wallet Load Guard Service

**File**: `mcp/services/wallet-load-guard.js`

Validates wallet load amounts against 3 RBI PPI rules **before** processing:

| Rule | Name | Limit | Scope |
|---|---|---|---|
| Rule 1 | `BALANCE_CAP` | Rs 1,00,000 max wallet balance | Main wallet + ALL sub-wallet balances combined |
| Rule 2 | `MONTHLY_LOAD` | Rs 2,00,000 total loads per calendar month | All load transactions in current month |
| Rule 3 | `MIN_KYC_CAP` | Rs 10,000 max balance for Minimum KYC users | Total balance (main + sub-wallets) |

**Flow**:
1. Fetch user's main balance and sub-wallet balances.
2. Calculate monthly loaded total from transaction history.
3. Check all 3 rules; collect violations.
4. If blocked, the most restrictive rule (lowest `max_allowed`) wins.
5. Call Claude Haiku 4.5 to generate a friendly, user-facing explanation (with graceful fallback to template message).
6. Log blocked attempt to in-memory ring buffer (last 10 attempts).

**RBI Limits (in paise)**:
```javascript
const BALANCE_CAP_PAISE     = 10000000;  // Rs 1,00,000
const MONTHLY_LOAD_LIMIT_PAISE = 20000000; // Rs 2,00,000
const MIN_KYC_BALANCE_CAP_PAISE = 1000000; // Rs 10,000
```

### 6.2 KYC Expiry Alert Service

**File**: `mcp/services/kyc-alert-service.js`

4-step pipeline for KYC expiry outreach:

1. **Fetch at-risk users**: Query KYC expiry with `urgency: 'critical'` (expiring within 7 days), exclude expired and inactive.
2. **Generate personalised messages**: Call Claude Haiku 4.5 per user to draft SMS/WhatsApp messages (<160 chars) mentioning name, expiry date, at-risk balance, and CTA for Full KYC upgrade.
3. **Simulate sending**: Log SMS delivery to console with formatted output.
4. **Generate ops summary**: Claude Haiku produces a 3-line Slack notification for the compliance team.

Graceful fallback at every step: if Claude API fails, a template message is used.

### 6.3 KYC Alert Scheduler

**File**: `mcp/services/scheduler.js`

- Uses `node-cron` to schedule the KYC Alert Service daily at **9:00 AM IST** (cron: `30 3 * * *` UTC).
- Supports `--run-now` flag for immediate execution in standalone mode.
- Requires `ANTHROPIC_API_KEY` environment variable.

### 6.4 Claude Models Used

| Model | Usage |
|---|---|
| `claude-sonnet-4-20250514` | Agentic chat (tool calling), transaction summarisation |
| `claude-haiku-4-5-20251001` | Load Guard block messages, KYC alert SMS/WhatsApp, ops summaries |

---

## 7. Consumer Wallet App Architecture

### 7.1 Tech Stack

- React 19 + TypeScript + Vite 8
- Tailwind CSS v4 (utility-first styling)
- Zustand (state management with localStorage persistence)
- Axios (HTTP client)
- HashRouter (required for GitHub Pages deployment)

### 7.2 Routes (21 Pages)

| Path | Page | Description |
|---|---|---|
| `/login` | LoginPage | Mock login (any 10-digit phone) |
| `/` | HomePage | Dashboard with wallet card + quick actions |
| `/wallet` | WalletOverviewPage | Wallet balance + sub-wallet list |
| `/wallet/detail` | WalletDetailPage | Detailed balance breakdown |
| `/wallet/add-money` | PaymentGatewayPage | Add money flow with Load Guard |
| `/wallet/sub/:type` | SubWalletDetailPage | Sub-wallet detail (add money, txn history, AI) |
| `/wallet/statement` | WalletStatementPage | Full transaction statement |
| `/pay` | MerchantPayPage | Scan & pay merchant |
| `/send` | SendMoneyPage | P2P transfer |
| `/transfer-bank` | TransferToBankPage | Wallet to bank transfer |
| `/bill-pay` | BillPayPage | Bill payments |
| `/passbook` | PassbookPage | Transaction history with filters |
| `/analytics` | SpendAnalyticsPage | Spending analytics & charts |
| `/budget` | BudgetPage | Budget category limits |
| `/kyc` | KycStatusPage | KYC tier status & upgrade |
| `/profile` | ProfilePage | User profile |
| `/transaction` | TransactionDetailPage | Single transaction detail |
| `/notifications` | NotificationsPage | Alert messages |
| `/rewards` | RewardsPage | Scratch cards & cashback |

### 7.3 Zustand Stores (7)

| Store | File | Purpose |
|---|---|---|
| Auth | `auth.store.ts` | Login state (walletId, userId, userName, phone) |
| Wallet | `wallet.store.ts` | Balance, sub-wallets, ledger |
| PIN | `pin.store.ts` | 4-digit PIN (default: 1234) |
| Budget | `budget.store.ts` | Budget category limits |
| Payees | `payees.store.ts` | Recent contacts by type (max 20 each) |
| Notifications | `notifications.store.ts` | Alert messages |
| Rewards | `rewards.store.ts` | Scratch cards + cashback |

### 7.4 API Layer (3-Tier Fallback)

The wallet app uses a 3-tier API fallback strategy so it always works, even offline:

```
Tier 1: Vite dev server middleware (local development)
    |
    v (if unavailable)
Tier 2: Express.js API server (Render production or local :3001)
    |
    v (if unavailable)
Tier 3: Client-side mock (localStorage-based, always available)
```

**API Files**:

| File | Purpose |
|---|---|
| `src/api/client.ts` | Axios client with reachability check |
| `src/api/saga.api.ts` | Transaction execution with 3-tier fallback |
| `src/api/mock.ts` | Client-side mock with localStorage persistence |
| `src/api/wallet.api.ts` | Wallet balance and info |
| `src/api/loadguard.api.ts` | Load Guard validation |
| `src/api/kyc.api.ts` | KYC operations |
| `src/api/limits.api.ts` | Limit checking |

### 7.5 Saga Pattern (Transaction Flow)

All financial transactions follow the saga pattern:

1. **Amount Input** -- User enters amount
2. **PinModal** -- PIN verification (default: 1234)
3. **Saga API Call** -- `sagaPost()` with idempotency key
4. **Result Screen** -- Success confirmation or retry option

The `sagaPost` function attempts the Express API first; on failure, falls through to the client-side mock.

**Saga Types**: `ADD_MONEY`, `MERCHANT_PAY`, `P2P_TRANSFER`, `WALLET_TO_BANK`, `BILL_PAY`, `REFUND`

**SagaResponse Schema**:
```typescript
{
  saga_id: string;
  saga_type: string;
  status: 'STARTED' | 'RUNNING' | 'COMPLETED' | 'COMPENSATING' | 'COMPENSATED' | 'DLQ';
  result?: { balance_after_paise: string; entry_id: string };
  error?: string;
  steps: Step[];
  created_at: string;
  updated_at: string;
}
```

### 7.6 Vite Dev Server Middleware

The wallet app's `vite.config.ts` registers a custom plugin (`walletSyncPlugin`) that provides several server-side endpoints during development:

| Endpoint | Method | Description |
|---|---|---|
| `/api/sync` | POST | Wallet app pushes events (login, transaction, balance update, KYC update) to shared JSON file |
| `/api/sync-status` | GET | Read current shared data (for debugging) |
| `/api/chat` | POST | Natural language AI chat via `chat-handler.js` |
| `/api/wallet/validate-load` | POST | Load Guard RBI limit validation |
| `/api/wallet/load-guard-log` | GET | Blocked attempts log |
| `/api/summarise-transactions` | POST | Claude AI transaction summariser |

The shared data file at `.shared-data/wallet-events.json` enables the admin dashboard to see real-time wallet app events during local development.

### 7.7 Design System

Paytm PODS design language:

| Element | Value |
|---|---|
| Navy (primary) | `#002E6E` |
| Cyan (accent) | `#00B9F1` |
| Green (success) | `#12B76A` |
| Layout | Card-based, max-width 480px centered on desktop |
| Navigation | Bottom nav: Home / Scan & Pay / Passbook / Profile |
| Buttons | Pill-style |

---

## 8. Admin Dashboard Architecture

### 8.1 Tech Stack

- React 19 + TypeScript + Vite 8
- Ant Design v5 (enterprise component library)
- Zustand (state management)
- HashRouter (GitHub Pages)

### 8.2 Routes & Permissions

| Route | Page | Required Permission |
|---|---|---|
| `/login` | LoginPage | -- |
| `/` | DashboardPage | All roles |
| `/users` | UsersPage | `users.view` |
| `/users/:id` | UserDetailPage | `users.view` |
| `/transactions` | TransactionsPage | `transactions.view` |
| `/transactions/:id` | TransactionDetailPage | `transactions.view` |
| `/kyc` | KycPage | `kyc.view` |
| `/benefits` | BenefitsPage | `dashboard.view` |
| `/settings` | SettingsPage | `settings.view` |
| `/403` | ForbiddenPage | -- |

### 8.3 Role-Based Access Control (RBAC)

| Role | Permissions |
|---|---|
| `SUPER_ADMIN` | All permissions |
| `BUSINESS_ADMIN` | Dashboard, users (read), transactions (read), analytics, KYC (read) |
| `OPS_MANAGER` | Dashboard, users, transactions, KYC (approve/reject) |
| `CS_AGENT` | Dashboard, users (read), transactions (read) |
| `COMPLIANCE_OFFICER` | Dashboard, users (read), transactions, KYC, analytics |
| `MARKETING_MANAGER` | Dashboard, analytics |

### 8.4 Mock Credentials

| Role | Username | Password |
|---|---|---|
| Super Admin | `admin` | `admin123` |
| Business Admin | `business` | `admin123` |
| CS Agent | `support` | `admin123` |

### 8.5 Key Components

| Component | Purpose |
|---|---|
| `AdminLayout` | Sidebar + content area, collapsible navigation |
| `AdminAuthGuard` | Route protection with permission checking (renders `<Outlet>` or redirects to `/403`) |
| `BulkLoadPanel` | Employer bulk-loads sub-wallets (select type, amount, preview, execute) |
| `UtilisationDashboard` | Sub-wallet metrics: loaded, spent, utilisation rate, expiry |
| `AiBenefitsInsight` | Claude-powered utilisation insights and recommendations |

### 8.6 Data Source

The admin dashboard fetches from the Render-hosted API (`ppi-wallet-api.onrender.com`) with a client-side mock fallback. In development mode, Vite middleware proxies `/api/sync` for real-time wallet app events.

---

## 9. Sub-Wallet System (Corporate Benefits)

### 9.1 Overview

The sub-wallet system manages purpose-locked corporate benefit wallets loaded by employers for employees. All sub-wallet balances count toward the Rs 1,00,000 RBI PPI main wallet cap.

### 9.2 Sub-Wallet Types

| Type | Icon | Monthly Limit | Self-Load | Eligible Categories | RBI Category |
|---|---|---|---|---|---|
| FOOD | Meal box | Rs 3,000 | No (employer-only) | Restaurants, Cafes, Food delivery, Swiggy, Zomato, Canteen | Meal voucher equivalent |
| NCMC TRANSIT | Metro | Rs 2,000 | Yes (up to Rs 3K balance cap) | Metro, Bus, Local train, Parking, Transit | NCMC Transit Card |
| FASTAG | Highway | Rs 10,000 | Yes (top-up flow) | NHAI toll plazas, FASTag recharge portals | Prepaid transit instrument |
| GIFT | Gift box | No limit (per occasion) | Yes (no cap) | All retail merchants (universal) | Gift instrument |
| FUEL | Fuel pump | Rs 2,500 | No (employer-only) | HP, Indian Oil, BPCL, Shell | Petroleum product PPI |

### 9.3 Special Mechanics

#### NCMC Transit
- Own Rs 3,000 **balance limit** (max balance at any time, not monthly)
- Self-load from main wallet allowed up to the cap
- Transit payments use **ONLY NCMC balance** -- no cascade to main wallet
- If NCMC balance insufficient, payment fails
- Interface field: `max_balance_paise: 300000`

#### FASTag (Security Deposit Model)
- Sub-wallet holds **Rs 300 security deposit per vehicle**
- Toll transactions **deduct from MAIN WALLET first**
- If main wallet is zero, security deposit is consumed as last resort
- On next top-up (Add Money on FASTag page):
  1. Refill security deposit first (up to Rs 300 x vehicle count)
  2. Remainder goes to main wallet
- "New Vehicle" = issue FASTag + Rs 300 deposit from main wallet
- Interface fields: `is_security_deposit: true`, `vehicle_count`, `security_deposit_per_vehicle_paise: 30000`, `security_deposit_used_paise`

#### Gift Wallet
- Employer-loaded with expiry date (typically 1 year)
- Self-load from main wallet allowed (no cap)
- Universal merchant eligibility
- Shows `EXPIRED` badge when past `expiry_date`

### 9.4 Cascade Spend Logic (Merchant Pay)

1. Auto-detect merchant category via MCC keyword matching (`src/utils/mcc.ts` -- 19 categories)
2. Check specific wallets first (Food/Transit/Fuel) based on category
3. Gift wallet as universal fallback
4. If sub-wallet balance < payment amount: split spend (sub-wallet + remainder from main wallet)

### 9.5 Employer Data

| Employer ID | Name | Allowed Sub-Wallet Types |
|---|---|---|
| `employer_001` | Paytm | All types |
| `employer_002` | TCS | All types |

### 9.6 Utilisation Dashboard API

`GET /api/wallet/benefits/utilisation` returns aggregate stats across all users:

```json
{
  "total_loaded": "Rs X,XXX",
  "total_spent": "Rs X,XXX",
  "utilisation_rate": 65,
  "expiring_this_month": "Rs X,XXX",
  "by_type": [
    {
      "type": "FOOD",
      "loaded": "Rs X,XXX",
      "spent": "Rs X,XXX",
      "remaining": "Rs X,XXX",
      "utilisation_percent": 72
    }
  ]
}
```

---

## 10. RBI PPI Compliance Rules

These rules are non-negotiable and enforced in all wallet operations.

### 10.1 KYC Tiers & Limits

| Rule | Min-KYC | Full-KYC |
|---|---|---|
| Balance cap | Rs 10,000 | Rs 2,00,000 |
| Monthly load cap | Rs 10,000 | Rs 1,00,000 |
| Annual load cap | Rs 1,20,000 | -- |
| P2P transfer | PROHIBITED (HTTP 403) | Rs 1,00,000/month |
| Wallet validity | 12 months then EXPIRED | Unlimited |
| Cash withdrawal | Not allowed | Rs 2,000/txn, Rs 10,000/month |

### 10.2 KYC State Machine

```
UNVERIFIED --> MIN_KYC (mobile OTP)
MIN_KYC --> FULL_KYC_PENDING (Aadhaar OTP submitted)
FULL_KYC_PENDING --> FULL_KYC (approved) | REJECTED (denied)
Any --> SUSPENDED (fraud/compliance)
MIN_KYC --> EXPIRED (12-month validity, 60-day grace period)
```

### 10.3 Wallet Load Guard (3 Rules on Every Add Money)

1. **BALANCE_CAP** -- Main wallet + ALL sub-wallet balances combined <= Rs 1,00,000
2. **MONTHLY_LOAD** -- Calendar month total loads <= Rs 2,00,000
3. **MIN_KYC_CAP** -- Min-KYC wallets: total balance <= Rs 10,000

### 10.4 Wallet Lifecycle States

```
ACTIVE --> SUSPENDED (fraud hold)
ACTIVE --> DORMANT (12 months no transactions)
DORMANT --> CLOSED (24 months, balance to suspense account)
MIN_KYC --> EXPIRED (12-month validity, 60-day grace)
Any --> CLOSED (user-initiated, T+1 balance refund)
```

### 10.5 AML/CFT Thresholds

| Threshold | Value |
|---|---|
| CTR (Cash Transaction Report) | Cash transactions > Rs 10 lakh/day |
| STR (Suspicious Transaction Report) | Suspicious activity, file within 7 days |
| EDD (Enhanced Due Diligence) | Triggered at Rs 50,000/month load |
| Data retention (ledger) | 5 years |
| Data retention (KYC artefacts) | 10 years |
| Dispute SLA | 30 days, auto-close in customer favour at day 30+1 |

---

## 11. Data Models

### 11.1 LedgerEntry (Append-Only Transaction Record)

```typescript
interface LedgerEntry {
  id: string;
  entry_type: 'CREDIT' | 'DEBIT' | 'HOLD' | 'HOLD_RELEASE';
  amount_paise: string;
  balance_after_paise: string;
  held_paise_after: string;
  transaction_type: string;  // ADD_MONEY | MERCHANT_PAY | P2P_TRANSFER | WALLET_TO_BANK | BILL_PAY | REFUND
  reference_id: string;
  description: string;
  idempotency_key: string;
  hold_id?: string;
  created_at: string;
  payment_source?: string;
}
```

### 11.2 SubWallet

```typescript
interface SubWallet {
  sub_wallet_id: string;
  type: string;           // FOOD | NCMC TRANSIT | FASTAG | GIFT | FUEL
  icon: string;
  color: string;
  label: string;
  balance_paise: number;
  status: string;         // ACTIVE | EXPIRED
  monthly_limit_paise: number;
  monthly_loaded_paise: number;
  loaded_by: string;
  last_loaded_at: string;
  expiry_date: string | null;
  eligible_categories: string[];
  transactions: SubWalletTransaction[];
  // NCMC-specific
  max_balance_paise?: number;        // 300000 (Rs 3,000)
  // FASTag-specific
  is_security_deposit?: boolean;
  vehicle_count?: number;
  security_deposit_per_vehicle_paise?: number;  // 30000 (Rs 300)
  security_deposit_used_paise?: number;
}
```

### 11.3 SagaResponse

```typescript
interface SagaResponse {
  saga_id: string;
  saga_type: string;
  status: 'STARTED' | 'RUNNING' | 'COMPLETED' | 'COMPENSATING' | 'COMPENSATED' | 'DLQ';
  result?: {
    balance_after_paise: string;
    entry_id: string;
  };
  error?: string;
  steps: Step[];
  created_at: string;
  updated_at: string;
}
```

### 11.4 localStorage Keys (Wallet App)

| Key | Content |
|---|---|
| `__mock_balance` | Main wallet balance in paise (string) |
| `__mock_ledger` | Transaction history (`LedgerEntry[]`) |
| `__mock_sub_wallets_v2` | Sub-wallet data with NCMC/FASTag fields |
| `wallet_pin` | 4-digit PIN (default: 1234) |
| `wallet_auth` | Login state (walletId, userId, userName, phone) |
| `wallet_budget` | Budget category limits |
| `wallet_payees` | Recent contacts by type (max 20 each) |
| `wallet_notifications` | Alert messages |
| `wallet_rewards` | Scratch cards + cashback |

### 11.5 Default Seed Data

- Main wallet: Rs 236.11 (23611 paise)
- 12 historical transactions spanning 20 days
- 5 sub-wallets: Food Rs 1,200, NCMC Rs 1,800, FASTag Rs 600 (2 vehicles), Gift Rs 2,000, Fuel Rs 1,500

---

## 12. Key Architecture Decisions

1. **3-Tier API Fallback**: Vite middleware -> Express API -> client-side mock. The app always works, even offline. This ensures the demo is resilient during presentations.

2. **Saga Pattern**: All financial transactions follow Amount -> PIN -> API call -> Result screen (success/retry). Idempotency keys on every transaction POST prevent duplicates.

3. **Sub-Wallet Balances Included in RBI Cap**: The Rs 1,00,000 limit = main wallet + all sub-wallet balances combined. This is enforced at both the Load Guard service and the sub-wallet load function.

4. **FASTag = Security Deposit Model**: Tolls deduct from main wallet first; the security deposit is a last resort. On next top-up, security deposit is refilled before main wallet.

5. **NCMC = Isolated Balance**: Rs 3,000 cap, transit-only, never cascades to main wallet. If NCMC balance is insufficient, the payment fails rather than cascading.

6. **localStorage Versioning**: Sub-wallet key is `__mock_sub_wallets_v2` to force re-seed on schema changes. Incrementing the version number clears stale data.

7. **BigInt Paise Arithmetic**: All monetary calculations in the MCP mock-data layer use BigInt internally to avoid floating-point errors. Values are serialised as strings at the API boundary.

8. **Amounts in Paise**: All monetary values are integers in paise (1 INR = 100 paise). Never use floats for money. Display formatted with `formatPaise()` using `en-IN` locale.

9. **UUID Entity IDs**: All entity IDs use UUIDs. Idempotency keys on every transaction POST.

10. **ISO 8601 UTC Timestamps**: All timestamps are ISO 8601 UTC format throughout the system.

---

## 13. Currency Formatting

```typescript
// src/utils/format.ts
formatPaise(paise: string | number): string
// Returns: "Rs X,XX,XXX.XX" using en-IN locale
```

All UI display of monetary values MUST use `formatPaise()`. Never format manually.

---

## 14. Deployment Architecture

### 14.1 Wallet App

- **Build**: Vite production build (`vite build`)
- **Host**: GitHub Pages at `gaurav2sheth.github.io/ppi-wallet-app`
- **Deploy**: `gh-pages -d dist`
- **Base path**: `/ppi-wallet-app/`

### 14.2 Admin Dashboard

- **Build**: Vite production build (`vite build`)
- **Host**: GitHub Pages at `gaurav2sheth.github.io/ppi-wallet-admin`
- **Deploy**: `gh-pages -d dist`

### 14.3 API Server

- **Deploy repo**: `ppi-wallet-api-deploy/` (mirrors `api-server/server.js` + `mcp/` directory)
- **Host**: Render (auto-deploys on push to GitHub)
- **URL**: `https://ppi-wallet-api.onrender.com`
- **Environment**: Requires `ANTHROPIC_API_KEY` env var on Render

### 14.4 MCP Server

- **Host**: Local machine via Claude Desktop
- **Transport**: stdio
- **No deployment needed** -- runs locally when Claude Desktop connects

---

## 15. Development Setup

### Quick Start

```bash
# Wallet app (works standalone with built-in mock data)
cd paytm-wallet-app && npm install && npm run dev

# Admin dashboard
cd admin-dashboard && npm install && npm run dev

# API server (requires ANTHROPIC_API_KEY)
cd api-server && npm install && npm run dev
```

### Type Checking & Build

```bash
# Wallet app
cd paytm-wallet-app
/usr/local/bin/node ./node_modules/.bin/tsc --noEmit   # type-check
/usr/local/bin/node ./node_modules/.bin/vite build      # build

# Admin dashboard
cd admin-dashboard
/usr/local/bin/node ./node_modules/.bin/tsc --noEmit
/usr/local/bin/node ./node_modules/.bin/vite build
```

### Node Path

The system uses `/usr/local/bin/node` (no nvm).

---

## 16. MCP Service Files Reference

| File | Purpose |
|---|---|
| `mcp/wallet-mcp-server.js` | All 39 tool definitions with Zod schemas for Claude Desktop |
| `mcp/mock-data.js` | 200 seeded users, 500+ transactions, BigInt paise arithmetic, 41 exported functions |
| `mcp/chat-handler.js` | Claude Sonnet 4 chat with agentic tool orchestration loop (max 5 iterations) |
| `mcp/services/sub-wallet-service.js` | Sub-wallet CRUD: load, spend, eligibility, utilisation |
| `mcp/services/wallet-load-guard.js` | 3 RBI rules + Claude Haiku 4.5 suggestions on block |
| `mcp/services/kyc-alert-service.js` | KYC expiry detection + Claude Haiku personalised SMS/WhatsApp messages |
| `mcp/services/scheduler.js` | node-cron daily 9 AM IST scheduler for KYC alerts |
| `mcp/test-tools.js` | Tool integration tests |
| `mcp/chat-handler.d.ts` | TypeScript declarations for chat handler |
| `mcp/claude_desktop_config.json` | Claude Desktop MCP server configuration |
