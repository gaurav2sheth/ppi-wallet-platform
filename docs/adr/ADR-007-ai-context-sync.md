# ADR-007: AI context sync — client provides ground truth

**Status:** Accepted
**Date:** 2026-04-16
**Deciders:** Platform team
**Supersedes:** Server-only mock data for AI context

## Context

The Customer Support Agent was giving inaccurate answers: users reported "I asked about my balance and the AI said ₹1,245.50 but my app shows ₹236.11."

Root cause: the server-side agent read from `mcp/mock-data.js` (seeded 200 test users), while the wallet app read from `localStorage` (user's actual demo state). These were different data universes. Every question about the user's *own* data diverged.

Two options to fix:
1. **Sync server → client** on every app boot (heavy, race-prone, violates offline promise)
2. **Sync client → server** on every chat request (lightweight, stateless, works offline)

## Decision

The **frontend sends real app state as `context`** in every `/api/support/chat` request:

```json
{
  "user_id": "demo-wallet",
  "message": "What is my balance?",
  "session_id": "SESS-xxx",
  "context": {
    "balance_paise": "23611",
    "balance_formatted": "₹236.11",
    "user_name": "Aarav",
    "kyc_tier": "FULL",
    "recent_transactions": [ /* last 10 from localStorage */ ]
  }
}
```

The agent's `investigate()` step **prefers client context over server mock data** when present:
- `clientContext:balance` replaces `getWalletBalance(userId)`
- `clientContext:transactions` replaces `getTransactionHistory(userId)`
- Other tools (sub-wallets, blocked attempts, compliance) still come from server — not in client context

## Consequences

### Positive
- AI answers now match the app exactly — trust restored
- Works in mock mode (client is the source of truth anyway)
- Works in production (client has the authoritative read-through cache)
- Zero server-state coupling for the "user's own data" question class
- Easy to verify: tools_used array includes `clientContext:*` tags

### Negative
- Larger request payload (~3-5KB per chat) — negligible at our volume
- Client can tamper with context before sending — BUT this only affects the user's own view, not server-side data. Not a trust issue at user-read scope.
- Admin dashboard is more complex: when admin queries about `user_XYZ`, admin dashboard must look up `user_XYZ`'s data in its mock layer and pass that as context
- Two code paths: "user's own data" (client context) vs "cross-user data" (server only)

### Trust boundary
Client context is trusted for:
- Balances / transactions **belonging to the same authenticated user**
- Display-time formatting hints

Client context is NEVER trusted for:
- Balance caps (server enforces)
- Load Guard approvals (server enforces)
- KYC state (server enforces)
- P2P approvals (server enforces)

Anything money-moving re-validates server-side.

## Alternatives considered

| Alternative | Rejected because |
|------------|------------------|
| Server pulls from client on-demand | Race conditions between chat turns |
| Unified data store | Defeats offline/mock mode (ADR-001) |
| Client ignores server response | Cross-user queries (admin) break |
| Embed context via hidden system message | Harder to audit in logs |
