# 14 — Tech Stack & Repository Structure

> Stack choices and how the implementation repo will be laid out. Sources: PRD §17, §25, §26, §55.

## Stack at a glance

| Layer | Choice | Why |
| --- | --- | --- |
| Language | TypeScript (strict) | PRD §25; strong typing critical for financial code |
| Runtime | Node.js LTS | PRD §25; sufficient for I/O-bound services, large ecosystem |
| Framework | NestJS | DI, modules align with DDD boundaries, mature |
| RDBMS | PostgreSQL 16 | RLS, advisory locks, robust constraint system |
| Cache / locks | Redis 7 | Idempotency cache, rate limiting, short-lived state |
| Messaging | NATS JetStream | See [ADR 0003](../adr/0003-messaging-nats-vs-kafka.md) |
| Containerization | Docker | Standard |
| Orchestration | Kubernetes | PRD §55 |
| IaC | Terraform | PRD §25 |
| CI | GitHub Actions | Matches the GitHub-hosted repo |
| Observability | OpenTelemetry → Grafana / Tempo / Loki / Prometheus | Open, vendor-neutral |
| Secrets | Cloud-native (AWS Secrets Manager / Vault) | Off-disk |

Rationale for the deltas from "default startup stack":

- **NestJS over Express/Fastify**: the explicit module / DI structure matches the bounded-context architecture; controllers/providers/middleware separation enforces the cleanliness rules in PRD §17.
- **NATS JetStream over Kafka**: target throughput (1M ledger entries/day ≈ 12/s) is well within NATS, operationally simpler for a small team. See ADR 0003.
- **PostgreSQL only, no MongoDB / DynamoDB**: financial workloads need ACID, foreign keys, and DB-level constraints (notably for [ADR 0004](../adr/0004-database-immutability-enforcement.md)).

## Repository structure

When implementation begins, the code lives in a separate repo (`trustc-platform`), not this planning repo. Proposed layout:

```
trustc-platform/
├── apps/
│   ├── auth-service/
│   ├── workflow-service/
│   ├── treasury-service/
│   ├── ledger-service/
│   ├── governance-service/
│   ├── audit-service/
│   ├── notification-service/
│   └── api-gateway/
├── libs/
│   ├── event-contracts/        # shared event types & validators
│   ├── api-contracts/          # shared HTTP types
│   ├── crypto/                 # signing, hashing, key management
│   ├── observability/          # OTel setup, log helpers
│   ├── postgres/               # connection, RLS helpers, migrations runner
│   ├── messaging/              # NATS client wrapper, outbox relay
│   ├── governance-client/      # SDK to call Governance synchronously
│   ├── auth-client/            # SDK to verify JWTs / fetch claims
│   └── testing/                # invariant tests, fixtures, builders
├── infrastructure/
│   ├── terraform/              # cloud infra
│   ├── kubernetes/             # service manifests, helm charts
│   └── docker/                 # local dev compose
├── tools/
│   ├── codegen/                # event schema → TS types
│   ├── policy-cli/             # author + validate governance policies
│   └── replay/                 # event replay for testing
└── docs/                       # implementation-level docs (operational, runbooks)
```

Rules:

- **No shared business logic in `libs/`.** Only protocol-level utilities (event contracts, crypto, transport). Cross-domain types and code are forbidden — that would re-introduce the "shared god service" anti-pattern (PRD §64).
- **Each app owns its DB schema and migrations.** Migrations live in `apps/<service>/migrations/`. No cross-schema reads at the application layer.
- **Tooling for monorepo: `pnpm` workspaces + `turbo`.** See [ADR 0001](../adr/0001-monorepo-tooling.md).

## Per-service folder structure (DDD)

Each service follows the same internal layout, mirroring PRD §26:

```
apps/<service>/src/
├── domain/                     # entities, value objects, domain services
├── application/                # use cases, commands, queries
├── infrastructure/
│   ├── postgres/               # repositories, migrations
│   ├── messaging/              # event publishers, consumers
│   └── http/                   # controllers, DTOs, validators
├── policies/                   # service-local invariant guards
├── events/                     # event payload definitions
└── main.ts
```

## Build & deploy

- Single CI workflow per app: lint → typecheck → unit → integration → invariant tests → build image → push
- Per-PR: lint + typecheck + unit + invariant tests
- Per-merge to main: full pipeline + deploy to dev
- Per-tag (vN.N.N): deploy to staging → manual gate → production
- Rolling deploys, never blue/green for stateful services (Postgres requires careful migration sequencing)

## Local dev

- `docker compose up` brings up Postgres, Redis, NATS, and a minimal subset of services
- Each service has a `dev` script that watches & rebuilds
- Integration tests run against the compose stack
- A `seed` script populates a fixture organization with sample data

## Migration policy

- Every PR that touches a schema includes the migration
- Migrations are reversible (down provided), even if rolling back is operationally expensive
- Ledger / audit table migrations require explicit reviewer + replay test
- See [06-ledger.md](./06-ledger.md#migration-safety) for ledger-specific constraints

## Testing strategy

| Layer | Tool | What it covers |
| --- | --- | --- |
| Unit | Vitest / Jest | Pure domain logic |
| Property-based | fast-check | Invariants (ledger balance, FSM transitions) |
| Integration | Vitest + Testcontainers | Service against real Postgres / NATS |
| Invariant | Dedicated suite, runs in CI | Cross-cutting invariants from [01-principles.md](./01-principles.md) |
| End-to-end | Playwright (when UI exists) | Top-level flows |
| Replay | Custom runner | Rebuild projections from event log, assert match |

Coverage gates: 80% line, 100% on policy aggregator and ledger validator.

## What is NOT in the stack (and why)

- **An ORM beyond a query builder.** ORMs hide query behavior; for financial code we want explicit SQL. Use a thin layer (kysely or similar).
- **GraphQL.** Adds complexity; resource-oriented REST suffices for this domain.
- **Event sourcing framework (e.g. EventStoreDB).** Outbox-on-Postgres is simpler; see [ADR 0002](../adr/0002-event-sourcing-vs-outbox.md).
- **A centralized "shared services" library.** Forbidden by domain isolation rules.

## See also

- [02-domains.md](./02-domains.md) — boundaries the repo structure enforces
- [15-roadmap.md](./15-roadmap.md) — what gets built first
- [ADR 0001](../adr/0001-monorepo-tooling.md), [ADR 0003](../adr/0003-messaging-nats-vs-kafka.md)
