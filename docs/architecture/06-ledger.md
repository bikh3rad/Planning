# 06 — Ledger Service

> The most load-bearing subsystem in trustC. Append-only, double-entry, deterministic, replay-safe. Sources: PRD §6.4, §27, §71, §72, §181.

## Responsibilities

- Maintain the chart of accounts per organization
- Accept transactions and write entries (debits + credits) atomically
- Enforce that `sum(debits) == sum(credits)` per transaction
- Provide point-in-time balance projections per account
- Provide trial-balance and other accounting reports
- Maintain a hash chain over entries per organization

## Non-responsibilities

- Approving anything
- Executing payments
- Notifying anyone
- Calling out to other services (the Ledger has no outbound HTTP clients)

## Account model

Account types follow standard double-entry accounting (PRD §71):

| Type | Normal balance | Examples |
| --- | --- | --- |
| `asset` | Debit | `cash`, `bank_main`, `accounts_receivable` |
| `liability` | Credit | `accounts_payable`, `taxes_payable` |
| `equity` | Credit | `paid_in_capital`, `retained_earnings` |
| `expense` | Debit | `marketing_expense`, `payroll_expense`, `software_expense` |
| `revenue` | Credit | `service_revenue` |
| `escrow` | Credit | `escrow_liability_<vendor>` |

Each organization has its own chart of accounts (CoA). Account `code` is org-scoped and unique within the org.

## Transaction model

A **transaction** is the atomic unit. It contains 2..N entries, summing to a balanced debit/credit pair.

```json
{
  "transaction_id": "uuid",
  "organization_id": "uuid",
  "correlation_id": "uuid",
  "business_reason": "Vendor payment for Q2 design work",
  "source_event_id": "uuid",
  "idempotency_key": "treasury:exec:expense_8a4f...",
  "posted_at": "2026-05-02T10:15:30.123Z",
  "entries": [
    { "account_code": "1000-cash", "side": "credit", "amount": 50000, "currency": "USD" },
    { "account_code": "5200-vendor-payments", "side": "debit", "amount": 50000, "currency": "USD" }
  ]
}
```

### Validation rules (enforced before insert)

1. `len(entries) >= 2`
2. `sum(debits) == sum(credits)` per currency (cross-currency txns are forbidden — see [04-data-model.md#currency-handling](./04-data-model.md#currency-handling))
3. All `account_id`s belong to the same `organization_id`
4. `business_reason` is non-empty
5. `source_event_id` references an existing inbound event
6. `(organization_id, idempotency_key)` does not already exist

A failed validation returns an error to the caller and emits **no entries** — partial writes are impossible because the entire write is one DB transaction.

## Append-only enforcement

Three layers of defense:

1. **DB grants:** application role lacks `UPDATE` and `DELETE` on `ledger_entry` and `ledger_transaction`. Only the migrator role can modify schema.
2. **Triggers:** `BEFORE UPDATE` and `BEFORE DELETE` triggers on both tables raise an exception, in case grants are misconfigured.
3. **Application:** the repository class exposes only `insert`. There is no `update` or `delete` method to call.

Corrections happen via **compensating transactions**: post a new transaction that reverses the original, with `business_reason` referencing the original `transaction_id`.

## Hash chain

Each `ledger_entry` carries `prev_hash` and `entry_hash`:

```
entry_hash = SHA-256(canonical(
  organization_id,
  ledger_transaction_id,
  account_id,
  side,
  amount,
  currency,
  prev_hash
))
```

`prev_hash` is the `entry_hash` of the previous entry **for the same `organization_id`**, ordered by `(created_at, id)`. The first entry per org has `prev_hash = ZERO`.

Verification job runs nightly and on demand:

```sql
-- pseudocode
for each org:
  walk entries in order
  recompute entry_hash from row + prev_hash
  assert match
```

A mismatch is a critical incident. The Ledger will continue serving reads but will reject writes until cleared.

## Idempotency

Every command into the Ledger carries an `idempotency_key`. The Ledger stores it on `ledger_transaction.idempotency_key` with a unique constraint on `(organization_id, idempotency_key)`.

A retry with the same key:
- Returns the original transaction's data (200 OK)
- Does **not** insert duplicate entries
- Does **not** emit a duplicate `ledger.transaction.committed` event (the outbox row has the same dedup key)

## Balance projections

Balances are derived, not stored. Two read paths:

### Real-time (slow but exact)

```sql
SELECT
  CASE WHEN side = 'debit'  THEN amount ELSE -amount END
    * CASE WHEN account_type IN ('asset','expense') THEN 1 ELSE -1 END
  AS signed_amount
FROM ledger_entry
JOIN account ON account.id = ledger_entry.account_id
WHERE account_id = $1
  AND organization_id = $2
  AND created_at <= $3
GROUP BY account_id
```

### Cached (fast, eventually consistent)

A `account_balance_snapshot` table stores `(account_id, as_of, balance)` rows written by a projection process. Reads serve the latest snapshot before the requested timestamp, then add real-time entries forward from there.

Snapshots are recomputable from scratch via replay. They are an optimization, never a source of truth.

## Reports

| Report | Endpoint | Notes |
| --- | --- | --- |
| Account balance | `GET /ledger/accounts/{id}/balance?as_of=...` | Single account at a point in time |
| Trial balance | `GET /ledger/reports/trial-balance?org_id=...&as_of=...` | All accounts; sum of debits must equal sum of credits |
| Transaction lookup | `GET /ledger/transactions/{id}` | Full transaction with entries |
| Lineage | `GET /ledger/transactions/by-correlation/{cid}` | All txns sharing a correlation |
| Period ledger | `GET /ledger/accounts/{id}/entries?from=&to=` | Stream of entries |

All reports are scoped to a single `organization_id` (via JWT claim, enforced by RLS).

## Performance targets

| Operation | Target (PRD §56) | How |
| --- | --- | --- |
| Single-transaction write | < 100 ms p99 | One DB transaction, indexed inserts |
| Account-balance read (cached) | < 50 ms p99 | Snapshot + delta |
| Trial balance for typical org (~10k accounts) | < 2 s p95 | Materialized snapshot, refreshed hourly |

## Failure modes

| Failure | Behavior |
| --- | --- |
| Validation error | 400 to caller, no write, audit event with reason |
| DB connection lost mid-write | Postgres rolls back atomically; outbox stays clean |
| Outbox relay can't publish | Entries are committed; relay retries with backoff; consumers see eventual delivery |
| Hash chain verification fails | Critical alert; Ledger goes read-only until investigated |
| Replay finds discrepancy in projection | Snapshots wiped, projection rebuilt from entries |

## Migration safety

Per PRD §38, financial migrations are high risk. Rules:

- **No destructive migrations on `ledger_entry` or `ledger_transaction` ever.** Add columns, don't remove.
- Schema changes that affect projections require a replay step in the deploy.
- Every migration is reversible (down migration provided), even if rolling back is operationally expensive.
- Audit-compatibility is mandatory: a hash recomputed today against an entry from 2 years ago must match.

## See also

- [04-data-model.md](./04-data-model.md) — schema details
- [07-treasury.md](./07-treasury.md) — the only writer to the Ledger (via events)
- [ADR 0004](../adr/0004-database-immutability-enforcement.md) — append-only enforcement strategy
