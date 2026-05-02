# 04 — Data Model

> Canonical entities, account types, and persistence rules. Sources: PRD §7, §24, §52, §54, §71.

This is the conceptual data model. Per-service schemas live with the services; this document is the cross-cutting reference.

## Modeling rules

1. **Every row carries `organization_id`.** No exceptions.
2. **Every row carries `created_at`.** Mutable rows also carry `updated_at` and a `version` integer for optimistic locking.
3. **Financial / audit / approval tables never get `deleted_at`.** Soft-delete is for operational entities only (PRD §24.3).
4. **UUIDs everywhere.** Sequential IDs leak information.
5. **Money is stored as `(amount: bigint_minor_units, currency: char(3))`.** Never floats. Never doubles. See [Currency handling](#currency-handling) below.
6. **Timestamps are `timestamptz` and stored UTC.** Display TZ is a presentation concern.

## Core entities

### Organization (Identity domain)

```
organization {
  id                uuid PK
  name              text
  type              enum(startup, enterprise, fund, ...)
  governance_mode   enum(passive, approval_required, strict, autonomous)  -- PRD §132
  created_at        timestamptz
}
```

### Actor (Identity domain)

```
actor {
  id                uuid PK
  organization_id   uuid FK
  type              enum(user, service_account)
  external_id       text                    -- OIDC sub or similar
  display_name      text
  signing_key_id    uuid                    -- current key, rotates
  status            enum(active, suspended, offboarded)
  created_at        timestamptz
}
```

### Role assignment (Identity domain)

Append-only.

```
role_assignment {
  id                uuid PK
  organization_id   uuid FK
  actor_id          uuid FK
  role              enum(investor, founder, finance_operator, supplier, employee, auditor, admin)
  effective_from    timestamptz
  effective_to      timestamptz NULL        -- NULL = currently active
  granted_by        uuid FK actor
  reason            text
  created_at        timestamptz
}
```

### Wallet (Treasury domain)

```
wallet {
  id                uuid PK
  organization_id   uuid FK
  segment           enum(operational, payroll, escrow, reserve, emergency)  -- PRD §124
  currency          char(3)
  status            enum(active, restricted, frozen, archived)
  version           integer                 -- optimistic lock
  created_at        timestamptz
  updated_at        timestamptz
}
```

The wallet has no `balance` column. Balance is derived from the Ledger.

### Budget (Treasury domain)

```
budget {
  id                  uuid PK
  organization_id     uuid FK
  wallet_id           uuid FK
  budget_type         enum(marketing, payroll, operations, procurement, vendor_payments, emergency)
  allocated_amount    bigint
  currency            char(3)
  reset_cycle         enum(none, monthly, quarterly, annual)
  status              enum(active, restricted, frozen, expired)
  effective_from      timestamptz
  effective_to        timestamptz NULL
  version             integer
  created_at          timestamptz
  updated_at          timestamptz
}
```

`remaining_amount` is **derived** (`allocated_amount - sum(committed transactions in period)`) — not stored. Storing it invites drift.

### Fund position (Treasury domain)

Tracks the lifecycle of a chunk of money allocated for a specific purpose.

```
fund_position {
  id                  uuid PK
  organization_id     uuid FK
  wallet_id           uuid FK
  budget_id           uuid FK NULL          -- NULL for unallocated
  amount              bigint
  currency            char(3)
  state               enum(available, allocated, locked, released, consumed, refunded)
  workflow_id         uuid NULL             -- set when allocated to a workflow
  escrow_id           uuid NULL             -- set when locked into escrow
  version             integer
  created_at          timestamptz
  updated_at          timestamptz
}
```

State transitions per PRD §73 — see [07-treasury.md](./07-treasury.md).

### Expense Request (Workflow domain)

```
expense_request {
  id                  uuid PK
  organization_id     uuid FK
  requestor_id        uuid FK actor
  amount              bigint
  currency            char(3)
  cost_center         text                  -- PRD §80
  business_reason     text NOT NULL
  vendor_id           uuid NULL
  budget_id           uuid FK NULL          -- which budget will fund it
  state               enum(draft, submitted, under_review, approved, rejected, executed, cancelled)
  workflow_definition_id uuid FK
  idempotency_key     text NOT NULL
  correlation_id      uuid NOT NULL
  version             integer
  created_at          timestamptz
  updated_at          timestamptz
  UNIQUE (organization_id, idempotency_key)
}
```

### Approval (Workflow domain)

Append-only. Once written, never updated.

```
approval {
  id                  uuid PK
  organization_id     uuid FK
  expense_request_id  uuid FK
  step_index          integer
  approver_id         uuid FK actor
  decision            enum(approved, rejected)
  reason              text
  signature           bytea NOT NULL        -- signed (expense_id, decision, timestamp)
  governance_decision_id uuid FK NULL      -- link to the policy evaluation
  created_at          timestamptz
}
```

### Ledger Account (Ledger domain)

```
account {
  id                  uuid PK
  organization_id     uuid FK
  code                text                  -- e.g. "1000-cash", org-scoped chart of accounts
  name                text
  type                enum(asset, liability, equity, expense, revenue, escrow)  -- PRD §71
  currency            char(3)
  parent_account_id   uuid NULL             -- for hierarchical CoA
  status              enum(active, archived)
  created_at          timestamptz
  UNIQUE (organization_id, code)
}
```

### Ledger Transaction (Ledger domain)

```
ledger_transaction {
  id                  uuid PK
  organization_id     uuid FK
  correlation_id      uuid NOT NULL         -- traces back to the originating workflow
  description         text
  business_reason     text NOT NULL
  source_event_id     uuid NOT NULL         -- the event that produced this transaction
  idempotency_key     text NOT NULL
  posted_at           timestamptz NOT NULL  -- effective accounting time
  created_at          timestamptz
  UNIQUE (organization_id, idempotency_key)
}
```

### Ledger Entry (Ledger domain)

**Strictly append-only.** See [ADR 0004](../adr/0004-database-immutability-enforcement.md).

```
ledger_entry {
  id                  uuid PK
  organization_id     uuid FK
  ledger_transaction_id uuid FK NOT NULL
  account_id          uuid FK NOT NULL
  side                enum(debit, credit)
  amount              bigint NOT NULL CHECK (amount > 0)
  currency            char(3) NOT NULL
  prev_hash           bytea                 -- hash chain over (org_id, ordered entries)
  entry_hash          bytea NOT NULL
  created_at          timestamptz
}
```

DB grants:
```sql
REVOKE UPDATE, DELETE ON ledger_entry, ledger_transaction FROM application_role;
```

### Escrow (Treasury domain)

```
escrow {
  id                      uuid PK
  organization_id         uuid FK
  expense_request_id      uuid FK
  supplier_id             uuid FK
  fund_position_id        uuid FK             -- the locked funds
  amount                  bigint
  currency                char(3)
  state                   enum(created, funds_locked, awaiting_delivery, delivery_confirmed, delivery_disputed, investigation, released, refunded)
  release_condition       jsonb               -- structured proof requirements
  version                 integer
  created_at              timestamptz
  updated_at              timestamptz
}
```

State machine per PRD §21.2 — see [10-escrow.md](./10-escrow.md).

### Audit Event (Audit domain)

Strictly append-only.

```
audit_event {
  id                  uuid PK
  organization_id     uuid FK
  event_type          text                  -- e.g. "workflow.expense.approved"
  actor_id            uuid NULL             -- NULL for system-generated
  correlation_id      uuid NOT NULL
  source_service      text
  source_event_id     uuid                  -- the originating domain event
  payload             jsonb NOT NULL
  prev_hash           bytea                 -- hash chain per organization_id
  event_hash          bytea NOT NULL
  signature           bytea NOT NULL        -- service signing key
  created_at          timestamptz
}
```

### Governance Decision (Governance domain)

```
governance_decision {
  id                  uuid PK
  organization_id     uuid FK
  correlation_id      uuid NOT NULL
  subject_type        text                  -- e.g. "expense_request", "treasury_release"
  subject_id          uuid
  policy_version      text                  -- which policy bundle evaluated
  inputs              jsonb NOT NULL        -- snapshot of evaluation inputs
  decision            enum(allow, reject, escalate)
  reasons             jsonb                 -- structured reasons per evaluator
  risk_score          numeric(4,3)          -- 0.000–1.000
  evaluated_at        timestamptz
  evaluator_versions  jsonb                 -- per-evaluator versions for replay
  created_at          timestamptz
}
```

Append-only.

## Currency handling

PRD §52 forbids hidden FX. Concrete rules:

- Money is `(amount: bigint, currency: char(3))`. `amount` is in the smallest unit (cents, satoshis, fils).
- A ledger transaction can only contain entries in a single currency. Cross-currency transactions are modeled as **two transactions** plus an explicit `currency.conversion` event recording the rate, source, and timestamp.
- Account currencies are immutable after creation.
- Reporting in a "display currency" is a read-side concern — the underlying ledger entries stay in their native currency.

## Retention

Per PRD §54:

| Table category | Retention |
| --- | --- |
| Ledger entries / transactions | Permanent |
| Audit events | Permanent |
| Approvals | Permanent |
| Governance decisions | Permanent |
| Workflow records (expense_request, etc.) | Permanent (operational soft-delete only) |
| Notification delivery logs | Configurable per organization (default: 2 years) |
| Session data | 90 days |

## See also

- [05-events.md](./05-events.md) — the event contracts that move data between these tables
- [06-ledger.md](./06-ledger.md) — full ledger design including hash chain
- [07-treasury.md](./07-treasury.md) — fund-state machine
- [12-security.md](./12-security.md) — multi-tenancy enforcement (RLS)
