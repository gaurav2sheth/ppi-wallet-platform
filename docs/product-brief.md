# Product Brief — PPI Wallet Reference Implementation

**One-line:** An RBI-compliant Prepaid Payment Instrument (PPI) wallet reference implementation, built as an AI-PM capability artefact, demonstrating that a non-engineering PM can ship a non-trivial regulatory-aware product end-to-end using Claude Code + structured documentation.

## Persona

**Primary: "Kavita Thakur" — 32, salaried, tier-2 metro, active digital-payments user.**

- Has a Paytm / PhonePe / GPay habit (daily transactions: 4-8)
- Full-KYC'd via Aadhaar eKYC at some point in the last 24 months
- Carries ₹5,000-₹50,000 in wallet balance as working capital + benefit loads
- Employer loads Food + Fuel + occasional Gift benefits (~₹6,000/month combined)
- Uses NCMC monthly for metro + local bus
- FASTag on personal vehicle
- Low tolerance for transaction failure at the point of purchase (metro gate, Swiggy checkout)
- High trust in the brand's regulatory compliance — expects balance to be available and correctly reported

**Secondary: "Ravi Kulkarni" — PPSL ops admin, 28, handles 200-500 KYC upgrade requests/month, fraud investigations.**

- Needs a dashboard that surfaces anomalies without paging through 10,000 users
- Escalations come via the KYC Upgrade Agent (autonomously flagged) + Customer Support Agent (user-initiated)
- SLA discipline: HIGH tickets in 1hr, MEDIUM in 2hr, LOW in 4hr

## Jobs To Be Done

| When | I want to | So I can |
|------|-----------|----------|
| My KYC is 7 days from expiry | Be reminded in my usual channel (SMS/in-app), with a clear balance-at-risk amount and 1-tap upgrade CTA | Not lose access to my ₹30K working balance at a toll plaza or metro gate |
| I tap my FASTag at a toll with ₹0 in main wallet | Have the security deposit silently cover the toll | Not be stopped at a boom barrier and not get a failure message on my phone |
| I pay ₹500 at Swiggy with only ₹400 in my Food wallet | Have the remaining ₹100 draw from my main wallet automatically | Not have to mentally reconcile two balances and not have the payment fail |
| My payment is blocked by the Load Guard | Ask the AI support chat "why?" and get an answer that matches the exact numbers in my app | Fix the situation in under 60 seconds without calling a human |
| I'm the ops admin opening the dashboard at 9am | See at a glance: KYC-expiring-today count, open escalations by priority, Support Agent resolution rate overnight | Triage my day in 90 seconds |

## Three Success Metrics

| Metric | Target | Why this |
|--------|--------|----------|
| **KYC expiry → upgrade conversion rate (agent-driven)** | >35% within 72h of first outreach | This is the direct revenue-preservation metric. Every un-upgraded user past grace is a frozen balance and an ops ticket. |
| **Support Agent self-resolution rate** | >60% of chats resolved without human escalation | This is the cost-per-ticket metric. Baseline human-support cost in Indian fintech is ~₹80/ticket; agent cost is ~$0.003 (~₹0.25/chat). Break-even is at 0.3% resolution — anything above that is pure economic win. |
| **Load Guard false-decline rate** | <0.5% (i.e., ≥99.5% of user-attempted loads that should be allowed by regulation are actually accepted) | This is the trust metric. Every false decline is a user question "why can't I add my own money?" and a potential churn event. |

Secondary metrics (track but don't gate on):
- AI cost per active user per month (<₹3.50)
- Mean time from escalation to ops resolution (MTTR, target <2h for HIGH priority)
- Cascade-spend coverage: % of merchant pay attempts that successfully used a sub-wallet instead of main (measures whether sub-wallet benefit is actually being consumed)

## Go / No-Go Criteria for External Use

Before this reference implementation is shown to any external audience (partner bank, regulator, investor, press), **all five** of these must be true:

1. **Compliance verification pass complete.** `docs/compliance-reference.md` has zero `⚠️ TO VERIFY` markers. Every RBI / PMLA / DPDP citation checked against current primary source.
2. **Legal sign-off on design-token attribution.** PPSL Legal has confirmed the Paytm PODS colour tokens + `paytm-wallet-app/` naming are acceptable for public-repo use.
3. **All 4 sibling repos have clean working trees.** No feature described in docs that isn't present on `main` of the relevant repo.
4. **CI green on main.** GitHub Actions runs `test` + `typecheck` + `lint` on every push, main branch protection enforces it.
5. **Executive summary (`docs/executive-summary.md`) approved** by the designated stakeholder (typically the author's reporting manager or the PPSL sponsor).

**Do not externalise if any of these are false.** The status banner at the top of the README is intentionally strong ("not PPSL production code") precisely so the repo is safe to share internally while the above list is being worked through.

## Non-Goals (Explicit)

Listed so no reviewer wastes time asking:

- **Not a production wallet.** See status banner and `scope-and-limitations.md`.
- **Not an API product.** The Express API serves the demo; it is not intended for third-party consumption.
- **Not a pitch for a new PPSL product.** No commercial positioning, no pricing, no roadmap commitment.
- **Not a Claude Code tutorial.** Claude Code is the tool used, but this repo doesn't document Claude Code practices themselves — `docs/claude-code-pm-guide.md` is internal scaffolding, not a generalizable guide.

## One-Liner for Each Audience

- **Technical recruiter:** "Solo PM shipped a reference implementation of an RBI-compliant PPI wallet with autonomous AI agents, using Claude Code."
- **Hiring PM:** "Here's how I think about RBI compliance, AI product strategy, and decomposing ambiguous regulatory requirements into testable invariants."
- **Internal PPSL audience:** "AI-assisted product prototyping demonstrator — not a product; a capability artefact showing what's possible in a 4-week sprint."
- **External partner bank / investor:** [**Do not share externally until all 5 Go/No-Go criteria are met.**]
