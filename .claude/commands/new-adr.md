# new-adr

Create a new Architecture Decision Record for the trustC planning repo.

## Steps

1. Read `docs/adr/README.md` to get the current ADR index and find the next available number (NNNN).
2. Ask the user (if not already provided in `$ARGUMENTS`): title, decision summary, alternatives considered, and deciders.
3. Create `docs/adr/NNNN-<slug>.md` using the exact template from `docs/adr/README.md`. Set status to `Proposed` and date to today.
4. Add a row to the index table in `docs/adr/README.md`:
   `| [NNNN](./NNNN-<slug>.md) | <Title> | Proposed |`
5. Tell the user the file path and remind them to open a branch named `adr/<slug>`.

## Invariants to check before filing

Before creating the file, verify the decision:
- Closes off at least one other reasonable option (if not, it may not need an ADR).
- Does not contradict the six hard architectural rules in `CLAUDE.md` (append-only ledger, double-entry, treasury isolation, operational context, event-driven, multi-tenant). If it does, flag this explicitly — a deviation itself needs an ADR.

## Usage

```
/new-adr
/new-adr "Use Redis for idempotency key storage"
```
