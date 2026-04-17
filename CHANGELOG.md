# Changelog

All notable changes to the PPI Wallet Platform are documented here.
Format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).
Versions are implicit (pre-1.0 rolling releases); dates are in IST.

## [Unreleased]

### Added
- `CHANGELOG.md` at platform root
- `docs/adr/` — 10 Architecture Decision Records promoted from CLAUDE.md
- `docs/diagrams.md` — Mermaid sequence diagrams (Load Guard, cascade-spend, agent flows)
- `docs/security.md` — Threat model, secret management, mock-mode guardrails, rate limiting gaps
- `docs/edge-cases.md` — 40+ edge-case specs (Load Guard racing, cap math, KYC downgrade, refund flows)
- README "Where's the code?" callout linking to all 4 sibling repos

### Changed
- README framing softened from "production-grade" → "reference implementation / platform blueprint"
- README License section made consistent with the public repo
- README now lists the 49 MCP tools by category (6 categories, with examples)

### Known Gaps (not blockers; tracked)
- Mock-mode balances are editable in localStorage and not explicitly labeled as demo data
- Admin auth is hardcoded `admin123` — demo-only, no token rotation
- No server-side rate limiting on the Render API
- No Content-Security-Policy headers on the GitHub Pages builds

---

## 2026-04-17 — AI Product Strategy & Evaluation

### Added
- `docs/kyc-agent.md` (1,262 lines) — KYC agent deep dive
  - Few-shot learning patterns (5-shot exemplar library + diversity matrix)
  - Context ordering optimization (primacy/recency, token budget, attention probes)
  - 200-case golden dataset across 7 categories
  - LLM-as-judge framework (5-dim rubric, 100 test cases, two-judge consensus)
  - Cost-accuracy tradeoffs table, monthly projections
  - Failure mode taxonomy (model/tool/orchestration/data)
  - 4-layer hallucination defense
  - 30 adversarial red-team cases
  - Model limitations, pricing strategy, scaling thresholds

---

## 2026-04-16 — NCMC Direct Load & AI Context Sync

### Added
- `mockDirectLoadNcmc(amountPaise, paymentSource)` — loads NCMC from UPI/DC/NB without touching main wallet
- Payment method selector pills on NCMC and FASTag Add Money panels
- Support agent accepts `context` parameter and uses client-provided balance/transactions
- `clientContext:balance`, `clientContext:transactions` tool logs

### Changed
- NCMC Add Money panel now shows: Main Wallet / UPI / Debit Card / Net Banking
- FASTag Add Money panel now shows: UPI / Debit Card / Net Banking (all route to main wallet top-up)
- Support agent investigate step prefers client context over server mock data
- Wallet app AiChatCard always sends real balance + recent transactions with each request
- Admin AiChatCard sends user context when query contains `user_` pattern

### Fixed
- AI chat responses now match exactly what the user sees in Balance and History sections (prior drift between server mock and client localStorage)

---

## 2026-04-15 — Customer Support Agent

### Added
- `mcp/agents/customer-support-agent.js` — Dynamic 5-step support agent
  (UNDERSTAND → INVESTIGATE → RESOLVE → RESPOND → ESCALATE)
- `mcp/agents/support-ticket-manager.js` — In-memory ticket store with SLA tracking
- 5 new MCP tools (total now 49): `get_support_tickets`, `create_support_ticket`, `resolve_support_ticket`, `get_reward_history`, `get_load_guard_log`
- 7 new API routes: `/api/support/chat`, `/api/support/tickets`, `/api/support/sessions/:id`, `/api/support/analytics`
- Admin dashboard "Support" tab (7th module): live metrics, open tickets table, escalations, analytics
- Wallet app "My Tickets" page at `/support/tickets`
- Escalation manager now shared between KYC and Support agents via `source` field

### Changed
- "Ask AI" components upgraded to support agent endpoint with session management
- Wallet app chat now shows intent badges, suggested-action pills, ticket banners
- Admin chat now shows metadata tags (intent, tools, response time, confidence)

---

## 2026-04-14 — KYC Upgrade Agent

### Added
- `mcp/agents/kyc-upgrade-agent.js` — Autonomous 7-step agent
- `mcp/agents/escalation-manager.js` — Shared escalation store
- `mcp/services/scheduler.js` — 3 cron jobs (KYC agent, alerts, follow-ups)
- 5 new MCP tools (total now 44): `send_kyc_notification`, `check_kyc_upgrade_status`, `grant_upgrade_reward`, `get_agent_escalations`, `resolve_escalation`
- 59 agent unit tests (MCP)
- 5 new test users (user_196–user_200) with varied KYC expiry profiles
- Admin dashboard KYC Agent Panel (status, run button, decision log, escalations, notifications, audit trail)

### Fixed
- Priority sort bug: `||` → `??` so P1_CRITICAL (value 0) sorts first not last

---

## 2026-04-13 — Documentation Consolidation

### Added
- `README.md` at platform root with architecture overview and quick start
- All 12 `.docx` design docs converted to `.md` in `docs/`
- `docs/ai-agents.md` — comprehensive agent architecture reference

---

## 2026-04-12 — Testing Infrastructure

### Added
- Vitest + React Testing Library in both frontends
- 94 tests in wallet app (utilities, mock layer, stores, components, pages)
- 111 tests in admin dashboard (RBAC, auth guards, stores, tables)
- 59 tests in MCP agents module (decision matrix, escalation, notifications)
- **Total: 264 tests** across 3 projects

---

## 2026-04-11 — Platform Baseline

### Added
- Consumer wallet app (React 19 + Vite 8 + Tailwind) — 21 pages
- Admin dashboard (React 19 + Ant Design 5) — 6 modules initially
- MCP server with 39 tools across 6 categories
- Express API server with Claude integration
- RBI compliance: Load Guard (3 rules), KYC state machine, balance caps
- Sub-wallet system: Food, NCMC Transit, FASTag, Gift, Fuel
- Cascade spend logic with MCC keyword matching
- Saga pattern for transactions with compensation
- 3-tier API fallback (Vite middleware → Express → client-side mock)
- Zustand state management with localStorage persistence
- GitHub Pages deployment for both frontends
- Render deployment for API server

---

## Versioning Policy

This platform is pre-1.0 and uses rolling releases. When/if we cut a 1.0:
- **MAJOR** bumps for RBI regulatory shape changes (e.g., new wallet cap policy)
- **MINOR** bumps for new features (e.g., new sub-wallet type, new agent)
- **PATCH** bumps for bug fixes, doc updates, dependency bumps
