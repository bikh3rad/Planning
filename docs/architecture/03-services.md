# 03 — Service Catalog

> One row per service: what it does, what it doesn't, what it talks to. Sources: PRD §6, §65.

## Catalog

| Service | Domain | External API? | Owns DB schema | Consumes events | Emits events |
| --- | --- | --- | --- | --- | --- |
| Auth | Identity | Yes | `identity` | — | `audit.actor.*` |
| Workflow | Workflow | Yes | `workflow` | `treasury.*`, `governance.*` | `workflow.*`, `audit.workflow.*` |
| Treasury | Treasury | Yes | `treasury` | `workflow.expense.executed`, `governance.decision.*` | `treasury.*`, `audit.treasury.*` |
| Ledger | Ledger | Read-only | `ledger` | `treasury.transaction.committed` | `ledger.*`, `audit.ledger.*` |
| Governance | Governance | Internal-only | `governance` | All `audit.*` for analysis | `governance.decision.*`, `governance.alert.*` |
| Audit | Audit | Read-only | `audit` | All `audit.*` events | — |
| Notification | Notification | Internal-only | `notification` | `notification.*` triggers from any service | `notification.delivered`, `notification.failed` |

## Per-service detail

### Auth Service

**Purpose:** Authenticate actors, attest their roles and cryptographic identity.

**Responsibilities:**
- User / service-account authentication (OIDC, mTLS for service-to-service)
- Session management
- Role assignment within an organization
- Key issuance for event signing
- RBAC policy storage (the *what roles exist*, not the *what they can do in business context*)

**Non-responsibilities:**
- Business-policy decisions (Governance)
- Approval workflows (Workflow)

**Key endpoints:**
- `POST /sessions` — login
- `GET /actors/me` — current actor + claims
- `POST /service-accounts/{id}/keys/rotate` — key rotation

**Critical invariants:**
- Sessions expire and cannot be extended
- Role assignments are append-only with effective_from/effective_to timestamps

---

### Workflow Service

**Purpose:** Orchestrate the lifecycle of expense and payment requests.

**Responsibilities:**
- Expense request CRUD and state machine (PRD §21.1)
- Approval chain execution
- Workflow definitions (org-configurable)
- Calling Governance at decision points
- Issuing execution commands to Treasury once fully approved

**Non-responsibilities:**
- Deciding whether something is allowed (Governance)
- Moving funds (Treasury)
- Generating ledger entries (Ledger)

**Key endpoints:**
- `POST /expenses`, `POST /expenses/{id}/approve`, `POST /expenses/{id}/reject`
- `GET /expenses/{id}` (with full approval history)
- `POST /workflow-definitions` (admin)

**Critical invariants:**
- An expense cannot transition states except via the declared FSM
- Approvals are immutable once submitted
- Self-approval is rejected by default

---

### Treasury Service

**Purpose:** Hold and route capital under governance constraints.

**Responsibilities:**
- Wallet management (per-org, segmented by purpose: operational / payroll / escrow / reserve)
- Budget allocation and tracking
- Fund-state machine (`available → allocated → locked → released → consumed | refunded`)
- Escrow positions
- Executing approved payments by emitting Ledger transactions

**Non-responsibilities:**
- Accounting truth (Ledger holds it; Treasury balances are derived)
- Approval state (Workflow)
- Policy decisions (Governance)

**Key endpoints:**
- `POST /treasury/allocate`
- `POST /treasury/lock`
- `POST /treasury/release`
- `POST /escrow/release`
- `GET /wallets/{id}`, `GET /budgets/{id}`

**Critical invariants:**
- No negative balances
- Locked funds cannot be reused for any other purpose
- Every state transition produces exactly one ledger transaction (T-3, see [01-principles.md](./01-principles.md))

---

### Ledger Service

**Purpose:** The accounting truth. Append-only double-entry book.

