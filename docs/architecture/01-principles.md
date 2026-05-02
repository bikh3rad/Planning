# 01 — Principles & Invariants

> The non-negotiables. Every architectural decision and every line of production code must respect these. If a design needs to bend one, it requires an ADR. Sources: PRD §5, §8, §11, §12, §31, §41, §69, §145, §170, §194.

## How to use this document

When reviewing a design, walk it against the **invariants** below. If you cannot prove the design preserves all of them, the design is wrong, not the invariant.

When implementing a feature, the **principles** are the lens: did I choose the option that maximizes integrity, traceability, and determinism?

---

## Principles (the lens)

### P1 — Money cannot move without operational context

Every financial state transition carries:

- `actor_id` — who initiated it
- `workflow_id` — what operational process justifies it
- `business_reason` — free-text rationale, captured at the source
- `correlation_id` — links the entire causal chain
- `idempotency_key` — makes retries safe

If any are missing, the operation is rejected at the API boundary. No exceptions, no admin overrides, no "internal" callers.

### P2 — Governance is enforced programmatically, not procedurally

Policies live in code or declarative config that the Governance Engine evaluates. They do not live in PDFs, runbooks, or human review queues. A policy that cannot be expressed as a Governance rule is not a policy — it's a wish.

### P3 — Accounting is a derivative of events, not a source of truth in its own right

The ledger is built by replaying domain events. Treasury balances are projections of ledger entries. There is no scenario where "the database says X but the events say Y" — events win.

### P4 — Treasury access is isolated from operational actors

The people who request and approve spending cannot directly mutate treasury balances. The Treasury service is the only writer; operational services issue *commands* and observe *events*.

### P5 — All financial activity is traceable and immutable

Anyone with audit access can answer, for any historical transaction:

- Who initiated it?
- Who approved it?
- What policy decisions were made and why?
- What ledger entries resulted?
- What was the system state at the time?

If any of these cannot be answered, the system has a bug.

### P6 — Integrity > convenience

Performance, ergonomic APIs, and developer happiness matter. They never matter more than correctness. See PRD §39 for the full priority order.

---

## Invariants (the unbreakable rules)

These are checked by:

- Database constraints where possible
- Application-level guards everywhere else
- A dedicated **invariant test suite** that runs in CI on every change

### Ledger invariants

| ID | Invariant | Enforcement |
| --- | --- | --- |
| L-1 | A ledger entry is never updated or deleted | DB grants: `REVOKE UPDATE, DELETE ON ledger_entries`. See [ADR 0004](../adr/0004-database-immutability-enforcement.md) |
| L-2 | For every transaction, `sum(debits) == sum(credits)` | DB constraint trigger + property test |
| L-3 | Every ledger entry references a non-null `transaction_id`, `account_id`, and `correlation_id` | NOT NULL columns + FK constraints |
| L-4 | No "orphan" entries — every transaction has at least two entries | Trigger + invariant test |
| L-5 | Account balance computed at any point in time is deterministic and replay-safe | Derived purely from immutable entries |

### Treasury invariants

| ID | Invariant | Enforcement |
| --- | --- | --- |
| T-1 | Locked escrow funds cannot be spent for any other purpose | Fund-state machine + Governance gate |
| T-2 | A wallet's available balance is never negative | Application guard + DB CHECK constraint |
| T-3 | Every treasury state transition produces exactly one ledger transaction | Outbox pattern, see [05-events.md](./05-events.md) |
| T-4 | Fund state transitions follow the declared FSM (`available → allocated → locked → released → consumed`) | FSM library + persisted state |

### Governance invariants

| ID | Invariant | Enforcement |
| --- | --- | --- |
| G-1 | A rejected workflow cannot transition into an executed state | Workflow FSM + DB constraint |
| G-2 | Approvals are immutable after submission (no edit, no delete) | Append-only `approvals` table |
| G-3 | An actor cannot approve a request they themselves submitted (unless an explicit policy permits it) | Default-deny in policy aggregator |
| G-4 | Every executed financial action has at least one matching approval recorded | Cross-table invariant test |

### Audit invariants

| ID | Invariant | Enforcement |
| --- | --- | --- |
| A-1 | Every state-changing API call produces at least one audit event | Middleware emits, asserted in CI |
| A-2 | Audit events form an unbroken hash chain per `organization_id` | `prev_hash` column, verified on read |
| A-3 | Audit events are never deleted, only archived | `REVOKE DELETE` + retention policy = forever |

### Multi-tenancy invariants

| ID | Invariant | Enforcement |
| --- | --- | --- |
| M-1 | Every row in every table carries `organization_id` | Schema standard, enforced by linter |
| M-2 | No query reads rows belonging to a different `organization_id` than the request's tenant | Postgres RLS + query helper |
| M-3 | Cross-tenant ledger contamination is impossible | RLS on ledger tables, periodic verification job |

---

## Forbidden patterns

These are prohibited regardless of context:

- **Hard deletes on financial records.** Use soft delete (`deleted_at`) only on operational entities — never on ledger, treasury, audit, or approval tables.
- **Direct balance writes.** No code path may `UPDATE wallets SET balance = ?`. Balance changes flow through events.
- **Silent failures.** Any error that affects financial state must surface to the caller and produce an audit event.
- **Bypass flags / admin overrides.** Admins are not superusers (PRD §58). Emergency overrides require multi-sig and full audit (see [09-governance.md](./09-governance.md#emergency-overrides)).
- **Implicit FX conversions.** Any currency conversion is an explicit, recorded event with documented rate and source.
- **Race-prone balance updates.** Treasury writes use optimistic locking with version columns; idempotency keys deduplicate retries.

---

## Engineering review checklist (per feature)

Before merging any feature touching financial state, the author and reviewer must answer all of these in writing on the PR:

1. **Security:** Can this be abused? Can approvals be bypassed?
2. **Financial integrity:** Can balances desync? Can ledger integrity break?
3. **Concurrency:** What race conditions exist? How are they prevented?
4. **Audit:** Can this action become invisible? What audit events fire?
5. **Idempotency:** What happens if the request is replayed three times?
6. **Multi-tenancy:** Can this leak data across organizations?

A "no" or "I don't know" on any line blocks the merge.
