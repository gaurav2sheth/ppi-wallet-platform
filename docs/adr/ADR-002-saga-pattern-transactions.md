# ADR-002: Saga pattern for transactions

**Status:** Accepted
**Date:** 2026-04-11
**Deciders:** Platform team

## Context

Money-moving operations (Add Money, Merchant Pay, P2P, Bill Pay, Wallet-to-Bank) touch multiple data stores:

- Main wallet ledger
- Sub-wallet ledger
- Held funds
- Escalation records
- Reward ledger
- Partner bank (simulated)

A single atomic DB transaction is not feasible across these (especially once a real partner bank API is wired). Without coordination, a partial failure leaves inconsistent state — e.g., money debited from main wallet but bank transfer failed.

## Decision

Model every money-moving operation as a **saga** with:

- A unique `saga_id`
- Ordered `steps[]` with per-step status (PENDING / RUNNING / COMPLETED / FAILED / COMPENSATED)
- An overall `status`: STARTED → RUNNING → COMPLETED (happy path) OR COMPENSATING → COMPENSATED (rollback) OR DLQ (dead-letter)
- Idempotency key on every write step
- Per-step compensating action for rollback

Frontend UX maps directly to saga state:

```
Amount → PIN → API call → Saga starts → Result screen
                                             ├─ COMPLETED → Success
                                             ├─ COMPENSATED → Retry
                                             └─ DLQ → "Contact support"
```

## Consequences

### Positive
- Every money operation is traceable end-to-end
- Rollback is explicit, not inferred
- UI can show progress per step (future UX enhancement)
- Enables idempotent retries without double-charging
- Compliance audit trail built in

### Negative
- More state per operation than a simple transaction
- Compensation logic is separate code path — needs its own tests
- DLQ requires manual ops intervention (human-in-the-loop by design for PPI)

### Not in scope yet
- True distributed saga across multiple services — our current saga is in-process
- Saga orchestrator vs choreography — we use orchestrator pattern (single process coordinates)

## Alternatives considered

| Alternative | Rejected because |
|------------|------------------|
| Two-phase commit (2PC) | Partner banks don't support it; blocks on failure |
| Event sourcing only | Overkill; harder to reason about for a team's first PPI |
| Best-effort with reconciliation job | RBI audit trail needs per-transaction finality |
