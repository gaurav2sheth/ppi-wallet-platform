# PPI Wallet Platform

Multi-repo workspace for an RBI-regulated Prepaid Payment Instrument (PPI) Wallet.

## Ecosystem

| Repo | Tech | Port | Purpose |
|------|------|------|---------|
| `paytm-wallet-app/` | React 19 + Vite + Tailwind | 5173 | Consumer wallet UI (mobile-first) |
| `admin-dashboard/` | React 19 + Vite + Ant Design | 5174 | Admin operations dashboard (desktop) |
| `mcp/` | Node.js + Zod | stdio | 49 Claude AI tools via MCP protocol |
| `api-server/` | Express.js + Claude API | 3001 | REST API (chat, KYC alerts, load guard, sub-wallets, AI agents) |
| `ppi-wallet-api-deploy/` | Express.js | Render | Production deploy of api-server |

## Quick Start

```bash
# Wallet app (works standalone with built-in mock data)
cd paytm-wallet-app && npm install && npm run dev

# Admin dashboard
cd admin-dashboard && npm install && npm run dev

# API server (requires ANTHROPIC_API_KEY)
cd api-server && npm install && npm run dev
```

## Build & Deploy

```bash
# Wallet app → GitHub Pages
cd paytm-wallet-app
/usr/local/bin/node ./node_modules/.bin/tsc --noEmit   # type-check
/usr/local/bin/node ./node_modules/.bin/vite build      # build
/usr/local/bin/node ./node_modules/.bin/gh-pages -d dist # deploy

# Admin dashboard → GitHub Pages
cd admin-dashboard
/usr/local/bin/node ./node_modules/.bin/vite build
/usr/local/bin/node ./node_modules/.bin/gh-pages -d dist

# API → push to ppi-wallet-api-deploy, auto-deploys on Render
```

## Live URLs

| Target | URL |
|--------|-----|
| Wallet App | https://gaurav2sheth.github.io/ppi-wallet-app |
| Admin Dashboard | https://gaurav2sheth.github.io/ppi-wallet-admin |
| API | https://ppi-wallet-api.onrender.com/health |

## Conventions (All Repos)

- All monetary values: **integers in paise** (1 INR = 100 paise). Never use floats for money.
- Currency display: `₹X,XX,XXX.XX` using `en-IN` locale with 2 decimal places.
- UUIDs for all entity IDs. Idempotency keys on every transaction POST.
- ISO 8601 UTC timestamps everywhere.
- Design system: Paytm PODS — Navy #002E6E, Cyan #00B9F1, Green #12B76A.
- Wallet app uses HashRouter (required for GitHub Pages).
- Both frontends use Zustand for state with localStorage persistence.
- Node path on this machine: `/usr/local/bin/node` (no nvm).

## Git Repos (5 Repos)

This platform repo contains documentation only. Clone the code repos alongside it:

```bash
# 1. Clone this platform repo (docs, specs, Claude context)
git clone https://github.com/gaurav2sheth/ppi-wallet-platform.git PPI_Wallet
cd PPI_Wallet

# 2. Clone the code repos inside it
git clone https://github.com/gaurav2sheth/ppi-wallet-app.git paytm-wallet-app
git clone https://github.com/gaurav2sheth/ppi-wallet-admin-dashboard.git admin-dashboard
git clone https://github.com/gaurav2sheth/ppi-wallet-mcp.git mcp
git clone https://github.com/gaurav2sheth/ppi-wallet-api-deploy.git ppi-wallet-api-deploy
```

