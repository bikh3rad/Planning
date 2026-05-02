# trustC — Planning & Architecture

This repository holds the planning and architecture documentation for **trustC**, a programmable financial operating system.

There is no application code here yet. When implementation starts it will live in a separate monorepo (see [`docs/architecture/14-tech-stack.md`](./docs/architecture/14-tech-stack.md)).

## What's in this repo

| Path | Purpose |
| --- | --- |
| [`prd.md`](./prd.md) | Full Product Requirements Document (~5,700 lines, 218 sections). The source of intent. |
| [`docs/`](./docs/README.md) | Implementation-oriented architecture documentation, distilled from the PRD |
| [`docs/architecture/`](./docs/architecture/) | One file per architectural concern (services, data, events, security, etc.) |
| [`docs/adr/`](./docs/adr/) | Architecture Decision Records — decisions that close off other reasonable options |
| [`CLAUDE.md`](./CLAUDE.md) | Repo-level guidance for Claude Code sessions |

## Where to start

- **Stakeholder / new joiner:** [`docs/README.md`](./docs/README.md) → [`docs/architecture/00-overview.md`](./docs/architecture/00-overview.md)
- **Engineer about to build a service:** read [`00-overview`](./docs/architecture/00-overview.md), [`01-principles`](./docs/architecture/01-principles.md), and the deep-dive for that service
- **Reviewing a design proposal:** check it against [`01-principles.md`](./docs/architecture/01-principles.md) — invariants are non-negotiable
- **Curious why a decision was made:** [`docs/adr/`](./docs/adr/README.md)

## Status

| | Status |
| --- | --- |
| PRD | Drafted, v1.0 |
| Architecture documentation | Drafted (this PR) |
| Implementation | Not started |
| Implementation repo | To be created (`trustc-platform`) |

## Contributing to the docs

- New architecture file: `docs/architecture/NN-topic.md`, continue the existing numbering, add to the index
- New decision: copy the template in [`docs/adr/README.md`](./docs/adr/README.md) into `docs/adr/NNNN-slug.md`
- See [`CLAUDE.md`](./CLAUDE.md) for AI-agent-specific guidance