**Responsibilities:**
- Maintain the chart of accounts
- Accept transactions and write entries (debits + credits) atomically
- Provide balance projections (per account, per period)
- Provide trial-balance and other accounting reports

**Non-responsibilities:**
- Anything except recording the truth and reading it back. The Ledger has no policy logic, no notifications, no callbacks.

**Key endpoints:**
- Internal: `POST /ledger/transactions` (idempotent, called only by Treasury)
- Read: `GET /ledger/accounts/{id}/balance?as_of=...`
- Read: `GET /ledger/transactions/{id}`
- Read: `GET /ledger/reports/trial-balance?org_id=...&as_of=...`

**Critical invariants:**
- Every invariant in the L-* group from [01-principles.md](./01-principles.md)

---

### Governance Engine

**Purpose:** Decide whether financial actions are permitted, given current state and policy.

**Responsibilities:**
- Evaluate composable policies (budget, role, risk, compliance, temporal)
- Produce explainable decisions (`allow / reject / escalate` + reasons + risk score)
- Maintain decision history
- Run continuous monitoring for anomaly signals
- Trigger emergency freezes when configured thresholds are crossed

**Non-responsibilities:**
- Executing decisions — it returns them, others act
- Storing financial state — it queries Treasury / Ledger when needed

**Key endpoints (internal):**
- `POST /governance/evaluate` — synchronous, returns decision in <50ms (PRD §56)
- `GET /governance/decisions/{id}` — explainability
- `POST /governance/policies` — admin

**Critical invariants:**
- Decisions are deterministic for a given input + policy version
- Policy evaluations are fully audited including denied paths

---

### Audit Service

**Purpose:** Immutable, queryable history of everything.

**Responsibilities:**
- Ingest every `audit.*` event from every service
- Maintain a per-tenant hash chain
- Provide investigation queries (by actor, by correlation, by time window, by entity)
- Sign and archive periodic snapshots

**Non-responsibilities:**
- Producing events (only ingests)
- Anything mutable

**Key endpoints (read-only):**
- `GET /audit/events?correlation_id=...`
- `GET /audit/events?actor_id=...&from=...&to=...`
- `GET /audit/integrity?org_id=...` — hash-chain verification
- `GET /audit/lineage/{transaction_id}`

**Critical invariants:**
- A-1, A-2, A-3 from [01-principles.md](./01-principles.md)

---

### Notification Service

**Purpose:** Deliver event-driven notifications across channels.

**Responsibilities:**
- Subscribe to `notification.*` triggers from any service
- Apply delivery preferences and templates
- Send via email, SMS, webhook, in-app
- Track delivery state, retry on transient failure

**Non-responsibilities:**
- Deciding *what* is worth notifying about (other services emit triggers)
- Storing the underlying business events (Audit holds them)

**Critical invariants:**
- Delivery is at-least-once with idempotent rendering (same trigger event produces identical notification body)
- Failed deliveries surface as audit events

---

## Service interaction matrix

| | Auth | Workflow | Treasury | Ledger | Governance | Audit | Notification |
| --- | --- | --- | --- | --- | --- | --- | --- |
| **Auth** | — | sync (verify) | sync (verify) | sync (verify) | sync (verify) | sync (verify) | sync (verify) |
| **Workflow** | sync | — | async cmd | — | sync | async event | async event |
| **Treasury** | sync | async event | — | async event | sync | async event | async event |
| **Ledger** | sync | — | — | — | — | async event | — |
| **Governance** | sync | sync (rare) | sync (rare) | sync (read) | — | sync (read) | async event |
| **Audit** | sync | — | — | — | — | — | — |
| **Notification** | sync | — | — | — | — | async event | — |

`sync` = HTTP/RPC; `async event` = published to the event bus.

## See also

- [05-events.md](./05-events.md) — full event catalog and contracts
- [13-api-standards.md](./13-api-standards.md) — how all of these services format requests and responses
