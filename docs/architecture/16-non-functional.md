# 16 — Non-Functional Requirements

> Performance, scalability, reliability, disaster recovery, retention. Sources: PRD §13, §14, §54, §55, §56, §85, §86.

## Performance targets

| Operation | Target | Source |
| --- | --- | --- |
| Ledger transaction write | p99 < 100 ms | PRD §56 |
| Workflow validation | p99 < 200 ms | PRD §56 |
| Policy evaluation | p99 < 50 ms | PRD §56 |
| Treasury command (allocate/lock/release) | p99 < 200 ms | derived |
| API gateway end-to-end | p99 < 300 ms | derived |
| Audit ingest lag | p99 < 5 s | derived |
| Notification delivery (after trigger) | p95 < 60 s | derived |

These are targets at production load. Each service includes load tests in CI that exercise 10x these volumes to provide headroom.

## Scalability

### Throughput targets

PRD §13: 1M ledger entries / day.

Translated:
- ~12 ledger entries / sec sustained
- Burst factor ~10 (peak hour might handle 120/sec)
- Per ledger transaction: 2–N entries (typically 2)

So ~6 ledger transactions/sec sustained, ~60/sec burst. Well within a single Postgres node.

### Service scaling

| Service | Scale strategy |
| --- | --- |
| API gateway | Horizontal, stateless, behind LB |
| Auth | Horizontal, stateless |
| Workflow | Horizontal, partitioned by `organization_id` for hot orgs |
| Treasury | Horizontal but with optimistic-lock conflict resolution; advisory locks per wallet for hot contention |
| Ledger | Vertical first (write throughput is single-node-bound); shard by `organization_id` if needed at scale |
| Governance | Horizontal, stateless evaluators; policy bundle cached in Redis |
| Audit | Horizontal consumers; hash chain forces serial commit per organization |
| Notification | Horizontal, stateless |

### Database scaling

- Single Postgres primary per service in v1
- Read replicas for the Ledger and Audit services (heavy read load)
- When `organization_id`-based sharding is required, the schema is already designed for it (no cross-org joins)
- Connection pooling via PgBouncer in transaction-pooling mode for stateless workloads

### Event bus scaling

NATS JetStream:
- Single cluster of 3 nodes for v1
- Streams partitioned by domain (`treasury.*`, `ledger.*`, etc.)
- Per-organization partitioning available within streams when needed

### Eventual consistency

PRD §70 distinguishes consistency requirements:

| Domain | Consistency |
| --- | --- |
| Ledger writes | Strong (single Postgres txn) |
| Treasury state transitions | Strong (single Postgres txn including outbox) |
| Treasury → Ledger | Eventual via event bus, typically < 1 s |
| Read projections (dashboards) | Eventual, typically < 5 s |
| Notifications | Eventual, typically < 60 s |
| Audit ingest | Eventual, typically < 5 s |

## Reliability

### Service uptime targets

PRD §55: 99.99% on critical services.

| Service | Target | Critical? |
| --- | --- | --- |
| API gateway | 99.99% | Yes |
| Auth | 99.99% | Yes (blocks everything) |
| Workflow | 99.95% | Yes |
| Treasury | 99.99% | Yes |
| Ledger | 99.99% | Yes |
| Governance | 99.99% | Yes (blocks all writes) |
| Audit | 99.95% | No (delays acceptable) |
| Notification | 99.5% | No |

### Failure handling

Per PRD §14:

- **Retryable workflows:** every command supports safe retry via idempotency keys
- **Dead-letter queues:** events that fail processing N times move to a DLQ with alerts
- **Event replay:** any consumer can rebuild state from the event log
- **Compensating transactions:** failed sagas roll back via explicit compensating events; never via direct DB rollback across services

### Circuit breakers

- All inter-service synchronous calls (notably Workflow → Governance) wrapped in a circuit breaker
- Open circuit on Governance: Workflow rejects new submissions with 503 — fail closed, not open. Better to refuse than to skip the gate.

### Graceful degradation

| If down | System behavior |
| --- | --- |
| Notification | Workflow continues; notifications buffered |
| Audit (write side) | All services buffer events locally (outbox); resume on recovery |
| Read projection / dashboard | Investor UI shows stale-by-N-minutes warning |
| Governance | All writes refused — fail closed |
| Ledger | All Treasury writes refused — fail closed |

## Disaster recovery

PRD §85, §86.

### Backup policy

| Data | Backup | RTO | RPO |
| --- | --- | --- | --- |
| Ledger DB | Continuous WAL streaming + nightly full | 1 h | 1 min |
| Audit DB | Continuous WAL streaming | 1 h | 1 min |
| Audit cold archive | Daily signed snapshots to write-once storage | 1 h | 24 h |
| Treasury DB | Continuous WAL streaming + nightly full | 1 h | 1 min |
| Workflow DB | Nightly full + daily incremental | 4 h | 24 h |
| Governance DB (decisions) | Continuous WAL streaming | 1 h | 1 min |
| Notification DB | Nightly full | 24 h | 24 h |

### Multi-region

- v1: single region with multi-AZ
- v2: read replicas in a second region; failover documented but manual
- v3: active-active for Audit reads (low write contention)

### Point-in-time restore

PostgreSQL PITR is configured for all tier-1 databases. Restore drills run quarterly.

### Replay-based recovery

Per PRD §49–§50: in a disaster scenario where a projection is lost but the event log survives:

1. Spin up a fresh service instance with empty DB
2. Replay events from the bus (or from the audit cold archive if past retention)
3. Verify projection matches expected state via invariant tests
4. Cut over

This is the **primary** recovery mechanism for read projections and is regularly exercised in CI.

## Data retention

| Category | Retention |
| --- | --- |
| Ledger entries / transactions | Permanent |
| Audit events | Permanent |
| Approvals | Permanent |
| Governance decisions | Permanent |
| Workflow records | Permanent (operational soft-delete only) |
| Notification delivery logs | 2 years (configurable per org) |
| Session data | 90 days |
| Operational metrics | 30 days raw, 1 year downsampled |
| Operational logs | 30 days |
| Operational traces | 30 days (90 days for traces with `governance.alert.*` flag) |
| Outbox rows (after dispatch) | 7 days |

## Capacity planning baseline

For 100 organizations × ~1k transactions/org/day:

| Resource | Estimate |
| --- | --- |
| Ledger entries/day | ~200k (well below 1M target) |
| Postgres storage | ~50 GB / year (ledger) growing |
| NATS messages/day | ~5M |
| Object storage (cold) | ~5 GB / year |
| Compute | ~16 vCPU / 32 GB RAM total across services |

10x growth: still single-region, single-cluster. 100x: introduce sharding by organization_id.

## Testing under load

- Load tests in CI: 10x production targets
- Soak test before any production deploy: 4 hours at production load
- Chaos engineering (post-v1): random pod kills, network partitions, DB failover drills

## See also

- [11-audit-observability.md](./11-audit-observability.md) — SLOs, alerting
- [14-tech-stack.md](./14-tech-stack.md) — infra choices
- [15-roadmap.md](./15-roadmap.md) — when capacity work happens
