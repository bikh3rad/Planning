# 12 — Security & Threat Model

> RBAC, multi-tenancy enforcement, defensive controls, and the explicit threat model. Sources: PRD §11, §32–§33, §45, §51, §57–§58, §60.

## Threat model

We design against four categories. For each, the goal is **structural impossibility**, not detection-after-the-fact.

### T1 — Internal fraud

Examples: fake invoices, unauthorized approvals, supplier collusion.

Defenses:
- Multi-step approvals required for any spend (no single-actor execution path)
- Self-approval default-denied
- Multi-sig for high-value or off-policy actions
- Continuous behavioral monitoring (Governance) detects approver-vendor patterns
- Append-only approval signatures provide non-repudiation

### T2 — Workflow bypass

Examples: direct treasury mutation, calling internal services to skip approval.

Defenses:
- Treasury has no "execute payment" RPC — only consumes Workflow-issued events
- Internal services authenticate with mTLS and signed event envelopes
- Governance gates every state transition; bypassing it makes the action invisible to Treasury
- Service-to-service requests carry the originating actor's identity (not just the calling service's)

### T3 — Ledger corruption

Examples: editing past entries, deleting incriminating transactions, schema-level rewrites.

Defenses:
- DB grants block `UPDATE` and `DELETE` on ledger tables (see [ADR 0004](../adr/0004-database-immutability-enforcement.md))
- Hash chain detects tampering even if grants are bypassed
- Periodic verification + on-demand verification API
- Daily signed snapshots in write-once cold storage
- Migrations on ledger tables require change review and replay testing

### T4 — Replay attacks

Examples: duplicate payment execution, replayed approval signatures.

Defenses:
- Idempotency keys on every command
- Approval signatures bind to `(expense_id, step_index, decision, timestamp)`
- Events deduplicated on `event_id` at every consumer
- Time-windowed nonces on signed admin actions

---

## Identity & permissions

### Authentication

- **Users:** OIDC against the org's IdP. Session tokens are short-lived (15 min) with refresh.
- **Service-to-service:** mTLS. Every service has a key pair issued by a private CA; rotation is automatic every 90 days.
- **Event signing:** each service holds an Ed25519 signing key for events. Verifiable by any consumer using published JWKS.

### Authorization model

Two layers:

1. **RBAC** (Identity domain) — coarse-grained: which roles exist, who has them, with effective windows.
2. **Policy-Based Access Control** (Governance) — fine-grained: given identity claims + business context, what is permitted?

### Roles (defaults)

| Role | Default permissions |
| --- | --- |
| `investor` | Approve budgets, view all reports, trigger emergency freeze |
| `founder` | Submit expenses, manage cost centers |
| `finance_operator` | Process workflows, manage suppliers, approve up to threshold |
| `auditor` | Read-only across all data + audit query API |
| `supplier` | Upload invoices, view their own escrow positions |
| `employee` | Submit personal expense requests, view own history |
| `admin` | User management only — **not** financial actions |

### Forbidden permissions

These cannot be granted to any role through any UI or API:

- Direct ledger mutation
- Bypassing policies (no `--force` flag exists)
- Deleting audit events
- Cross-organization read or write
- Modifying own role assignment

`admin` is **not** a superuser. PRD §58 is explicit on this. Admins manage actors and their roles, nothing else.

---

## Multi-tenancy

### Isolation requirements (PRD §51)

- Tenant data isolation — no row may be read by an actor whose JWT does not carry the matching `organization_id`
- Tenant-specific policies — Governance bundles are per-org
- Tenant-specific treasury — wallets, accounts, ledger entries are all org-scoped

### Enforcement

Three layers:

1. **JWT claim** — every authenticated request carries `organization_id`. The API gateway rejects requests without it.
2. **Postgres Row-Level Security (RLS)** — every table has an RLS policy: `USING (organization_id = current_setting('app.current_org')::uuid)`. The application sets `app.current_org` per connection from the JWT before any query.
3. **Cross-tenant reconciliation job** — a periodic job samples random reads to verify no cross-tenant leaks; failures trigger an incident.

### Forbidden

- Cross-tenant joins of any kind
- "Internal" admin endpoints that bypass `app.current_org`
- Any ETL job that materializes data without preserving tenant scope

---

## Defensive controls

### Idempotency

Required on every command:

- Header: `Idempotency-Key`
- Stored on `(organization_id, idempotency_key)` unique constraint at the receiving table
- Retry returns the original result (200) without re-executing

### Approval signing

Critical approvals carry Ed25519 signatures:

```
signature = sign(approver_signing_key, canonical(
  expense_request_id, step_index, decision, reason_hash, timestamp
))
```

Verified at write time. Replayed signatures fail because timestamps differ; reused timestamps are caught by the idempotency constraint.

### Immutable audit logs

`REVOKE DELETE ON audit_event FROM application_role`. Operationally, only the migrator role can drop tables, and the migration policy forbids dropping audit tables ever.

### Sensitive operations

Per PRD §57, these require multi-step approval, signed actions, and elevated authorization:

- Emergency freeze / unfreeze
- Policy bundle publication
- Adding a new wallet segment
- Adding an organization
- Onboarding / offboarding investor-tier actors

### Rate limiting

At the API gateway, per `(organization_id, actor_id)`:

- Default: 100 RPS
- Sensitive endpoints (admin, governance config): 10 RPS
- Audit query API: 30 RPS, with daily quotas

### Encrypted secrets

- App-level secrets in a managed secret store (AWS Secrets Manager / GCP Secret Manager / Vault)
- Database encryption at rest via cloud provider features
- TLS 1.3 for all network traffic
- Field-level encryption for PII (actor email, SSN-equivalents)

---

## Edge cases (must be explicitly handled)

Per PRD §60:

| Case | Handling |
| --- | --- |
| Same payment submitted twice | Idempotency key dedupes; original result returned |
| Payment executed but ledger write failed | Treasury detects via missing `ledger.transaction.committed` event; raises critical alert; manual reconciliation per runbook (do **not** auto-retry the payment) |
| Network retry storm during outage | Idempotency dedupes the API surface; outbox pattern dedupes downstream; circuit breakers on inter-service calls |
| Approval submitted with replayed signature | Idempotency key on `(expense_id, step_index, approver)` rejects duplicate |
| Actor offboarded mid-flow | Existing approvals stand; new approvals from this actor rejected; audit shows the role change |

---

## Security review checklist (per PR)

Same as the engineering review in [01-principles.md](./01-principles.md#engineering-review-checklist-per-feature), with these additional security questions:

1. Does this introduce a new authentication or authorization decision point? If yes, is it gated by Governance?
2. Does this expose any data across organizations?
3. Does this add an attack surface (new public endpoint, new file upload, new admin action)?
4. Are all user-supplied strings validated before being used in queries, paths, or shell commands?
5. Are secrets read from the secret store, not hardcoded or env-vars in source?
6. If this is a sensitive action, does it require multi-step / signed approval?

---

## Out of scope (for v1)

- Hardware security module (HSM) integration — keys live in the cloud KMS
- Customer-managed encryption keys (BYOK)
- SOC 2 Type II audit (the architecture supports it; certification is an organizational milestone, not an architectural one)
- On-chain settlement (PRD section "Final Objective" mentions Chainlink/Kleros — explicitly post-v1)

## See also

- [01-principles.md](./01-principles.md) — invariants
- [09-governance.md](./09-governance.md) — runtime enforcement
- [11-audit-observability.md](./11-audit-observability.md) — what audit captures
