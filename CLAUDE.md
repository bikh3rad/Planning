# CLAUDE.md

Guidance for Claude Code sessions working in this repository.

## Repository purpose

This repo holds the **architecture and planning documentation** for **trustC** — a programmable financial operating system. The full product spec lives in [`prd.md`](./prd.md). The distilled, implementation-oriented architecture lives in [`docs/`](./docs/README.md).

There is **no application code in this repo yet**. When implementation starts it will live in a separate monorepo (see [docs/architecture/14-tech-stack.md](./docs/architecture/14-tech-stack.md)).

## What "done" looks like in this repo

This is a planning repository. Changes here should fall into one of:

1. **New / updated architecture docs** under `docs/architecture/` — keep one file per concern, mirror the numbering scheme.
2. **New ADR** under `docs/adr/` — every decision that closes off other reasonable options gets one. Use the template in `docs/adr/README.md`.
3. **PRD updates** — only when the product owner has signed off; the PRD is the source of intent, the architecture docs are how we'll execute against it.

Do **not** add code, build configs, or scaffolding to this repo. When the project moves into implementation, scaffold a fresh repo using the structure described in `docs/architecture/14-tech-stack.md`.

## Reading order for a fresh session

If you've just been dropped into this repo, read in this order before answering substantive questions:

1. [`docs/README.md`](./docs/README.md) — index
2. [`docs/architecture/00-overview.md`](./docs/architecture/00-overview.md) — what trustC is
3. [`docs/architecture/01-principles.md`](./docs/architecture/01-principles.md) — the non-negotiables
4. [`docs/architecture/15-roadmap.md`](./docs/architecture/15-roadmap.md) — what's being built first

The full PRD (`prd.md`, ~5,700 lines) contains a lot of duplicated and aspirational material. Sections 1–90 are the substantive spec; sections 90–218 are roadmap and philosophy. Cite section numbers when relevant but don't treat the whole document as load-bearing.

## Doc style rules

- One concern per file. If a doc is starting to cover two subsystems, split it.
- Lead with a one-paragraph summary of what the doc covers and who should read it.
- Prefer Mermaid diagrams over ASCII art. They render in GitHub.
- Use tables for ownership / responsibility matrices.
- Cross-link with relative paths (`../adr/0002-...md`), not absolute URLs.
- Keep prose short. Bullet lists beat paragraphs for technical reference docs.
- When citing the PRD, use the form `(PRD §27)`.

## Hard architectural rules (don't violate when planning)

These come from the PRD and are load-bearing for everything downstream:

- **Append-only ledger.** No `UPDATE` or `DELETE` on `ledger_entries`. Corrections happen via compensating entries.
- **Double-entry.** Every ledger transaction must satisfy `sum(debits) == sum(credits)`.
- **Treasury isolation.** Operational actors never mutate treasury balances directly; balance changes always flow through events.
- **Operational context.** Every financial action carries `actor_id`, `workflow_id`, `business_reason`, and an idempotency key.
- **Event-driven.** State transitions originate from events; events are immutable and signed.
- **Multi-tenant from day 1.** Every row carries `organization_id`; cross-tenant reads are forbidden.

When proposing a new design, explicitly check it against these. If a proposal needs to bend one, write an ADR.

## Working on docs

- New architecture file: `docs/architecture/NN-topic.md` where `NN` continues the existing numbering. Add an entry to `docs/README.md`.
- New decision: copy `docs/adr/README.md`'s template into `docs/adr/NNNN-slug.md`, status `Proposed`. Add to the ADR index.
- Updates to existing files: keep the original headings stable — other docs link into them.

## Commit & PR conventions

- Branch names: `docs/<topic>` for doc work, `adr/<slug>` for new decisions.
- Commit messages: imperative mood, one logical change per commit.
- PRs should reference any ADRs they depend on or supersede.

## What to ask before doing significant work

- "Is this implementation guidance, or is it product scope?" If product scope, it belongs in the PRD, not in architecture docs.
- "Does this decision close off another reasonable option?" If yes, it needs an ADR before it lands in an architecture doc.
- "Am I about to duplicate something that already exists?" Search `docs/` first.
