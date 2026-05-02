# Architecture Decision Records

Decisions that close off other reasonable options live here. The format is lightweight (Michael Nygard style).

## Why ADRs

When we look at this repo in 12 months, "why did we pick NATS over Kafka?" should be answerable in 30 seconds. ADRs make that possible.

## Index

| # | Title | Status |
| --- | --- | --- |
| [0001](./0001-monorepo-tooling.md) | Monorepo tooling | Accepted |
| [0002](./0002-event-sourcing-vs-outbox.md) | Event sourcing vs CRUD-with-outbox | Accepted |
| [0003](./0003-messaging-nats-vs-kafka.md) | NATS JetStream vs Kafka | Accepted |
| [0004](./0004-database-immutability-enforcement.md) | Database-level immutability enforcement | Accepted |

## When to write a new ADR

Write one if:

- The decision closes off another reasonable option ("we picked X *over* Y")
- The decision will be hard to reverse later
- A future engineer might be tempted to ask "why don't we just…?"

Don't write one for:

- Style choices that don't affect architecture
- Decisions already documented in the architecture docs without close alternatives
- Implementation details internal to a single service

## Format

Copy this template into `NNNN-slug.md`, where `NNNN` is the next available number:

```markdown
# ADR NNNN — Title

**Status:** Proposed | Accepted | Deprecated | Superseded by ADR-NNNN
**Date:** YYYY-MM-DD
**Deciders:** name(s)

## Context

What is the situation? What forces are at play? What constraints?

## Decision

What did we decide?

## Alternatives considered

- **Option A** — what it is, why we didn't pick it
- **Option B** — same

## Consequences

### Positive

- What this gets us

### Negative

- What this costs us
- What we'll need to revisit

## Related

Other ADRs, architecture docs, PRD sections.
```

## Lifecycle

- **Proposed** — under discussion, not yet committed
- **Accepted** — current truth; behavior should follow
- **Deprecated** — no longer recommended but no replacement yet
- **Superseded** — replaced by a newer ADR (link forward)

ADRs are append-only. To change a decision, write a new ADR that supersedes the old one and update the old one's status to `Superseded by ADR-NNNN`.
