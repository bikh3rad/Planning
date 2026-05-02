# 15 — Delivery Roadmap

> What we build, in what order, and what we explicitly defer. Sources: PRD §18, §19, §61.

This is a phased plan, not a calendar. Estimates assume one focused engineer; multiply / divide as the team scales.

## Guiding rules

- **Build the load-bearing things first.** The Ledger is mandatory before anything else can be trusted.
- **No phase ships without its invariant tests.** If we can't prove the invariants hold, we don't ship.
- **No phase ships without an audit trail.** Even Phase 0 emits structured events (we just don't consume them yet).
- **Each phase has an external demo.** Internal milestones don't count.

## Phase 0 — Foundation (1–2 weeks)

**Goal:** A clean repo with the cross-cutting concerns wired up.

Deliverables:
- Monorepo scaffolded per [14-tech-stack.md](./14-tech-stack.md)
- `libs/event-contracts`, `libs/crypto`, `libs/postgres`, `libs/messaging`, `libs/observability` skeletons
- Local dev stack (`docker compose`): Postgres + Redis + NATS
- API gateway shell with required-headers enforcement
- CI: lint, typecheck, unit, migration validation, invariant test runner stub
- Multi-tenant `organization_id` baked into the event envelope and DB helpers
- Auth service: minimal — issues / verifies JWTs against a seeded user table

Demo: `curl` an authenticated request through the gateway, see it logged with trace_id, see the audit event land in a stub consumer.

Exit criteria:
- All future phases can plug into this scaffold without re-litigating cross-cutting choices
- ADRs 0001–0004 closed

## Phase 1 — Ledger MVP (2–3 weeks)

**Goal:** A working append-only double-entry ledger.

Deliverables:
- Account model + chart-of-accounts management
- `ledger_transaction` + `ledger_entry` with append-only enforcement (DB grants + triggers)
- Hash chain implementation + verification job + verification API
- Idempotency on transaction posting
- Balance projection (real-time + cached snapshots)
- Trial balance report
- Internal-only API; no governance gating yet (Treasury isn't built)
- **Invariant test suite for L-1 through L-5**

Demo: post 10,000 randomly generated balanced transactions; verify trial balance balances; tamper with an entry (via raw SQL); verify the chain detects it.

Exit criteria: invariant tests pass; Ledger handles 1k transactions/sec sustained in a load test (10x the production target).

## Phase 2 — Treasury MVP (2 weeks)

**Goal:** Wallets, budgets, fund-state lifecycle, payment execution that produces ledger entries.

Deliverables:
- Wallet, budget, fund_position tables and services
- Fund-state machine with FSM library
- Outbox pattern: Treasury writes state + event in one DB txn; relay publishes
- Ledger consumer in Treasury that reacts to `ledger.transaction.committed` (or rejection)
- `POST /treasury/allocate`, `/lock`, `/release`, `/refund`
- Wallet balance derivation from Ledger
- **Invariant tests T-1 through T-4**

Demo: allocate budget, lock funds for a fictitious payment, release, verify ledger entries land and balance derivation matches.

Exit criteria: T-* invariants pass; concurrent allocation tests show no race conditions (property-based test with N concurrent allocators).

## Phase 3 — Workflow Engine (2–3 weeks) — first user-visible milestone

**Goal:** End-to-end expense → approval → payment → ledger.

Deliverables:
- Workflow definition + instance tables
- Expense request FSM
- Approval chain execution (sequential, multi-step)
- Approval signing + verification
- Saga orchestrator: approved → treasury allocate → release → ledger commit (with compensation)
- API: `/expenses` family
- **Invariant tests G-1 through G-4** (subset that doesn't require Governance yet — self-approval check, FSM enforcement)

**Stub Governance:** for this phase, Governance is a fixed `allow` for everything that isn't a self-approval. Real Governance lands in Phase 4.

Demo: a real user creates an expense in Postman, approver approves, payment executes, ledger shows the entries, audit log shows the full chain.

Exit criteria: golden-path demo runs reliably; failure modes (governance reject, treasury lock failure, ledger reject) trigger correct compensation; all approval signatures verify.

## Phase 4 — Governance Engine (2 weeks)

**Goal:** Real policy evaluation replacing the Phase 3 stub.

Deliverables:
- Policy bundle storage with versioning
- Evaluators: `budget`, `threshold`, `role`, `temporal`, `velocity`, `supplier` (basic), `compliance` (basic)
- Aggregator with deny-overrides semantics
- Decision persistence + explainability endpoint
- Wire Workflow + Treasury to call Governance at all gate points
- Risk scoring (rules-based; no ML yet)
- Continuous monitoring jobs for `governance.alert.*` events

Demo: configure a per-org policy bundle (budget caps, night-time review, vendor verification); push expenses through that hit each rule; see explainable decisions in the API.

Exit criteria: evaluator p99 < 50 ms; decisions are deterministic in replay tests.

## Phase 5 — Escrow (2 weeks)

**Goal:** Conditional release flows for vendor payments.

Deliverables:
- Escrow tables, FSM
- Verification engine (manual confirmation + document verification condition types)
- Dispute handling
- Supplier risk feedback loop
- API: `/escrow` family
- Integration with Workflow (escrow-typed expense requests)

Demo: vendor expense → escrow created → delivery confirmation submitted → escrow releases → ledger reflects; alternative path: dispute → investigation → refund → reversing ledger entry.

Exit criteria: escrow state machine invariants hold; locked funds verifiably cannot be reused.

## Phase 6 — Audit & Observability (2 weeks)

**Goal:** First-class Audit service + production-grade observability.

Deliverables:
- Audit service: ingests every `audit.*` event, writes hash-chained `audit_event` rows
- Audit query API (correlation, actor, time-range, lineage, integrity)
- Daily snapshot signing + cold storage
- OTel instrumentation across all services
- Grafana dashboards per service + per domain
- Alerting wired up (must-page list from [11-audit-observability.md](./11-audit-observability.md))
- Log aggregation

Demo: investigate a synthetic incident — pull the full event chain for a correlation_id; verify integrity; see the trace in Tempo, the metrics in Grafana, the logs in Loki.

Exit criteria: audit hash chain integrity verifies for 1M synthetic events; SLOs measurable from Grafana.

## Phase 7 — Notification Service (1 week)

**Goal:** Email + webhook delivery driven by `notification.*` events.

Deliverables:
- Notification service consuming notification events
- Email channel (SES or equivalent)
- Webhook channel with HMAC signatures
- Delivery state tracking + retry with exponential backoff
- Per-user delivery preferences

Demo: expense rejection → user receives email; budget threshold breach → webhook fires.

## Phase 8 — Investor Dashboard / Read Models (2–3 weeks)

**Goal:** Read-side projections powering the investor view.

Deliverables:
- CQRS read models: burn rate, cash runway, budget usage, department spend
- Eventual-consistency projections from the event stream
- Read API for the dashboard
- (Frontend is a separate workstream — out of scope for this phase)

Exit criteria: read models reconstructable by replay; dashboard reflects events within 5 seconds p95.

---

## Total: ~16–20 weeks for v1, single engineer

A team of three would land this in ~8–10 weeks if work is parallelizable along service boundaries (which it is, after Phase 1).

---

## Explicitly out of scope for v1

The PRD includes a great deal of forward-looking material (sections 100–218). For v1 we are **deliberately not building**:

- AI-driven anomaly detection (PRD §16, §35, §77, §187, §206)
- Predictive runway forecasting / treasury simulation (PRD §88, §134, §135)
- Self-healing governance (PRD §189)
- Autonomous restriction engine (PRD §199)
- Treasury intelligence / capital efficiency analytics (PRD §200, §159, §160)
- Behavioral monitoring / financial nervous system (PRD §127, §198)
- Stress testing framework (PRD §135, §165)
- On-chain settlement, Chainlink, Kleros (PRD final-objective section)
- Cross-currency liquidity pool infrastructure
- Insurance and customs workflows
- Operational digital twin (PRD §186)

These belong in a v2+ roadmap; the v1 architecture is designed to leave room for them without rework. In particular: Governance evaluators are pluggable, the event stream supports unlimited additional consumers, and read projections can be added without touching the write side.

## Per-feature engineering review

Per PRD §19, §61, every feature merge requires:

1. Security review — see [12-security.md](./12-security.md#security-review-checklist-per-pr)
2. Financial integrity review — invariant tests + reasoning on ledger consequences
3. Concurrency review — what happens under N concurrent calls?
4. Audit review — what events fire, are they sufficient to reconstruct the action?
5. Scaling review — does this scale to 10x the target load?

A "no" or "I don't know" on any line blocks the merge.

## See also

- [01-principles.md](./01-principles.md) — invariants every phase must preserve
- [14-tech-stack.md](./14-tech-stack.md) — the stack we're building on
