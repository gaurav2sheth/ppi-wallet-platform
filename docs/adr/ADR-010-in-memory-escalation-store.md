# ADR-010: In-memory shared escalation store

**Status:** Accepted
**Date:** 2026-04-15
**Deciders:** Platform team

## Context

Both the KYC Upgrade Agent and the Customer Support Agent create escalations to ops when they can't resolve an issue. Open questions:

- Separate stores per agent, or shared?
- In-memory, file-backed, or DB-backed?
- How do we avoid duplicate escalations for the same user across agents?

Real production would use a DB (Postgres), but this is a reference implementation — want the simplest thing that shows the design correctly.

## Decision

A **single in-memory escalation store**, shared between both agents, with a `source` field distinguishing origin:

```javascript
// mcp/agents/escalation-manager.js
let escalations = [];  // module-level

export function escalateToOps(userId, context) {
  escalations.push({
    escalation_id: `ESC-${Date.now()}-${userId}`,
    user_id: userId,
    source: context.source,  // 'KYC_AGENT' or 'SUPPORT_AGENT'
    priority: context.priority,
    ...context,
    status: 'OPEN',
    created_at: formatIST(),
  });
  return escalations[escalations.length - 1];
}
```

Dedup is by `(user_id, day)` — if an escalation exists for the same user on the same day, update rather than create duplicate.

## Consequences

### Positive
- Single source of truth for ops dashboard
- Admins see one unified queue, not two
- Shared resolve flow — a KYC escalation and a Support escalation resolve the same way
- Minimum code

### Negative
- **All escalations lost on server restart** — unacceptable for production, acceptable for demo
- No persistence across Render free-tier cold starts (wake-up drops state)
- No cross-instance coordination if we horizontally scaled
- No historical analytics — can't answer "how many escalations last quarter?"

### Migration path to production
When we move to real ops, this module becomes a thin wrapper over Postgres:
- Same API surface (`escalateToOps`, `getEscalations`, `resolveEscalation`)
- Swap the `escalations[]` array for a `db.query()` call
- Add indexes on `(status, priority)` and `(user_id, created_at)`
- Schema already defined by the current shape

The in-memory implementation was designed explicitly to be swap-in-replaceable.

## Alternatives considered

| Alternative | Rejected because |
|------------|------------------|
| Separate stores per agent | Duplicate escalations for same user; messy ops UX |
| SQLite file | File state tricky on serverless platforms (Render free tier) |
| Redis | Extra infra for demo |
| Direct Postgres | Right answer for production; overkill for reference impl |
| JSONB file with append-only writes | Race conditions; no TTL |
