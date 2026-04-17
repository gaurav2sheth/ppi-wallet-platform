# ADR-001: Three-tier API fallback

**Status:** Accepted
**Date:** 2026-04-11
**Deciders:** Platform team

## Context

The wallet app is a reference implementation that must work:
- **In production** — talking to a real backend
- **During local dev** — without requiring every developer to boot the API
- **In the GitHub Pages demo** — hosted as a static site with no backend
- **During backend outages** — gracefully, not with a wall of 500s

A single API call path fails at least one of these. We need layered fallback.

## Decision

Adopt a **3-tier request path** in every API wrapper:

1. **Tier 1 — Vite dev middleware**: in `npm run dev`, requests to `/api/*` are intercepted by Vite plugins and answered inline.
2. **Tier 2 — Express API server**: in production, Vite is absent, so requests pass through to the deployed Render API.
3. **Tier 3 — Client-side mock**: if both tiers above fail (timeout / network error / 5xx), the client falls back to `src/api/mock.ts` backed by `localStorage`.

Each API wrapper (`walletApi`, `transactionsApi`, etc.) catches errors from Tier 1+2 and transparently calls the Tier 3 mock.

## Consequences

### Positive
- GitHub Pages demo is fully interactive without a backend
- Local dev works with zero infra setup
- Render outages degrade to mock mode rather than a broken app
- Same codebase powers dev / prod / demo

### Negative
- **Silent mode drift** — users in mock mode can see `localStorage`-backed balances and may not realize the data is local. Addressed (partially) by a future "demo-mode banner" — not yet shipped.
- **No write guarantees** — mock-mode "transactions" are lost on `localStorage.clear()`. This is expected for a demo but could confuse real users if mode silently flips.
- **Increased bundle size** — mock data + mock handlers ship to every user (~45KB gz).
- **Test surface 2x** — every flow has a real-API path and a mock-API path; both need tests.

### Mitigations added in 2026-04-16
- Mock mode is now logged to console on each fallback
- Support agent explicitly uses client context (ADR-007) so AI answers match what the user sees in mock mode

### Open risks
- No explicit "read-only in mock mode" guard — a determined user could edit localStorage to see a fake ₹10L balance. See [`docs/security.md`](../security.md#mock-mode-guardrails).

## Alternatives considered

| Alternative | Rejected because |
|------------|------------------|
| Require backend always | GitHub Pages demo impossible |
| Mock-only demo build | Two build pipelines to maintain |
| Service worker proxy | Adds complexity; breaks HMR in dev |
| PouchDB / IndexedDB | More infrastructure for marginal benefit over localStorage |
