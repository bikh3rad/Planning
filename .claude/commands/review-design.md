# review-design

Review a proposed design or feature against the trustC hard architectural rules and the current roadmap phase.

## What this skill does

Given a design description (or a draft doc path in `$ARGUMENTS`), produce a structured review covering:

1. **Invariant compliance** — check each of the six hard rules from `CLAUDE.md`:
   - Append-only ledger (no UPDATE/DELETE on `ledger_entries`)
   - Double-entry (sum debits == sum credits)
   - Treasury isolation (no direct balance mutation by operational actors)
   - Operational context (`actor_id`, `workflow_id`, `business_reason`, idempotency key on every financial action)
   - Event-driven (state transitions originate from events; events are immutable and signed)
   - Multi-tenant (`organization_id` on every row; no cross-tenant reads)

   For each rule: **Pass / Fail / N/A** + one-line reasoning. A Fail means the design needs rework or an ADR.

2. **Roadmap fit** — read `docs/architecture/15-roadmap.md` and determine which phase this design belongs to. Flag if it pulls in v2+ material that was explicitly deferred.

3. **ADR requirement** — does this decision close off another reasonable option? If yes, state that an ADR is required before the design lands.

4. **Per-feature engineering checklist** (from `15-roadmap.md` §Per-feature engineering review):
   - Security review covered?
   - Financial integrity / invariant tests addressed?
   - Concurrency reasoning present?
   - Audit events specified?
   - Scaling to 10x considered?

5. **Verdict**: Ready to doc | Needs rework | Needs ADR first | Out of scope for v1

## Usage

```
/review-design
/review-design "proposal: use a saga pattern for cross-service refunds"
/review-design docs/architecture/10-escrow.md
```
