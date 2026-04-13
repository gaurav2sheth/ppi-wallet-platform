# PPI Wallet Product – Key Components & Architecture

# 1. Core Wallet Architecture (Ledger-first design)

- Wallet Ledger (double-entry system)- Balance Service (real-time balance)- Transaction Orchestrator- Idempotency LayerDesign principle: Ledger is the single source of truth

# 2. RBI Compliance & PPI Types

- Minimum KYC Wallet (₹10,000 limit)- Full KYC Wallet (₹2 lakh limit, interoperability)- Gift / Closed WalletCompliance: KYC, AML monitoring, limits engine, audit logs

# 3. User Identity & KYC Stack

- User Profile Service- KYC Orchestrator (Aadhaar, PAN, Video KYC)- Risk Tiering Engine

# 4. Money Movement Flows

- Add Money (UPI, Cards, Net banking)- Spend (merchant payments, P2P, bill pay)- Withdraw / Transfer (bank, UPI)- Refunds handling

# 5. Banking & Escrow Layer

- Escrow account integration- Nodal account management- Daily reconciliation- Settlement engine

# 6. Risk, Fraud & Security

- Transaction risk scoring- Velocity checks- Device fingerprinting- OTP / 2FA authentication

# 7. Limits & Rules Engine

- Transaction limits- KYC-based rules- Configurable system via admin panel

## Wallet Load Guard — Pre-Transaction Validation

Wallet Load Guard — RBI PPI Limit Validation

The Wallet Load Guard is a pre-transaction validation layer that enforces RBI PPI limits before any wallet load (add money) transaction is processed. It runs 3 compliance checks and generates Claude AI explanations when a transaction is blocked.

Three RBI Rules Validated:

1. BALANCE_CAP — Maximum wallet balance of ₹1,00,000 for all users

2. MONTHLY_LOAD — Maximum ₹2,00,000 total loads per calendar month

3. MIN_KYC_CAP — Maximum ₹10,000 balance for Minimum KYC users

Architecture:

- Backend Service: mcp/services/wallet-load-guard.js — validates load amounts, calculates monthly loads from transaction history, calls Claude Haiku API (claude-haiku-4-5-20251001) for user-friendly block explanations

- API Route: POST /api/wallet/validate-load — accepts user_id and amount (₹), returns allowed/blocked status with max_allowed amount

- Admin Visibility: GET /api/wallet/load-guard-log — returns last 10 blocked attempts (in-memory ring buffer)

- Client Fallback: mockValidateLoad() in wallet app for offline validation

User-Facing Flow:

- Add Money page shows live "You can add up to ₹X right now" helper text

- On Continue, calls validate-load API BEFORE payment processing

- If blocked: red alert with Claude-generated explanation + bold suggestion + "Add ₹X instead" button

- If max_allowed is 0 and blocked by MIN_KYC_CAP: shows "Upgrade to Full KYC" button

Admin Dashboard:

- Load Guard Log panel on the Dashboard page showing last 10 blocked attempts

- Columns: User, Attempted Amount, Blocked By (color-coded tag), Max Allowed, Timestamp

- Read-only, refreshes on page load

Most restrictive rule wins — if multiple rules fail, the one with the lowest max_allowed amount is reported to the user.

# 8. Reconciliation & Settlement

- Internal vs bank reconciliation- PG reconciliation- Exception handling workflows

# 9. Reporting & Compliance

- RBI reports- Suspicious transaction reports- Business metrics dashboards

# 10. Merchant Ecosystem

- Merchant onboarding- QR issuance- Payment APIs- Settlement systems

# 11. AI / Differentiation Layer

- Smart spend insights- Fraud detection using ML- Conversational wallet- Personalized offers

**Implementation Status (April 2026):**

The AI layer has been implemented via the Model Context Protocol (MCP) with 35 tools across 9 categories. The system includes:

- MCP Server (wallet-mcp-server.js): 35 tools accessible via Claude Desktop for natural language wallet management

- Chat Handler (chat-handler.js): Agentic AI with multi-turn tool use supporting user and admin roles

- Mock Data Layer (mock-data.js): 200 users + 500+ transactions with 41 exported functions

- Render Backend: Express.js at ppi-wallet-api.onrender.com serving MCP data via 13 REST endpoints

- AI Transaction Summarizer: Claude API integration for admin dashboard pattern analysis

- KYC Expiry Intelligence: 3 dedicated tools for expiry filtering, database queries, and compliance reports

- Spend Analytics: MCC-based category mapping with 19 categories and merchant insights

- Smart insights: Balance runway estimation, recurring payment detection, spending comparisons

# 12. Frontend Experiences

- Consumer app (wallet, payments)- Merchant dashboard- Admin panel

**Current Implementation (April 2026):**

- Consumer Wallet App: 19 pages, 18 components, 7 stores, deployed on GitHub Pages (gaurav2sheth.github.io/paytm-wallet-app)

- Admin Dashboard: 10 pages with Ant Design, AI summarizer, deployed on GitHub Pages (gaurav2sheth.github.io/ppi-wallet-admin)

- Both apps use VITE_API_URL to connect to Render backend in production, with local mock fallback for development

- Transaction sync: Wallet app mutations sync to Render backend via sync.ts, keeping AI agent data current

# 13. Platform & Infrastructure

- Microservices architecture- Event-driven systems- High availability design

**Current Deployment:**

- GitHub Pages: Static frontend hosting for both wallet app and admin dashboard

- Render: Express.js backend with MCP data layer (13 REST endpoints, 35 MCP tools)

Additionally, the platform includes a KYC Expiry Alert System that uses Claude Haiku to automatically detect at-risk users and generate personalised outreach messages, accessible via API and the admin dashboard's KYC Management page.

- GitHub Actions: CI/CD pipeline for frontend builds and deployment

- Render Auto-Deploy: Backend auto-deploys on GitHub push to main

- 4 GitHub repositories: paytm-wallet-app, ppi-wallet-admin, ppi-wallet-api, ppi-wallet-mcp
