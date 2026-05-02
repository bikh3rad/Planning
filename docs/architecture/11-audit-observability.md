# 11 — Audit & Observability

> Two related but distinct concerns: **audit** (immutable forensic record of business events) and **observability** (operational telemetry for running the system). Sources: PRD §12, §15, §34, §44, §137, §214.

## Audit vs observability

| | Audit | Observability |
| --- | --- | --- |
| **Audience** | Compliance, investigators, investors, regulators | Engineers operating the system |
| **Retention** | Forever | Days to months |
| **Mutability** | Strictly immutable, hash-chained, signed | Free to drop, sample, aggregate |
| **Latency** | Eventual (seconds OK) | Near-real-time |
| **Owner** | Audit service + Postgres + cold storage | Standard observability stack |

Both are mandatory. Neither substitutes for the other.

---

## Audit architecture

### Goals

- Reconstruct any past system state by replay
- Answer "who, what, when, why, source, affected entities" for any action
- Provide cryptographic proof that history has not been tampered with

### What gets audited

Every state-changing operation. Read operations of sensitive data (audit queries themselves, exports) are also audited.

### Audit event shape

See [05-events.md](./05-events.md#event-contract-standard) and the `audit_event` table in [04-data-model.md](./04-data-model.md#audit-event-audit-domain).

Key fields:

- `correlation_id` — ties together every event in a single business operation
- `prev_hash` / `event_hash` — per-organization hash chain
- `signature` — service-issued, verifiable

### Hash chain

Per organization, audit events form an unbroken chain:

```
event_hash[n] = SHA-256(canonical(event[n]) || prev_hash[n])
prev_hash[n+1] = event_hash[n]
prev_hash[0] = ZERO
```

Verification:

- Periodic background job walks the chain per org and asserts integrity
- A `GET /audit/integrity?org_id=...` endpoint runs the verification on demand
- Any mismatch is a critical incident — the hash is the canonical "this hasn't been tampered with" signal

### Snapshot signing

Periodically (daily by default), Audit signs and archives a snapshot manifest:

```json
{
  "organization_id": "...",
  "as_of": "2026-05-01T00:00:00Z",
  "head_event_id": "...",
  "head_event_hash": "...",
  "manifest_signature": "..."
}
```

Snapshots go to write-once cold storage (S3 object lock or equivalent). They allow proving the historical state at any point in the past, even if the live database is later compromised.

### Investigation queries

The Audit service exposes query endpoints for investigators (PRD §137):

| Query | Endpoint |
| --- | --- |
| All events for a correlation | `GET /audit/events?correlation_id=...` |
| All events by an actor in a window | `GET /audit/events?actor_id=...&from=...&to=...` |
| Lineage of a transaction | `GET /audit/lineage/{transaction_id}` |
| Approval history of a request | `GET /audit/approvals/{expense_id}` |
| Decision history at a moment | `GET /audit/state-snapshot?correlation_id=...&as_of=...` |

All read endpoints are authenticated, audited (yes, audited), and rate-limited.

### Replay safety

The audit log is the source for replay. Requirements:

- Events are stored in deterministic order per `(organization_id, sequence)`
- Replay rebuilds projection state byte-identically given the same code version
- Code that affects projection logic is versioned; replays must specify which version to use

---

## Observability architecture

Standard three pillars: metrics, logs, traces. All are emitted via OpenTelemetry.

### Tracing

Every external request gets a `trace_id` at the API gateway. It propagates through:

- HTTP via `traceparent` header
- Event bus via the `correlation_id` field in the event envelope (also stored as a trace attribute)

A trace ties together all spans for one business operation, across services.

Required span attributes:

- `organization_id`
- `actor_id` (when known)
- `correlation_id`
- `idempotency_key` (when applicable)

Trace storage: 30-day retention by default; longer for traces flagged by `governance.alert.*` events.

### Metrics

Service-level (per-service):

- `http_requests_total{service, route, status}`
- `http_request_duration_seconds{service, route}` (histogram)
- `event_publish_total{service, event_type, status}`
- `event_consume_lag_seconds{service, event_type}` (histogram)
- `db_query_duration_seconds{service, operation}` (histogram)

Domain-specific (PRD §34, §155):

- `ledger_transactions_committed_total{organization_id}`
- `ledger_validation_failures_total{organization_id, reason}`
- `treasury_locks_total{organization_id, segment}`
- `treasury_balance_negative_attempts_total{organization_id, wallet_id}` — should be 0
- `governance_decisions_total{organization_id, decision}`
- `governance_policy_rejections_total{organization_id, rule_id}`
- `workflow_approval_latency_seconds{organization_id}` (histogram)
- `escrow_lock_duration_seconds{organization_id}` (histogram)
- `audit_hash_chain_verifications_total{organization_id, status}`
- `audit_hash_chain_failures_total{organization_id}` — alerts at >0

### Logs

Structured JSON. Required fields:

- `timestamp` (ISO-8601)
- `level`
- `service`
- `trace_id`
- `correlation_id`
- `organization_id` (when scoped)
- `actor_id` (when scoped)
- `message`

Logs at `WARN` and above carry the originating event payload (sanitized) for postmortem triage.

PII handling: actor names and emails are redacted in logs by default; full PII lives only in the database.

### SLOs

| Service | SLI | SLO |
| --- | --- | --- |
| Ledger | Write latency p99 | < 100 ms |
| Ledger | Write availability | 99.99% over rolling 30 days |
| Treasury | Command latency p99 | < 200 ms |
| Treasury | Command availability | 99.99% |
| Workflow | API latency p95 | < 250 ms |
| Workflow | API availability | 99.95% |
| Governance | Evaluate latency p99 | < 50 ms |
| Audit | Event ingest lag p99 | < 5 s |
| Notification | Delivery success rate | > 99% within 1 minute |

Error budgets enforced via standard SRE practice: budget burned → freeze on non-essential changes.

### Alerting (must page)

- Hash-chain verification failure (any organization)
- Ledger validation failure (any)
- `treasury_balance_negative_attempts_total > 0`
- Audit ingest lag > 60 s
- Any service availability below SLO threshold for 5 min
- `governance.alert.integrity_break` event
- `governance.freeze.triggered` event

### Alerting (informational)

- `governance.alert.liquidity_low`
- `governance.alert.budget_drift`
- `governance.alert.actor_anomaly`
- Approval SLA breach

---

## Read paths for stakeholders

Different audiences see different surfaces:

| Audience | Surface |
| --- | --- |
| Engineers | Grafana dashboards, OTel traces, structured logs |
| Compliance / auditors | Audit service query API + signed snapshot exports |
| Investors | Governance analytics dashboard (read-side projections) |
| Operators / finance | Workflow + Treasury UIs |

The investor / governance dashboard is built from CQRS read projections (PRD §66, §68) maintained from the event stream — eventually consistent, never directly mutating.

## See also

- [05-events.md](./05-events.md) — what gets audited
- [04-data-model.md](./04-data-model.md) — audit_event schema
- [12-security.md](./12-security.md) — who can read audit data
