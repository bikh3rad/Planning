# trustC — Architecture & Planning Documentation

This directory is the engineering-oriented companion to [`../prd.md`](../prd.md). The PRD describes **what** trustC must do; these docs describe **how** we'll build it.

## How to read this

If you have 10 minutes, read:

1. [00 — System overview](./architecture/00-overview.md)
2. [01 — Principles & invariants](./architecture/01-principles.md)
3. [15 — Roadmap](./architecture/15-roadmap.md)

If you're about to design or implement a subsystem, read its dedicated deep-dive (06–10) plus the cross-cutting docs that touch it (events, security, API standards).

## Architecture documents

| # | Document | Read if you're working on… |
| --- | --- | --- |
| 00 | [System overview](./architecture/00-overview.md) | Anything — start here |
| 01 | [Principles & invariants](./architecture/01-principles.md) | Anything — non-negotiables |
| 02 | [Domains & bounded contexts](./architecture/02-domains.md) | Service boundaries, ownership |
| 03 | [Service catalog](./architecture/03-services.md) | Any service |
| 04 | [Data model](./architecture/04-data-model.md) | Schemas, accounts, persistence |
| 05 | [Events & event sourcing](./architecture/05-events.md) | Anything that emits or consumes events |
| 06 | [Ledger service](./architecture/06-ledger.md) | Accounting, balances, immutability |
| 07 | [Treasury service](./architecture/07-treasury.md) | Wallets, allocations, locks |
| 08 | [Workflow engine](./architecture/08-workflow.md) | Approvals, state machines |
| 09 | [Governance engine](./architecture/09-governance.md) | Policies, risk, decisions |
| 10 | [Escrow](./architecture/10-escrow.md) | Conditional release flows |
| 11 | [Audit & observability](./architecture/11-audit-observability.md) | Audit trail, tracing, metrics |
| 12 | [Security & threat model](./architecture/12-security.md) | RBAC, multi-tenancy, attack surface |
| 13 | [API design standards](./architecture/13-api-standards.md) | Any HTTP API |
| 14 | [Tech stack & repo structure](./architecture/14-tech-stack.md) | Setup, tooling, deployment |
| 15 | [Delivery roadmap](./architecture/15-roadmap.md) | Sequencing, milestones |
| 16 | [Non-functional requirements](./architecture/16-non-functional.md) | Performance, scale, DR, retention |
| 17 | [UI structure](./architecture/17-ui-structure.md) | Client surfaces, IA per persona, design language |

## Architecture Decision Records

Decisions that close off other reasonable options live as ADRs:

- [ADR index & template](./adr/README.md)
- [0001 — Monorepo tooling](./adr/0001-monorepo-tooling.md) — superseded by 0005
- [0002 — Event sourcing vs outbox pattern](./adr/0002-event-sourcing-vs-outbox.md)
- [0003 — NATS JetStream vs Kafka](./adr/0003-messaging-nats-vs-kafka.md)
- [0004 — Database-level immutability enforcement](./adr/0004-database-immutability-enforcement.md)
- [0005 — Go workspace & build tooling](./adr/0005-go-workspace-and-build.md)

## Conventions

- All numeric prefixes are stable — other docs link to specific files. Don't renumber.
- Mermaid for diagrams. Tables for matrices. Bullet lists for reference material.
- Cite the PRD as `(PRD §N)`.
- See [`../CLAUDE.md`](../CLAUDE.md) for AI-agent-specific guidance.
