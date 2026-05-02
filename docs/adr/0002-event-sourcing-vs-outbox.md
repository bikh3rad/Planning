# ADR 0002 — Event sourcing vs CRUD-with-outbox

**Status:** Accepted
**Date:** 2026-05-02
**Deciders:** Solution architecture

## Context

The PRD strongly recommends event sourcing (PRD §66, §67) for ledger and workflow systems and requires event-driven coordination across services. We need to choose between:

- **Pure event sourcing**: state is the fold of an immutable event log; current-state tables are projections that can be rebuilt
- **CRUD-with-outbox**: state lives in normal tables; a transactional outbox guarantees that state changes and event publication happen atomically

Forces:

- Auditability: every change must produce an event, no exceptions
- Replay: must be able to reconstruct projections from events
- Team experience: the team knows CRUD; pure ES has a steep learning curve and a thin tooling ecosystem in the Node.js world
- Operational complexity: pure ES requires snapshots, projection rebuilds, and careful schema versioning of every event type forever
- The Ledger is naturally event-sourced (entries *are* immutable events), but most other domains have a small set of mutable entities with clear current state

## Decision

Use **CRUD-with-outbox** as the default pattern across services.

The Ledger is the exception: its `ledger_entry` table *is* the event log. There are no separate "current state" tables for accounts beyond cached snapshots that are explicitly projections.

Concretely:

1. Each service owns a Postgres schema with state tables and an `outbox` table
2. State changes write the new state row(s) AND insert the corresponding event row(s) into the outbox in the same DB transaction
3. A separate `outbox_relay` process reads outbox rows and publishes to NATS, marking each row dispatched
4. Consumers dedupe on `event_id`

## Alternatives considered

- **Pure event sourcing across all services** — most architecturally pure. But: heavy ceremony for simple CRUD use cases (workflow definitions, notification preferences, supplier records); tooling immaturity in the Node.js world; team learning curve; harder to query current state. Reject for v1; reconsider for specific subsystems if pain emerges.
- **Two-phase commit / distributed transactions** — would solve the dual-write problem differently. But: requires XA-capable infra (Postgres + NATS would not coordinate cleanly); operational complexity is high; pretty universally regretted. Reject.
- **Change Data Capture (Debezium on Postgres WAL)** — clean separation, no application-level outbox code. But: introduces Debezium + Kafka Connect into the stack (we're choosing NATS — see ADR 0003); WAL-derived events lack business semantics (you get rows, not events); harder to enrich with business context. Reject.
- **Best-effort dual write** (write to DB then publish to bus, hope both succeed) — common, simple, broken. Will silently lose events on partial failures. Reject.

## Consequences

### Positive

- Atomicity guaranteed: state and event publication cannot diverge
- Familiar mental model for the team
- Easy to query current state (it's in normal tables)
- Easy to evolve schema using normal migrations
- Ledger still gets the auditability of pure ES where it matters most

### Negative

- Outbox table requires a relay process per service (operational surface)
- Outbox rows accumulate and need cleanup (7-day retention after dispatch — see [16-non-functional.md](../architecture/16-non-functional.md#data-retention))
- Replay rebuilds projections, but for non-Ledger services replay rebuilds **only event-driven projections**, not the primary state (which is authoritative). For full disaster recovery, primary state restores from backup; events handle propagation to other services.
- "Event-sourced or not?" is a per-service answer — the team must understand the distinction

### Revisit when

- A specific service's current-state schema becomes a bottleneck and an event-sourced design would be clearer
- We need true time-travel queries on a non-Ledger domain (in which case event-source that domain)
- Outbox relay complexity becomes a problem (consider Debezium at that point)

## Related

- [docs/architecture/05-events.md](../architecture/05-events.md)
- [docs/architecture/06-ledger.md](../architecture/06-ledger.md)
- ADR 0003 (messaging choice)
