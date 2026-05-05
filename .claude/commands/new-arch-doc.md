# new-arch-doc

Create a new architecture document under `docs/architecture/` for the trustC planning repo.

## Steps

1. Read `docs/README.md` to find the current highest doc number and the existing topics — avoid duplicating covered ground.
2. Determine the next number `NN` (continue the existing sequence).
3. Ask the user (if not already provided in `$ARGUMENTS`) for the topic/concern this doc will cover.
4. Create `docs/architecture/NN-<topic>.md` with this structure:
   - **Opening paragraph** (one paragraph: what this doc covers, who should read it).
   - Sections appropriate to the subsystem (context, responsibilities, data model, invariants, interfaces, open questions).
   - At least one Mermaid diagram if the doc covers a service or flow.
   - Links to related docs using relative paths.
   - PRD citations in the form `(PRD §N)` where relevant.
5. Add a row to the table in `docs/README.md`:
   `| NN | [Topic](./architecture/NN-<topic>.md) | <one-line "read if you're working on…"> |`

## Style rules (from CLAUDE.md)

- One concern per file. If the draft covers two subsystems, split it.
- Prefer Mermaid over ASCII art.
- Use tables for ownership / responsibility matrices.
- Keep prose short — bullet lists beat paragraphs for technical reference docs.
- Cross-link with relative paths, not absolute URLs.

## Usage

```
/new-arch-doc
/new-arch-doc "multi-currency support"
```
