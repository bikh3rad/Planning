# ADR 0004 — Database-level immutability enforcement

**Status:** Accepted
**Date:** 2026-05-02
**Deciders:** Solution architecture

## Context

The PRD is unambiguous: `ledger_entry` and `audit_event` (and several other tables) must be **append-only** (PRD §5.1, §11, §24, §44). Updates and deletes are forbidden.

The question is: where do we enforce this?

Forces:

- "Application-level" enforcement (just don't write the code that updates/deletes) is the cheapest. It is also useless if anyone gains direct DB access — even a friendly engineer running a "quick fix" script.
- DB-level enforcement is the strongest guarantee but adds operational complexity (different roles, careful migration patterns).
- We will have multiple ways to access the DB over the lifetime of the system: the application, migration tools, ops debugging sessions, BI tools, vendors. All of them must be incapable of mutating immutable data.
- A future regulator audit will ask "prove your ledger cannot be edited." A code reading is not a proof; a DB grant matrix is.

## Decision

Enforce immutability with **three layers of defense**:

### Layer 1 — DB grants

Two Postgres roles:

- `application_role` — what every service connects as for normal traffic. Grants: `INSERT, SELECT` on append-only tables. **No `UPDATE`, no `DELETE`.**
- `migrator_role` — used only by the migration tool, ephemeral connections only. Grants: full DDL but explicit denial on `DELETE FROM ledger_entry` etc.

```sql
-- example grants
REVOKE ALL ON ledger_entry, ledger_transaction, audit_event, approval, governance_decision FROM application_role;
GRANT INSERT, SELECT ON ledger_entry, ledger_transaction, audit_event, approval, governance_decision TO application_role;
```

### Layer 2 — Triggers (defense in depth)

Even if grants are misconfigured, triggers raise:

```sql
CREATE OR REPLACE FUNCTION reject_immutable_change() RETURNS trigger AS $$
BEGIN
  RAISE EXCEPTION 'Table % is append-only; UPDATE/DELETE not permitted', TG_TABLE_NAME;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER ledger_entry_no_update BEFORE UPDATE ON ledger_entry
  FOR EACH ROW EXECUTE FUNCTION reject_immutable_change();
CREATE TRIGGER ledger_entry_no_delete BEFORE DELETE ON ledger_entry
  FOR EACH ROW EXECUTE FUNCTION reject_immutable_change();
```

Triggers fire even for `migrator_role` and superuser, which is what we want.

### Layer 3 — Hash chain

Tampering that bypasses both layers (e.g. someone with raw filesystem access to the WAL) is detected by the hash chain (see [docs/architecture/06-ledger.md](../architecture/06-ledger.md#hash-chain) and [docs/architecture/11-audit-observability.md](../architecture/11-audit-observability.md#hash-chain)).

## Alternatives considered

- **Application-only enforcement** — repository pattern that exposes only `insert`. Cheapest, but a bash one-liner can defeat it. **Reject** — provides no real assurance.

- **Trigger-only enforcement** (no role separation) — works, but a superuser session bypasses triggers (`ALTER TABLE ... DISABLE TRIGGER` exists). **Reject** as sole defense; **adopt as one of three layers**.

- **Separate database for ledger** with extreme lockdown — eliminates the cross-table grant complexity. But: distributed transactions across DBs are nasty; complicates operations; not worth it. **Reject**.

- **Blockchain-anchored ledger** — periodically anchor the hash chain head into a public chain for tamper-evidence at the global level. Interesting but **out of scope for v1**; the architecture already supports this as an extension (the hash chain head is a small artifact).

- **Append-only databases** (e.g. Datomic, immudb, QLDB) — purpose-built for this. But: introduces a new DB engine; tooling and operational maturity vary; we already have Postgres expertise; gives up familiar features. **Reject** for v1; reconsider only if Postgres-level enforcement proves insufficient.

## Consequences

### Positive

- Three independent failure modes required to actually mutate immutable data: misconfigured grants AND missing trigger AND undetected hash-chain mismatch
- Provable to auditors via SQL: `\dp ledger_entry` shows the grants
- No special application code needed — the DB rejects bad operations
- Clear separation between application and migration responsibilities

### Negative

- Migrations on append-only tables become awkward: adding a column is fine, but anything that requires backfilling values into existing rows is impossible. Solution: design schemas with future-proof columns (nullable additions only); use compensating-row patterns for corrections.
- Triggers add a small per-write cost (negligible at our throughput)
- Two database roles add deployment complexity (rotation, secret management)
- Some ORM tools assume `UPDATE` works; we use a thin query layer (kysely) that doesn't

### Migration discipline

For ledger / audit tables specifically:

- New columns: must be `NULL` or have a constant default
- No `DROP COLUMN`, no `ALTER COLUMN ... TYPE` that requires rewrite, no `RENAME COLUMN` of in-use columns
- New constraints: only addable in `NOT VALID` state for existing rows, then validated forward

This is restrictive on purpose. Schema evolution requires more thought than for normal tables; that's the point.

### Revisit when

- A schema change need genuinely cannot be expressed within these rules (so far we have no example)
- We outgrow Postgres for the Ledger and consider a purpose-built immutable store

## Related

- [docs/architecture/01-principles.md](../architecture/01-principles.md) — Ledger invariants
- [docs/architecture/06-ledger.md](../architecture/06-ledger.md) — append-only enforcement section
- [docs/architecture/11-audit-observability.md](../architecture/11-audit-observability.md) — hash chain
