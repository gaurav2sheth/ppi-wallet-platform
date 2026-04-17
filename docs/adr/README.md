# Architecture Decision Records (ADRs)

Each ADR documents a single, significant architectural decision. The format
follows [Michael Nygard's template](https://cognitect.com/blog/2011/11/15/documenting-architecture-decisions.html):

- **Status** — Proposed / Accepted / Deprecated / Superseded by ADR-XXX
- **Context** — What is the problem or forcing function?
- **Decision** — What did we decide?
- **Consequences** — What are the tradeoffs? What gets easier, what gets harder?

## Index

| # | Title | Status | Area |
|---|-------|--------|------|
| [001](ADR-001-three-tier-api-fallback.md) | Three-tier API fallback | Accepted | Frontend architecture |
| [002](ADR-002-saga-pattern-transactions.md) | Saga pattern for transactions | Accepted | Transaction engine |
| [003](ADR-003-subwallets-in-rbi-cap.md) | Sub-wallet balances count toward ₹1L RBI cap | Accepted | Compliance |
| [004](ADR-004-fastag-security-deposit.md) | FASTag as security deposit model | Accepted | Sub-wallets |
| [005](ADR-005-ncmc-isolated-balance.md) | NCMC isolated balance (no cascade) | Accepted | Sub-wallets |
| [006](ADR-006-localstorage-versioning.md) | localStorage key versioning | Accepted | Frontend state |
| [007](ADR-007-ai-context-sync.md) | AI context sync (client provides ground truth) | Accepted | AI agents |
| [008](ADR-008-dual-model-claude-strategy.md) | Dual-model strategy (Sonnet + Haiku) | Accepted | AI agents |
| [009](ADR-009-five-shot-learning.md) | 5-shot in-context learning for KYC agent | Accepted | AI agents |
| [010](ADR-010-in-memory-escalation-store.md) | In-memory shared escalation store | Accepted | AI agents |
| [011](ADR-011-monetary-arithmetic-in-paise-bigint.md) | Monetary arithmetic in paise with BigInt | Accepted | Foundations |

## Creating a new ADR

1. Copy an existing ADR as a template
2. Number sequentially (`ADR-011-...`)
3. Fill in Context / Decision / Consequences
4. Add to the index table above
5. If superseding, mark the old ADR as "Superseded by ADR-XXX" and link

## When to write an ADR

Write one when a decision:
- Is hard to reverse (data model shapes, wire protocols, regulatory rules)
- Has non-obvious tradeoffs that future contributors will question
- Affects multiple repos or teams
- Was contested — i.e., at least 2 viable alternatives were considered

Do NOT write an ADR for:
- Pure style choices (use the style guide)
- Library picks with clear winners (use `package.json`)
- Implementation details that fit in a code comment
