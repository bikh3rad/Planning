# phase-status

Report the current delivery phase status for the trustC project and surface the next concrete actions.

## What this skill does

1. Read `docs/architecture/15-roadmap.md` for the full phase plan and exit criteria.
2. Read `docs/README.md` and scan `docs/architecture/` and `docs/adr/` to assess what has been documented / decided vs. what the roadmap requires.
3. Produce a concise status table:

| Phase | Name | Docs complete | ADRs required | Exit criteria met | Notes |
|-------|------|--------------|---------------|-------------------|-------|
| 0 | Foundation | … | … | … | … |
| 1 | Ledger MVP | … | … | … | … |
| … | … | … | … | … | … |

4. List the **top 3 next actions** — the highest-leverage things to document or decide before implementation starts.

5. Flag any gaps: decisions that appear in architecture docs but lack a backing ADR, or roadmap deliverables that have no corresponding doc yet.

## Notes

- This is a planning repo — "complete" means the doc or ADR exists and is coherent, not that the code is written.
- Do not speculate about implementation status; only assess what is present in this repo.

## Usage

```
/phase-status
```