| Directory | GitHub Repo |
|-----------|-------------|
| `paytm-wallet-app/` | [ppi-wallet-app](https://github.com/gaurav2sheth/ppi-wallet-app) |
| `admin-dashboard/` | [ppi-wallet-admin-dashboard](https://github.com/gaurav2sheth/ppi-wallet-admin-dashboard) |
| `mcp/` | [ppi-wallet-mcp](https://github.com/gaurav2sheth/ppi-wallet-mcp) |
| `api-server/` | (local only, deployed via ppi-wallet-api-deploy) |
| `ppi-wallet-api-deploy/` | [ppi-wallet-api-deploy](https://github.com/gaurav2sheth/ppi-wallet-api-deploy) |

## Key Architecture Decisions

- **3-tier API fallback**: Vite middleware → Express API → client-side mock. The app always works, even offline.
- **Saga pattern** for transactions: Amount → PIN → API call → Result screen (success/retry).
- **Sub-wallet balances included in RBI cap**: ₹1L limit = main wallet + all sub-wallet balances combined.
- **FASTag = security deposit model**: Tolls deduct from main wallet; deposit is last resort.
- **NCMC = isolated balance**: ₹3K cap, transit-only, never cascades to main wallet. Supports direct load from UPI/DC/NB (bypasses main wallet).
- **AI context sync**: Support agent receives actual app balance + transactions from the frontend (not server mock data), ensuring AI responses match what the user sees.
- **localStorage versioning**: Sub-wallet key is `__mock_sub_wallets_v2` to force re-seed on schema changes.

## AI Agents

Three autonomous AI agents power the platform's operations:

| Agent | Architecture | File | Trigger |
|-------|-------------|------|---------|
| KYC Upgrade Agent | Fixed 7-step loop (PERCEIVE→REASON→PLAN→ACT→OBSERVE→FOLLOW-UP→SUMMARY) | `mcp/agents/kyc-upgrade-agent.js` | Cron 8 AM IST / manual |
| Customer Support Agent | Dynamic 5-step (UNDERSTAND→INVESTIGATE→RESOLVE→RESPOND→ESCALATE) | `mcp/agents/customer-support-agent.js` | User chat message |
| KYC Alert Service | 4-step pipeline (FETCH→GENERATE→SIMULATE→SUMMARY) | `mcp/services/kyc-alert-service.js` | Cron 9 AM IST / manual |

Supporting modules:
- `mcp/agents/escalation-manager.js` — Shared in-memory escalation store (used by both agents)
- `mcp/agents/support-ticket-manager.js` — Ticket store with SLA tracking (HIGH=1hr, MEDIUM=2hr, LOW=4hr)
- `mcp/services/scheduler.js` — Cron orchestration for all 3 jobs

**Models used:**
- `claude-sonnet-4-20250514` — Intent classification, KYC decision reasoning
- `claude-haiku-4-5-20251001` — SMS/notification drafting, response generation, ops summaries

All Claude API calls have keyword/template fallback. Agents work at $0 cost without an API key.

## Reference Documents

See `docs/` for detailed specs (all converted from original .docx to markdown):
- `docs/architecture.md` — Full system architecture
- `docs/product-requirements.md` — PRD v1.1 with RBI compliance
- `docs/product-requirements-v1.md` — PRD v1.0 original requirements
- `docs/product-document.md` — Product overview and vision
- `docs/engineering-brief.md` — API spec, DB schema, escrow engine
- `docs/epic-breakdown.md` — Sprint-level epic breakdown
- `docs/admin-dashboard-prd.md` — Admin dashboard requirements
- `docs/compliance-gap-analysis-v1.md` — Initial compliance gaps
- `docs/compliance-gap-analysis-v2.md` — Updated gaps with remediation
- `docs/competitive-benchmark.md` — PPSL vs competitors
- `docs/partner-bank-evaluation.md` — Partner bank evaluation matrix
- `docs/claude-code-pm-guide.md` — Claude Code PM guide
- `docs/ai-agents.md` — AI agents architecture, API endpoints, data flows, cost estimates

See `.claude/rules/` for auto-loaded context:
- `rules/compliance.md` — RBI limits, KYC states, Load Guard rules
- `rules/sub-wallets.md` — NCMC, FASTag, Food, Fuel, Gift mechanics
- `rules/data-models.md` — TypeScript interfaces, storage keys
