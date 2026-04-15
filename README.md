# PPI Wallet Platform

A production-grade **Prepaid Payment Instrument (PPI) Wallet** platform built for RBI compliance, featuring a consumer mobile wallet, an admin operations dashboard, an AI-powered API server, 49 Claude AI tools via MCP, and 3 autonomous AI agents for KYC compliance and customer support.

## Live Demo

| App | URL |
|-----|-----|
| Consumer Wallet | [gaurav2sheth.github.io/ppi-wallet-app](https://gaurav2sheth.github.io/ppi-wallet-app) |
| Admin Dashboard | [gaurav2sheth.github.io/ppi-wallet-admin](https://gaurav2sheth.github.io/ppi-wallet-admin) |
| API Health | [ppi-wallet-api.onrender.com/health](https://ppi-wallet-api.onrender.com/health) |

## Architecture

```
┌─────────────────────┐   ┌─────────────────────┐
│  Consumer Wallet UI │   │  Admin Dashboard UI  │
│  React 19 + Vite 8  │   │  React 19 + Ant Design│
│  Tailwind + Zustand  │   │  ECharts + Zustand   │
│  Port 5173           │   │  Port 5174           │
└────────┬────────────┘   └────────┬─────────────┘
         │                         │
         └──────────┬──────────────┘
                    ▼
         ┌─────────────────────┐
         │   Express API       │
         │   Claude AI + MCP   │
         │   Port 3001         │
         └──────────┬──────────┘
                    │
         ┌──────────┴──────────┐
         │                     │
┌────────▼────────┐  ┌────────▼────────┐
│  MCP Server     │  │  AI Agents      │
│  49 Tools (Zod) │  │  KYC + Support  │
│  stdio transport│  │  Cron Scheduler │
└─────────────────┘  └─────────────────┘
```

**3-tier API fallback** — The wallet app always works, even offline:
1. Vite dev middleware (proxied API)
2. Express API server (production)
3. Client-side mock data (fallback)

## Repositories

Clone this platform repo, then clone the code repos inside it:

```bash
# 1. Platform repo (docs, specs, Claude AI context)
git clone https://github.com/gaurav2sheth/ppi-wallet-platform.git PPI_Wallet
cd PPI_Wallet

# 2. Code repos
git clone https://github.com/gaurav2sheth/ppi-wallet-app.git paytm-wallet-app
git clone https://github.com/gaurav2sheth/ppi-wallet-admin-dashboard.git admin-dashboard
git clone https://github.com/gaurav2sheth/ppi-wallet-mcp.git mcp
git clone https://github.com/gaurav2sheth/ppi-wallet-api-deploy.git ppi-wallet-api-deploy
```

| Directory | Repo | Tech | Purpose |
|-----------|------|------|---------|
| `paytm-wallet-app/` | [ppi-wallet-app](https://github.com/gaurav2sheth/ppi-wallet-app) | React 19, Vite 8, Tailwind, Zustand | Consumer wallet (mobile-first) |
| `admin-dashboard/` | [ppi-wallet-admin-dashboard](https://github.com/gaurav2sheth/ppi-wallet-admin-dashboard) | React 19, Vite 6, Ant Design 5, ECharts | Admin dashboard (desktop) |
| `mcp/` | [ppi-wallet-mcp](https://github.com/gaurav2sheth/ppi-wallet-mcp) | Node.js, Zod | 49 Claude AI tools + 3 AI agents via MCP |
| `api-server/` | — | Express.js, Claude API | REST API (local dev) |
| `ppi-wallet-api-deploy/` | [ppi-wallet-api-deploy](https://github.com/gaurav2sheth/ppi-wallet-api-deploy) | Express.js | Production API on Render |

## Quick Start

Each app works standalone — no backend required for development.

```bash
# Consumer wallet (includes built-in mock data)
cd paytm-wallet-app && npm install && npm run dev
# → http://localhost:5173

# Admin dashboard
cd admin-dashboard && npm install && npm run dev
# → http://localhost:5174

# API server (optional, requires ANTHROPIC_API_KEY)
cd api-server && npm install && npm run dev
# → http://localhost:3001
```

### Admin Dashboard Credentials

| Role | Username | Password |
|------|----------|----------|
| Super Admin | admin | admin123 |
| Business Admin | business | admin123 |
| CS Agent | support | admin123 |

## Features

### Consumer Wallet App (21 pages)

- **Wallet Balance** — Main balance with collapsible sub-wallet strip
- **Add Money** — UPI, debit card, net banking with RBI Load Guard validation
- **Merchant Pay** — QR scan, category-aware cascade spend across sub-wallets
- **P2P Transfer** — Send to contacts (Full-KYC only, ₹1L/month cap)
- **Bank Transfer** — Wallet-to-bank withdrawal
- **Bill Payment** — Utilities, recharge, subscriptions
- **Passbook** — Full transaction history with filters and search
- **Spend Analytics** — Category breakdown, trends, comparisons
- **Budget Manager** — Set category limits with alerts
- **Rewards** — Scratch cards + cashback
- **KYC Verification** — Min-KYC → Full-KYC upgrade flow
- **Notifications** — Real-time alerts for transactions, KYC, limits
- **AI Support Chat** — Claude-powered support agent with intent classification, dynamic tool selection, session management, and auto-escalation
- **My Support Tickets** — View ticket status, SLA deadlines, resolution notes

### Admin Dashboard (7 modules)

- **Dashboard** — KPI cards, transaction trends (ECharts), AI-powered alerts
- **User Management** — Search, filter, suspend users, view detailed profiles
- **Transaction Monitoring** — Real-time feed, status filtering, dispute management
- **KYC Management** — Verification queue, approve/reject with audit trail, KYC Upgrade Agent panel
- **Benefits Management** — Sub-wallet bulk loading, employer benefit administration
- **Support Operations** — Live support agent metrics, open tickets with SLA tracking, escalations, analytics
- **Settings** — RBAC role management (6 roles, 12 permissions)

## Sub-Wallet System

Five corporate benefit wallet types, each with distinct business rules:

| Type | Cap | Load Source | Eligible Merchants | Special Rules |
|------|-----|-------------|-------------------|---------------|
| 🍱 Food | ₹3,000/month | Employer only | Restaurants, Swiggy, Zomato | Direct deduction |
| 🚇 NCMC Transit | ₹3,000 balance | Self-load from main | Metro, Bus, Local train | **Isolated balance** — no cascade to main wallet |
| 🛣️ FASTag | ₹300/vehicle | Add Money → main wallet | Toll plazas | **Security deposit model** — tolls deduct from main wallet first |
| 🎁 Gift | No cap | Self-load + employer | Universal | Expiry date, shows EXPIRED badge |
| ⛽ Fuel | ₹2,500/month | Employer only | HP, IOCL, BPCL, Shell | Category-restricted |

**Cascade spend logic**: Merchant Pay auto-detects category → checks specific sub-wallet → Gift as fallback → split across sub-wallet + main wallet if needed.

## AI Agents

Three autonomous AI agents handle operational workflows without human intervention:

### KYC Upgrade Agent (Fixed 7-Step Loop)

```
PERCEIVE → REASON → PLAN → ACT → OBSERVE → FOLLOW-UP → SUMMARY
```

- Detects users with KYC expiring within 7 days
- Claude Sonnet decides priority (P1-P4) and intervention strategy per user
- Claude Haiku drafts personalised SMS (<160 chars) and in-app notifications
- Observes user response and auto-escalates unresponsive high-value accounts
- Runs daily at 8 AM IST via cron, with 6-hourly follow-up checks
- **File:** `mcp/agents/kyc-upgrade-agent.js`

### Customer Support Agent (Dynamic Tool Selection)

```
UNDERSTAND → INVESTIGATE → RESOLVE → RESPOND → ESCALATE (if needed)
```

- 13 intent types: balance, payments, transactions, KYC, sub-wallets, rewards, escalation
- Claude Sonnet classifies intent with entity extraction and sentiment detection
- Dynamically selects wallet tools based on intent (e.g., PAYMENT_BLOCKED calls 4 tools)
- Claude Haiku drafts sentiment-matched responses (empathetic for frustrated users)
- Auto-escalates after 2 unresolved same-intent turns or explicit request
- Session management with 30-minute expiry
- SLA-tracked tickets: HIGH (1hr), MEDIUM (2hr), LOW (4hr)
- **File:** `mcp/agents/customer-support-agent.js`

### Shared Infrastructure

| Module | File | Purpose |
|--------|------|---------|
| Escalation Manager | `mcp/agents/escalation-manager.js` | Shared ops escalation store (both agents) |
| Ticket Manager | `mcp/agents/support-ticket-manager.js` | SLA-tracked ticket CRUD |
| Scheduler | `mcp/services/scheduler.js` | Cron orchestration (3 jobs) |
| KYC Alert Service | `mcp/services/kyc-alert-service.js` | Alert message generation pipeline |

### Claude Model Cost

| Operation | Cost |
|-----------|------|
| 1 support chat | ~$0.003 |
| 100 support chats | ~$0.29 |
| 1 KYC agent run (4 users) | ~$0.01 |
| Fallback mode (no API key) | $0.00 |

> Full documentation: [`docs/ai-agents.md`](docs/ai-agents.md)

## RBI Compliance

All regulatory rules are enforced at every transaction:

| Rule | Min-KYC | Full-KYC |
|------|---------|----------|
| Balance cap | ₹10,000 | ₹2,00,000 |
| Monthly load | ₹10,000 | ₹1,00,000 |
| P2P transfer | Prohibited | ₹1,00,000/month |
| Cash withdrawal | Not allowed | ₹2,000/txn, ₹10,000/month |

**Load Guard** validates every Add Money against 3 rules:
1. **BALANCE_CAP** — Main wallet + all sub-wallets ≤ ₹1,00,000
2. **MONTHLY_LOAD** — Calendar month total ≤ ₹2,00,000
3. **MIN_KYC_CAP** — Min-KYC wallets ≤ ₹10,000

## Testing

All projects have comprehensive test suites using Vitest + React Testing Library:

```bash
# Wallet app — 94 tests
cd paytm-wallet-app && npm test

# Admin dashboard — 111 tests
cd admin-dashboard && npm test

# MCP agents — 59 tests
cd mcp && npx vitest run
```

**264 total tests** covering utilities, mock data layer, Zustand stores, components, pages, RBAC, auth guards, and AI agent functions (intent classification, escalation, ticket management, notification lifecycle).

## Build & Deploy

```bash
# Wallet app → GitHub Pages
cd paytm-wallet-app
npx tsc --noEmit && npx vite build && npx gh-pages -d dist

# Admin dashboard → GitHub Pages
cd admin-dashboard
npx tsc -b && npx vite build && npx gh-pages -d dist

# API → push to ppi-wallet-api-deploy, auto-deploys on Render
```

## Documentation

All source `.docx` documents have been converted to markdown and organized in `docs/`:

### Product & Architecture

| File | Description |
|------|-------------|
| `docs/product-document.md` | Product overview and vision |
| `docs/product-requirements-v1.md` | PRD v1.0 — original requirements |
| `docs/product-requirements.md` | PRD v1.1 — expanded with RBI compliance |
| `docs/architecture.md` | Full system architecture |
| `docs/engineering-brief.md` | API spec, DB schema, escrow reconciliation engine |
| `docs/epic-breakdown.md` | Sprint-level epic breakdown |
| `docs/admin-dashboard-prd.md` | Admin dashboard requirements |

### Compliance & Strategy

| File | Description |
|------|-------------|
| `docs/compliance-gap-analysis-v1.md` | Initial RBI/PMLA compliance gap analysis |
| `docs/compliance-gap-analysis-v2.md` | Updated compliance gaps with remediation plans |
| `docs/competitive-benchmark.md` | PPSL vs PhonePe, Amazon Pay, Airtel wallets |
| `docs/partner-bank-evaluation.md` | Yes Bank, IndusInd, RBL, IDFC First evaluation |
| `docs/claude-code-pm-guide.md` | Claude Code PM guide for engineering teams |

### AI Agents

| File | Description |
|------|-------------|
| `docs/ai-agents.md` | Comprehensive agent documentation — architecture, API endpoints, data flows, Claude model usage, cost estimates, fallback strategies, business rules |

### Claude AI Context

| File | Description |
|------|-------------|
| `CLAUDE.md` | Ecosystem overview, conventions, architecture decisions |
| `PROMPT.md` | Comprehensive prompt to recreate the entire platform |
| `.claude/rules/compliance.md` | KYC tiers, Load Guard, AML thresholds |
| `.claude/rules/sub-wallets.md` | All 5 wallet types, cascade spend logic |
| `.claude/rules/data-models.md` | TypeScript interfaces, localStorage keys |
| `.claude/settings.json` | Shared Claude Code team permissions |

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Frontend (Wallet) | React 19, TypeScript, Vite 8, Tailwind CSS 4, Zustand 5 |
| Frontend (Admin) | React 19, TypeScript, Vite 6, Ant Design 5, ECharts 5, Zustand 5 |
| Backend | Express.js, Anthropic Claude API |
| AI Tools | MCP Protocol, Zod validation, 49 tools across 6 categories |
| AI Agents | KYC Upgrade Agent (Sonnet+Haiku), Customer Support Agent (Sonnet+Haiku), KYC Alert Service (Haiku) |
| Testing | Vitest, React Testing Library, jsdom |
| Deployment | GitHub Pages (frontends), Render (API) |
| State | Zustand with localStorage persistence |
| Routing | HashRouter (GitHub Pages compatible) |

## Design System

Paytm PODS — consistent across both frontends:

| Token | Value | Usage |
|-------|-------|-------|
| Navy | `#002E6E` | Primary headers, nav, buttons |
| Cyan | `#00B9F1` | Accents, links, highlights |
| Green | `#12B76A` | Success states, positive amounts |

All monetary values stored as **integers in paise** (1 INR = 100 paise). Displayed as `₹X,XX,XXX.XX` using `en-IN` locale.

## License

Private project — not open source.
