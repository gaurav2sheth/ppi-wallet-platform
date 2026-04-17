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

## Architecture Decisions

See [`docs/adr/`](docs/adr/) for the full set of architecture decision records. Each ADR captures context, decision, consequences (positive and negative), alternatives considered, and references.

Currently accepted:

| # | Title |
|---|-------|
| [001](docs/adr/ADR-001-three-tier-api-fallback.md) | Three-tier API fallback |
| [002](docs/adr/ADR-002-saga-pattern-transactions.md) | Saga pattern for transactions |
| [003](docs/adr/ADR-003-subwallets-in-rbi-cap.md) | Sub-wallet balances count toward the RBI cap |
| [004](docs/adr/ADR-004-fastag-security-deposit.md) | FASTag as security deposit model |
| [005](docs/adr/ADR-005-ncmc-isolated-balance.md) | NCMC isolated balance (no cascade) |
| [006](docs/adr/ADR-006-localstorage-versioning.md) | localStorage key versioning |
| [007](docs/adr/ADR-007-ai-context-sync.md) | AI context sync (client provides ground truth) |
| [008](docs/adr/ADR-008-dual-model-claude-strategy.md) | Dual-model strategy (Sonnet + Haiku) |
| [009](docs/adr/ADR-009-five-shot-learning.md) | 5-shot in-context learning for KYC agent |
| [010](docs/adr/ADR-010-in-memory-escalation-store.md) | In-memory shared escalation store |
| [011](docs/adr/ADR-011-monetary-arithmetic-in-paise-bigint.md) | Monetary arithmetic in paise with BigInt |

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

**Models used (current pins):**
- `claude-sonnet-4-20250514` — Intent classification, KYC decision reasoning
- `claude-haiku-4-5-20251001` — SMS/notification drafting, response generation, ops summaries

**Model-per-task rationale:** Sonnet for reasoning (classification accuracy matters), Haiku for generation (fast + cheap, strong at constrained writing). Full analysis in [ADR-008](docs/adr/ADR-008-dual-model-claude-strategy.md) and [`kyc-agent.md` §7.3](docs/kyc-agent.md#73-model-selection-rationale).

**Version upgrade policy:** Quarterly re-evaluation on the 200-case golden set. Promote if +3% accuracy OR -20% cost with equal quality. Sonnet 4.6 and Opus 4.x have been spot-tested; not yet promoted — see ADR-008 for current decisions.

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
- `docs/scope-and-limitations.md` — What's real vs mocked, anti-patterns, production gap list (the "what this is and isn't" doc)
- `docs/ai-agents.md` — AI agents architecture, API endpoints, data flows, cost estimates
- `docs/kyc-agent.md` — KYC agent deep dive: few-shot patterns, context ordering, 200-case golden dataset, LLM-as-judge, failure modes, adversarial tests, product strategy
- `docs/adr/` — 10 Architecture Decision Records (3-tier fallback, saga pattern, NCMC isolation, AI context sync, dual-model strategy, 5-shot learning, etc.)
- `docs/diagrams.md` — Mermaid sequence diagrams (Load Guard, cascade spend, FASTag, NCMC, agents, saga lifecycle)
- `docs/security.md` — Threat model, mock-mode guardrails, secret management, rate limiting gaps
- `docs/edge-cases.md` — 68 edge-case specs with test-fixture status (Load Guard racing, cap math, KYC downgrade, refund flows, timezone bug)
- `CHANGELOG.md` — Change history in Keep-a-Changelog format

See `.claude/rules/` for auto-loaded context:
- `rules/compliance.md` — RBI limits, KYC states, Load Guard rules
- `rules/sub-wallets.md` — NCMC, FASTag, Food, Fuel, Gift mechanics
- `rules/data-models.md` — TypeScript interfaces, storage keys
