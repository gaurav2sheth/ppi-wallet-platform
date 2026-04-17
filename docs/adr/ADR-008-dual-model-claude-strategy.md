# ADR-008: Dual-model strategy (Claude Sonnet + Haiku)

**Status:** Accepted
**Date:** 2026-04-14
**Reviewed:** 2026-04-17 — considered bump to Opus 4.x / Sonnet 4.6, decided to stay
**Deciders:** Platform team

## Context

AI agent workflows have two distinct cognitive loads:

1. **Reasoning** — classify intent, decide priority, choose strategy (KYC REASON, Support UNDERSTAND)
2. **Generation** — write natural SMS/response within constraints (KYC ACT, Support RESPOND)

Using a single model for both is wasteful:
- Using Sonnet everywhere: generation calls over-pay for reasoning capability they don't need
- Using Haiku everywhere: reasoning accuracy drops ~6-10% on structured classification

## Decision

Use **Sonnet for reasoning** and **Haiku for generation**:

| Step | Model | Why |
|------|-------|-----|
| KYC REASON | `claude-sonnet-4-20250514` | Multi-factor decision, structured JSON output, 94% priority accuracy |
| KYC SMS drafting | `claude-haiku-4-5-20251001` | Creative writing within 160 chars, strong at templates |
| KYC in-app drafting | `claude-haiku-4-5-20251001` | Short JSON with title/body/cta, low hallucination risk |
| KYC summary | `claude-haiku-4-5-20251001` | 4-line ops Slack message |
| Support UNDERSTAND | `claude-sonnet-4-20250514` | 13 intents, entity extraction, sentiment |
| Support RESPOND | `claude-haiku-4-5-20251001` | Sentiment-matched reply, constrained length |

## Consequences

### Positive
- **Cost savings:** ~5x reduction per-generation call vs all-Sonnet
- **Latency:** Haiku responses feel instant (~400ms) vs Sonnet (~1.3s)
- **Accuracy preserved:** Sonnet only on classification paths where it matters
- **Clear model-per-task mapping** for future tuning

### Negative
- Two model dependencies to track for deprecation
- Generation tone can differ between the two agents (mitigated by system prompts)
- Must manage API rate limits across both model families

## Model version policy

- **Quarterly review** — re-run golden dataset on latest Claude versions
- **Promote if** +3% accuracy OR -20% cost with equal quality
- **Current pins:**
  - Sonnet: `claude-sonnet-4-20250514` (evaluated Sonnet 4.6 on 2026-04-17; +1.8% accuracy, +18% cost; not yet promoted pending regression testing)
  - Haiku: `claude-haiku-4-5-20251001` (stable, no challenger)
- **Opus 4.x** — reserved for judge-disagreement spot checks (10% sampling); not used in production paths

## Alternatives considered

| Alternative | Rejected because |
|------------|------------------|
| Sonnet everywhere | 5x cost, latency hit on generation |
| Haiku everywhere | -6-10% classification accuracy |
| Opus for reasoning | +6x cost, +2.5% accuracy — not worth it |
| Fine-tuned Haiku for intent | Worth revisiting at 1M+ DAU; not now |
| Mix Claude + GPT-4o | Eval showed -3% accuracy on our domain; split tooling |
