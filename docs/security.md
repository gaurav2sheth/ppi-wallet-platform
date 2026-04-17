# Security Model

This document catalogs the security posture of the PPI Wallet Platform — both what exists and what's deliberately out of scope for a reference implementation. Nothing here is marketing; known gaps are listed explicitly.

> **Reference implementation, not production.** The admin password is `admin123`. Do not deploy this as-is to handle real money. See [Threat Model](#threat-model) for what's acceptable demo-risk vs what needs hardening.

## Table of Contents

- [1. Scope Statement](#1-scope-statement)
- [2. Threat Model](#2-threat-model)
- [3. Authentication & Authorization](#3-authentication--authorization)
- [4. Secret Management](#4-secret-management)
- [5. Mock-Mode Guardrails](#5-mock-mode-guardrails)
- [6. API Rate Limiting](#6-api-rate-limiting)
- [7. Content Security Policy (CSP)](#7-content-security-policy-csp)
- [8. PII & Data Handling](#8-pii--data-handling)
- [9. AI-Specific Security](#9-ai-specific-security)
- [10. Dependency Hygiene](#10-dependency-hygiene)
- [11. Disclosure Policy](#11-disclosure-policy)

---

## 1. Scope Statement

**In scope (demo-grade, documented, tested):**
- Load Guard enforcement of RBI balance caps
- Saga compensation on partial transaction failure
- Idempotency key handling on money-moving POSTs
- Ledger integrity (append-only, BigInt paise)
- AI agent prompt injection resistance (30 red-team cases)

**Out of scope (documented gaps — NOT recommendations to ship):**
- Real authentication (demo uses `admin123`, no token rotation)
- Server-side rate limiting
- CSP headers
- WAF / DDoS protection
- Audit log persistence across restarts
- Real-time fraud detection
- End-to-end encryption of PII at rest

---

## 2. Threat Model

Using STRIDE as a frame, evaluated against each asset.

### 2.1 Assets

| Asset | Sensitivity | Location |
|-------|------------|----------|
| Wallet balances | High | Express API in prod; localStorage in demo |
| Ledger entries | High | Same as above |
| KYC state | High | Same |
| PII (name, phone) | Medium | Same |
| Admin credentials | High | In-app constants (demo); env var (prod) |
| Anthropic API key | High | Render env var only |
| Chat session data | Low | In-memory server-side |
| Support tickets | Medium | In-memory server-side |

### 2.2 Threats & Mitigations

| Threat | Asset | Mitigation status |
|--------|-------|-------------------|
| **S**poofing — user impersonates another | Wallet | Demo: walletId in auth store; Prod: needs OTP + session tokens |
| **T**ampering — user edits localStorage to inflate balance | Wallet | Demo: accepted risk, no enforcement. Prod: server is source of truth |
| **R**epudiation — user denies a transaction | Ledger | Idempotency key on every POST; server-side append-only log |
| **I**nformation disclosure — admin sees other admin's data | Admin data | RBAC (6 roles, 12 permissions) in frontend; **backend does NOT enforce** yet — known gap |
| **D**enial of service — flood API with requests | All | **No rate limiting** — known gap |
| **E**levation of privilege — CS agent becomes super admin | RBAC | Client-side guard only in demo; backend RBAC needed for prod |

### 2.3 AI-Specific Threats

| Threat | Mitigation |
|--------|-----------|
| Prompt injection in support chat | 8 injection red-team cases, 100% pass ([kyc-agent.md §6.3.1](kyc-agent.md#631-prompt-injection-8-cases)) |
| Hallucinated balance claims | Context-preferred grounding (ADR-007); 4-layer defense ([kyc-agent.md §6.2](kyc-agent.md#62-hallucination-recovery)) |
| Social engineering ("I'm an RBI auditor") | 6 red-team cases; refuses without credentials |
| Token exfiltration via model output | Server never puts secrets in prompts; model has no tool for outbound HTTP |

---

## 3. Authentication & Authorization

### 3.1 Demo auth (current state)

```
Admin login (demo-only):
  username: admin       password: admin123        role: Super Admin
  username: business    password: admin123        role: Business Admin
  username: support     password: admin123        role: CS Agent
```

Wallet app auth is just a walletId stored in `localStorage` — no password at all in demo.

### 3.2 What's missing for production

- [ ] Real password hashing (bcrypt / argon2)
- [ ] JWT with short-lived access tokens + refresh rotation
- [ ] MFA for admin accounts (TOTP or SMS)
- [ ] Device binding / session fingerprinting
- [ ] Forced password rotation for admin (90-day)
- [ ] Backend RBAC enforcement (currently frontend-only)
- [ ] Audit log for every admin action (login, KYC approval, suspend)

### 3.3 Token rotation

Currently: **there is none.** Notes for production:
- Access token TTL: 15 minutes
- Refresh token TTL: 7 days, single-use, bound to device
- Revoke on: logout, password change, suspicious activity
- Store refresh tokens in HttpOnly cookies, not localStorage

---

## 4. Secret Management

### 4.1 Rules (apply to contributors and CI)

1. **Never commit `ANTHROPIC_API_KEY`** — use `.env` locally, Render env vars in prod
2. **Never commit `.env` files** — `.gitignore` covers `.env` and `.env.*` except `.env.example`
3. **Never commit partner bank credentials** — not even "test" ones
4. **Never commit customer PII** — seed data uses synthetic names only
5. **Never log secrets** — redact API keys in error logs (see `[Chat] Error:` handler)

### 4.2 Current secret surface

| Secret | Location | Rotation policy |
|--------|---------|----------------|
| `ANTHROPIC_API_KEY` | Render env var | Manual; recommend quarterly |
| Admin password | Hardcoded (demo) | N/A — replace for prod |
| Session tokens | Not implemented | N/A |
| Signing keys | Not implemented | N/A |

### 4.3 Pre-commit hook (recommended, not yet installed)

```bash
# .husky/pre-commit (suggested)
npx detect-secrets-hook --baseline .secrets.baseline
```

### 4.4 What to do if a secret leaks

1. Immediately rotate the secret at the provider (Anthropic console → new key)
2. Revoke the leaked key
3. Force-push is NOT sufficient to remove from git history — rewrite history via `git filter-repo` and notify all clones
4. Audit access logs for unauthorized use (Anthropic provides API usage logs)
5. If customer data was exposed, follow breach notification procedures

---

## 5. Mock-Mode Guardrails

This addresses the review point: "what prevents a user in mock mode from seeing a fake ₹10L balance and trying to spend it?"

### 5.1 Current state (honest assessment)

**Nothing server-side prevents it in mock mode** — that's the definition of mock mode. Client mocks read from localStorage, which the user can edit in DevTools.

### 5.2 Protections that exist

1. **Every money-moving operation re-validates server-side** (when server is reachable). Client-side balance checks are UX-only.
2. **Mock-mode fallback is logged** to console: `[API] Backend unavailable — using mock data`
3. **Saga DLQ path** catches impossibly large transfers (e.g., ₹10L when cap is ₹1L) at the server validate step.
4. **Load Guard is applied client-side** in mock mode, so tampering with main balance doesn't unlock tampered sub-wallet totals.

### 5.3 Proposed additions (not yet shipped)

- [ ] **Demo-mode banner** — persistent amber banner "You're viewing demo data" when any API call has fallen back to mock mode in the session
- [ ] **Read-only toggle in mock mode** — developer-facing flag to disable all writes in demo builds
- [ ] **Max-mock-balance cap** — mock layer refuses to return balances > ₹1,00,000 even if localStorage is tampered
- [ ] **Mock-mode fingerprint in ledger** — every mock-mode transaction gets `source: "CLIENT_MOCK"` so a future migration can identify and exclude them

### 5.4 Why it's acceptable for a reference implementation

- No real money is at risk
- No real user data is at risk (all PII is synthetic)
- No external systems (banks, UPI, regulators) accept transactions from this platform
- Clearly labeled as demo in the README and CLAUDE.md

But the proposed additions should ship before any "pilot" or "partner preview" where a non-developer might encounter it.

---

## 6. API Rate Limiting

### 6.1 Current state

**No rate limiting exists.** The Express API on Render accepts unlimited requests from any origin in the CORS allowlist.

### 6.2 Attack surface

| Endpoint | Cost per hit | Risk if abused |
|----------|-------------|----------------|
| `POST /api/support/chat` | ~$0.003 Anthropic | $450/month at 5 req/s |
| `POST /api/chat` | ~$0.01 Anthropic | $1,500/month at 5 req/s |
| `POST /api/kyc-agent/run` | ~$0.01 per 4 users | Runs full loop each time |
| `POST /api/wallet/validate-load` | Low | Sub-wallet read-heavy |
| `GET /api/support/tickets` | Low | In-memory read |

The Anthropic endpoints are the main economic risk — a bad actor could exhaust a month's budget in hours.

### 6.3 Recommended controls (not yet shipped)

```javascript
// suggested addition to api-server/server.js
import rateLimit from 'express-rate-limit';

const chatLimiter = rateLimit({
  windowMs: 60 * 1000,        // 1 minute
  max: 20,                     // 20 requests per minute per IP
  message: { error: 'Rate limited, try again in a minute' },
  standardHeaders: true,
});

app.post('/api/support/chat', chatLimiter, async (req, res) => { /* ... */ });
app.post('/api/chat', chatLimiter, async (req, res) => { /* ... */ });
```

Additional recommended:
- Per-`user_id` limit (vs per-IP) once auth exists
- Circuit breaker: if 10 chat requests fail in 60s, short-circuit and serve template fallback
- Global daily AI spend cap (see [kyc-agent.md §7.2.4](kyc-agent.md#724-budget-controls))

### 6.4 Why not shipped yet

Reference implementation with low daily traffic. Shipping rate limiting without real auth means IP-based limits, which are easy to bypass. Blocked behind "add real auth first" priority.

---

## 7. Content Security Policy (CSP)

### 7.1 Current state

**No CSP headers** are set on the GitHub Pages builds. The meta CSP tag is absent from `index.html` in both frontends.

### 7.2 Risk

XSS would be possible if malicious content were rendered unescaped. React's default JSX escaping mitigates most, but:
- `dangerouslySetInnerHTML` usage is zero in the codebase (confirmed 2026-04-17)
- Markdown rendering (if added) would need careful handling
- Third-party embeds (YouTube, etc.) could be exploited if added

### 7.3 Recommended CSP (not yet shipped)

```html
<!-- suggested addition to paytm-wallet-app/index.html and admin-dashboard/index.html -->
<meta http-equiv="Content-Security-Policy" content="
  default-src 'self';
  script-src 'self';
  style-src 'self' 'unsafe-inline';
  connect-src 'self' https://ppi-wallet-api.onrender.com;
  img-src 'self' data: https:;
  font-src 'self' data:;
  base-uri 'self';
  form-action 'self';
">
```

Caveats for Tailwind + Ant Design:
- `style-src 'unsafe-inline'` needed for Ant Design's runtime CSS-in-JS — ideally replace with nonce-based CSP
- `script-src 'self'` may break Vite dev mode; apply only to production build

### 7.4 Additional headers (GitHub Pages limits)

GitHub Pages doesn't let us set arbitrary response headers. For production, move to a platform that does (Cloudflare Pages, Vercel, Netlify):

- `Strict-Transport-Security: max-age=31536000; includeSubDomains`
- `X-Content-Type-Options: nosniff`
- `X-Frame-Options: DENY`
- `Referrer-Policy: strict-origin-when-cross-origin`
- `Permissions-Policy: geolocation=(), camera=(), microphone=()`

---

## 8. PII & Data Handling

### 8.1 PII inventory

| Field | Source | Storage | Retention |
|-------|--------|---------|-----------|
| Name | Seed / user input | localStorage + server mock | Session |
| Phone | Seed | localStorage + server mock | Session |
| KYC state | Derived | server mock | Session |
| Aadhaar / PAN | Not collected | — | — |
| Bank account | Not collected | — | — |
| IP address | HTTP request | Render logs (30-day) | Render default |
| User agent | HTTP request | Render logs (30-day) | Render default |

The demo intentionally does NOT collect Aadhaar, PAN, or bank details. Production would need:
- Tokenization at collection point
- Encryption at rest (AES-256)
- 10-year retention for KYC artefacts (RBI rule)
- Right-to-erasure handling (where legal under IT Act 2000)

### 8.2 Log redaction

Current logs include:
- `user_id` — low sensitivity, not PII by itself
- Balance amounts — sensitive but needed for debugging
- Claude API responses — potentially echoes user message, which could include PII if user types it

Mitigations in place:
- No credit card / bank details ever logged
- API key errors are logged as "Could not resolve authentication method" — not the key itself

Proposed additions:
- Phone number redaction in chat logs (`+91-9XXXXXX123` → `+91-9XXX***123`)
- User message truncation at 500 chars in logs

---

## 9. AI-Specific Security

See [kyc-agent.md §6.3](kyc-agent.md#63-adversarial-robustness-tests) for the 30-case red-team suite. Summary:

| Category | Cases | Pass rate |
|----------|-------|-----------|
| Prompt injection | 8 | 8/8 (100%) |
| Data poisoning | 6 | 6/6 (100%) |
| Social engineering | 6 | 6/6 (100%) |
| Ambiguity | 5 | 4/5 (80% — emoji case fixed) |
| Stress / robustness | 5 | 5/5 (100%) |
| **Total** | **30** | **29/30 (96.7%)** |

### 9.1 AI-specific privileges (intentionally tight)

The Claude-backed agents CAN:
- Read wallet/transaction/KYC data
- Draft SMS / notification text
- Create escalations and tickets
- Suggest actions to users

The agents CANNOT (by design — no tool available):
- Initiate transfers or payments
- Modify KYC state
- Grant reward payouts
- Access raw KYC documents
- Delete records
- Execute arbitrary code
- Make outbound HTTP calls

This follows least-privilege: agents advise and request, humans approve.

---

## 10. Dependency Hygiene

### 10.1 Tooling

- `npm audit` should run in CI — not yet automated (manual review)
- Dependabot alerts enabled on all 5 repos
- No critical alerts open as of 2026-04-17

### 10.2 Third-party dependency trust

High-trust:
- React, Vite, Ant Design, Tailwind — well-maintained major projects
- `@anthropic-ai/sdk` — first-party

Medium-trust:
- `gh-pages`, `vitest`, testing utilities — stable but lower review volume

Review policy: any new dependency must be:
- In top 80% of npm download volume for its category, OR
- Reviewed by 2 contributors

---

## 11. Disclosure Policy

This is a reference implementation — we don't run a bug bounty. If you find a security issue:

1. **Do not open a public issue** describing the vulnerability
2. Email the maintainer (contact listed in the repo owner's GitHub profile)
3. Allow 14 days for response before any disclosure

Out of scope:
- Findings that exploit the documented "demo-mode risks" (mock mode tampering, admin123 password) — these are known
- Findings that require physical access to the user's device
- Findings in sibling repos' demo infrastructure (gh-pages, Render free tier)
