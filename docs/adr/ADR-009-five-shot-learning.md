# ADR-009: 5-shot in-context learning for KYC agent

**Status:** Accepted
**Date:** 2026-04-14
**Deciders:** Platform team

## Context

The KYC agent's REASON step classifies each at-risk user into priority (P1-P4), intervention strategy, tone, and offer. Options for teaching the model this policy:

1. **Fine-tune** on historical decisions → expensive, slow iteration, creates model drift
2. **0-shot prompting** → 74% accuracy, hallucinates edge cases
3. **N-shot in-context learning** → tradeoff between accuracy and token cost

Need to pick N.

## Decision

Use **5-shot in-context learning** with a **diversity-optimized exemplar set**:

| # | Archetype | Purpose |
|---|-----------|---------|
| 1 | P1_CRITICAL: 1 day, ₹75K, HIGH activity | Anchor the urgent-multi-touch class |
| 2 | P2_HIGH: 4 days, ₹15K, MEDIUM activity | Anchor standard outreach |
| 3 | P3_MEDIUM: 6 days, ₹2K, HIGH activity | Anchor gentle reminder |
| 4 | P4_LOW: 7 days, ₹200, DORMANT | Anchor monitor-only |
| 5 | Edge case: 2 days, ₹7.5K, HIGH activity | Teach priority inversion (habit continuity) |

Exemplars are ordered by diversity (not recency) and placed before the target user at the END of the prompt (recency bias favors the target).

## Evidence

On the 200-case golden set:

| Strategy | Priority accuracy | Hallucination rate | Tokens in | Cost per user |
|---------|-------------------|-------------------|-----------|---------------|
| 0-shot | 74% | 8.2% | 450 | $0.0014 |
| 3-shot | 89% | 3.1% | 920 | $0.0028 |
| **5-shot** | **94%** | **1.4%** | **1,380** | **$0.0041** |
| 7-shot | 94.5% | 1.2% | 1,840 | $0.0055 |
| 10-shot | 94.3% | 1.3% | 2,560 | $0.0077 |

**5-shot is the knee of the accuracy curve** — beyond 5, marginal accuracy gains (<0.5%) don't justify 33%+ token cost.

## Consequences

### Positive
- 94% priority accuracy vs 74% 0-shot — 20 point lift
- Exemplars are version-controlled, diffable, and reviewable
- Easy to iterate: swap an exemplar, rerun golden set, ship if +1% accuracy
- No fine-tuning infrastructure needed

### Negative
- Token cost per reasoning call is +3x vs 0-shot (but still $0.004)
- Exemplar drift — exemplars must be re-validated when policy changes
- "Priority inversion" exemplar (case 5) is counterintuitive and can confuse future contributors — documented explicitly in `docs/kyc-agent.md#38-exemplar-5--priority-inversion-edge-case`

### Maintenance rules
- Any change to exemplars requires re-running the full golden set
- New exemplar added → must document rationale in this ADR amendment
- Exemplar removed → must have a replacement covering that region of the decision space

## Alternatives considered

| Alternative | Rejected because |
|------------|------------------|
| Fine-tune Sonnet | $$ and slow iteration; our policy changes quarterly |
| Rule-based only | 76% accuracy ceiling; doesn't handle priority inversion |
| Chain-of-thought (CoT) | +1.5% accuracy for +66% cost; not worth it |
| Dynamic exemplar retrieval (RAG) | Overkill at 200 users/day |
