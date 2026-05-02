# ADR 0003 — NATS JetStream vs Kafka

**Status:** Accepted
**Date:** 2026-05-02
**Deciders:** Solution architecture

## Context

The PRD calls for an event bus and lists "Kafka or NATS" as the recommendation (PRD §25). We need to pick one.

Forces:

- Throughput target: 1M ledger entries/day ≈ 12/sec sustained, ~120/sec peak
- ~5M total events/day across all event types
- Ordering: per-organization ordering matters for the Audit hash chain; global ordering does not
- Durability: events must survive broker crashes; "at-least-once" delivery is required
- Replay: consumers must be able to replay from a position
- Operational complexity: small team, can't dedicate someone to message broker babysitting
- Multi-tenancy: 100s of organizations in v1, growing
- Cloud independence: should run on any cloud or on-prem

## Decision

Use **NATS JetStream**.

- Streams partitioned by domain (`treasury.*`, `ledger.*`, `governance.*`, etc.)
- Per-organization ordering achieved via subject naming: `treasury.org_<uuid>.funds.locked`
- 3-node cluster in v1; scales horizontally if needed
- Consumers use durable JetStream consumers with explicit acks
- Retention: configurable per stream (7 days for most, 90 days for audit triggers)

## Alternatives considered

- **Apache Kafka** — battle-tested, the default for financial event streaming. But:
  - Operational weight: ZooKeeper (or KRaft) + brokers + Schema Registry + Kafka Connect ecosystem. We don't need most of it.
  - At our throughput it is dramatically over-provisioned.
  - JVM operational quirks (GC tuning, off-heap memory) eat engineering time.
  - The team has no production Kafka experience.
  - Vendor-managed (MSK, Confluent Cloud) reduces ops but adds spend and lock-in.
  - The features that make Kafka unique (massive throughput, KSQL, deep partitioning) are unused.
  - **Reject for v1.** Reconsider if we cross 100k events/sec or need cross-region active-active.

- **AWS SQS + SNS** — managed, simple. But:
  - Locks us to AWS
  - Replay support is poor (SQS messages are deleted on ack; SNS doesn't persist)
  - No native ordering guarantees within a topic at scale (FIFO has tight throughput limits)
  - **Reject.** Cloud lock-in + replay weakness are dealbreakers.

- **Google Pub/Sub / Azure Service Bus** — same class of objections as SQS+SNS. Reject.

- **RabbitMQ** — well-known, simpler than Kafka. But:
  - Replay support is awkward (streams are a recent addition)
  - Multi-tenant fan-out at scale gets complex
  - JetStream's declarative consumer model is cleaner
  - **Reject.** NATS is the modern equivalent with better fit.

- **Postgres LISTEN/NOTIFY + a custom outbox-shipper** — radically simple, no extra infra. But:
  - Doesn't support real consumer groups, durable replay, or fan-out at scale
  - Couples all event traffic to one Postgres
  - **Reject** as the primary bus, though the outbox itself sits in Postgres.

## Consequences

### Positive

- Lightweight: 3-node cluster runs in ~3 GB RAM total
- Single binary, written in Go — fast to start, easy to operate
- JetStream provides durability + replay + at-least-once semantics natively
- Subject hierarchy maps naturally to our `domain.org.entity.action` pattern
- Multi-tenancy via subject prefixing and per-account isolation
- Built-in observability (NATS exposes Prometheus metrics out of the box)
- Cloud-neutral

### Negative

- Smaller community than Kafka; harder to hire Stack Overflow answers
- Less mature client ecosystem in some languages (TS/Node is fine)
- JetStream tooling is less rich than Kafka's (no Confluent UI equivalent — `nats-top`, `nsc`, and Synadia's UI cover the basics)
- If we ever need 100k+ msg/sec sustained, we'd revisit Kafka

### Revisit when

- We approach 10k events/sec sustained
- We need cross-region active-active replication
- A specific consumer needs streaming SQL (KSQL-equivalent) — though we'd more likely solve with a different tool than swap brokers
- The team grows enough to dedicate a platform engineer to message infrastructure

## Related

- [docs/architecture/05-events.md](../architecture/05-events.md)
- [docs/architecture/14-tech-stack.md](../architecture/14-tech-stack.md)
- [docs/architecture/16-non-functional.md](../architecture/16-non-functional.md)
- ADR 0002 (outbox pattern depends on this choice)
