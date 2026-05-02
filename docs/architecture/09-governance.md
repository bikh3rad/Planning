# 09 — Governance Engine

> The decision layer. Returns explainable allow / reject / escalate verdicts on every financial action. Sources: PRD §6.5, §29, §75, §82–§83, §146–§158, §183.

## What it does

- Evaluates composable policies against transaction requests
- Returns decisions with structured reasons and a risk score
- Persists every decision (allowed and denied) for audit and replay
- Monitors continuously for anomalies and emits `governance.alert.*` events
- Triggers emergency freezes when thresholds are crossed
- Manages exception requests with controlled lifetimes

## What it does not do

- Execute decisions (other services act on them)
- Hold financial state (Treasury, Ledger)
- Run the approval chain (Workflow)

## Decision shape

Every Governance evaluation returns:

```json
{
  "decision_id": "uuid",
  "decision": "allow | reject | escalate",
  "reasons": [
    { "evaluator": "budget", "code": "within_limit", "detail": "remaining 32000/50000" },
    { "evaluator": "supplier", "code": "verified", "detail": "supplier_id=..." },
    { "evaluator": "risk", "code": "low_score", "detail": { "score": 0.12 } }
  ],
  "risk_score": 0.12,
  "policy_version": "2026-05-01.7",
  "evaluated_at": "2026-05-02T10:15:30.123Z",
  "additional_approvers_required": []
}
```

`escalate` includes `additional_approvers_required` — Workflow will extend the approval chain accordingly.

## Policy model

A policy bundle is a versioned collection of rules. Each rule is structured (YAML or JSON) and evaluated by a typed evaluator.

```yaml
policy_bundle:
  version: 2026-05-01.7
  organization_id: org_8a4f...
  rules:
    - id: payroll_only_for_employees
      evaluator: budget
      conditions:
        budget_type: payroll
        receiver_type: employee
      action: allow
      otherwise: reject

    - id: large_amount_requires_investor
      evaluator: threshold
      conditions:
        amount_gte: 100000
      action: escalate
      additional_approvers:
        - role: investor

    - id: vendor_must_be_verified
      evaluator: supplier
      conditions:
        supplier_status: verified
      action: allow
      otherwise: reject

    - id: night_time_review
      evaluator: temporal
      conditions:
        time_local_after: "22:00"
      action: escalate
      additional_approvers:
        - role: finance_director

    - id: high_risk_block
      evaluator: risk
      conditions:
        risk_score_gte: 0.85
      action: escalate
      additional_approvers:
        - role: investor
        - role: compliance
```

## Evaluator catalog

| Evaluator | Reads | Decides on |
| --- | --- | --- |
| `budget` | Treasury budget state | Budget-type / receiver-type compatibility, remaining headroom |
| `threshold` | Request amount | Amount-based escalation rules |
| `role` | Identity (actor roles) | Whether this actor can perform this action |
| `supplier` | Treasury (supplier verification state) | Supplier eligibility |
| `temporal` | Wall clock | Time-of-day, weekend, quarter-end rules |
| `risk` | Risk score (computed inline or from cached features) | Risk-based escalation |
| `compliance` | Governance config (sanctions lists, restricted regions) | Compliance gates |
| `velocity` | Recent decision history | Frequency-based anomaly detection |
| `relationship` | Audit lineage | Approver-vendor collusion patterns (PRD §175) |

Evaluators are pluggable; new ones can be added without changing the aggregator.

## Aggregation

The decision aggregator combines individual evaluator outputs:

```
final_decision =
  if any evaluator returns reject → reject
  else if any evaluator returns escalate → escalate (union of additional_approvers)
  else → allow
```

This is an explicit **deny-overrides** strategy. Conservative by design.

## Risk scoring

Per PRD §76, every transaction may receive a risk score in `[0.0, 1.0]`. Inputs:

- Transaction amount (relative to historical)
- Vendor history
- Approval pattern
- Budget variance
- Frequency
- Geo / IP context

In v1, risk scoring is **rules-based** (a configurable weighted sum). The architecture leaves room for an ML model in a later phase (PRD §77, §188 — explicitly out of scope for v1).

## Continuous (runtime) governance

Beyond per-request evaluation, Governance runs background monitors (PRD §146):

- Liquidity pressure → `governance.alert.liquidity_low`
- Budget drift → `governance.alert.budget_drift`
- Velocity spike → `governance.alert.velocity_anomaly`
- Repeated rejections from same actor → `governance.alert.actor_anomaly`
- Hash-chain mismatch in Audit/Ledger → `governance.alert.integrity_break`

These alerts are consumed by Notification (for humans) and may auto-trigger emergency freeze.

## Emergency freeze

Per PRD §84, certain conditions trigger an emergency freeze:

| Trigger | Action |
| --- | --- |
| Fraud signal score > 0.95 | Freeze the suspected wallet, alert compliance |
| Hash-chain integrity failure | Freeze write side of Ledger and Treasury, alert ops |
| Manual investor freeze | Freeze entire org's spend, require multi-sig to unfreeze |

When frozen:
- Treasury rejects all `release` and `lock` commands
- Workflow continues to accept new requests (state drafts) but cannot execute them
- The freeze itself is an audit event with full context

## Emergency overrides

PRD §157 permits controlled overrides in extreme situations. Mandatory constraints:

- Multi-signature approval (minimum 2 distinct roles, not the same actor)
- Time-bounded: overrides expire automatically (default 24 hours)
- Every override action emits a high-priority audit event
- Override usage triggers a post-incident review workflow

There is no admin UI button to bypass — overrides go through their own Governance-evaluated workflow.

## Exception management

Per PRD §183, exceptions are controlled and temporary:

- An exception is a scoped, time-bounded relaxation of one rule
- Requires explicit policy granting the exception, not just an approval
- Auto-reviewed at expiry — does not silently extend
- Cannot exempt an entire policy bundle — only individual rules

## Explainability

Every decision must be explainable to a human (PRD §158). The decision record stores:

- The exact policy bundle version used
- The full input snapshot
- Per-evaluator outputs and reasons
- The aggregated decision

A `GET /governance/decisions/{id}/explanation` endpoint returns this in human-readable form.

## API surface

Internal-only (no public exposure):

| Method | Path | Purpose |
| --- | --- | --- |
| `POST` | `/governance/evaluate` | Synchronous evaluation, < 50 ms |
| `GET`  | `/governance/decisions/{id}` | Retrieve a past decision |
| `GET`  | `/governance/decisions/{id}/explanation` | Human-readable explanation |
| `POST` | `/governance/policies` | Admin: publish a new policy bundle version |
| `POST` | `/governance/freeze` | Admin: emergency freeze (multi-sig) |
| `POST` | `/governance/exceptions` | Admin: grant an exception |

## Performance targets

| Operation | Target (PRD §56) |
| --- | --- |
| `evaluate` | < 50 ms p99 |
| Decision lookup | < 30 ms p99 |

## Determinism

Same `(input, policy_version)` must produce the same `(decision, reasons, risk_score)` always. Evaluators that need wall-clock time accept it as input rather than reading it themselves, so replay produces identical results.

## See also

- [08-workflow.md](./08-workflow.md) — primary caller
- [01-principles.md](./01-principles.md) — invariants enforced by Governance
- [11-audit-observability.md](./11-audit-observability.md) — every decision is audited
