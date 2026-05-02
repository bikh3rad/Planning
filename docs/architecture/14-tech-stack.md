# 14 — Tech Stack & Repository Structure

> Stack choices and how the implementation repos will be laid out. Sources: PRD §17, §25, §26, §55.

trustC ships as three independently-deployable artifacts: a Go backend (~7 services + a gateway), a Next.js web application, and a React Native mobile app. Each lives in its own repository, sharing types via a generated contracts package.

## Stack at a glance

### Backend

| Layer | Choice | Why |
| --- | --- | --- |
| Language | Go 1.22+ | Strong static typing, fast builds, simple deployment, mature stdlib |
| HTTP router | `chi` | Idiomatic, light, composable middleware, sticks to `net/http` |
| DB driver | `pgx` (v5) | Postgres-native, fastest in benchmarks, exposes Postgres features cleanly |
| Query layer | `sqlc` | Generates type-safe Go from explicit SQL — no ORM magic |
| Migrations | `golang-migrate` | Up/down migrations, integrates with Postgres, used by every service |
| Messaging client | `nats.go` (JetStream) | Official Go client, durable consumers, replay |
| Crypto | `crypto/ed25519`, `crypto/sha256` | stdlib — no third-party deps for security-critical code |
| Cache / locks | Redis 7 / Valkey | Idempotency cache, rate limiting, short-lived state |
| Build tooling | Single Go module + Taskfile | See [ADR 0005](../adr/0005-go-workspace-and-build.md) |
| Logging | `log/slog` (stdlib) | Structured, zero-dep, Go 1.21+ |
| Observability | OpenTelemetry Go SDK → Grafana / Tempo / Loki / Prometheus | Vendor-neutral |
| Testing | stdlib `testing` + `testcontainers-go` + `pgregory.net/rapid` | Standard idioms; real Postgres / NATS in integration tests |
| Containerization | Docker (multi-stage, distroless final) | Static Go binaries → tiny images |
| Orchestration | Kubernetes | PRD §55 |
| IaC | Terraform | PRD §25 |
| CI | GitHub Actions | Matches the repo host |
| Secrets | Cloud-native (AWS Secrets Manager / Vault) | Off-disk |

### Web application

| Layer | Choice | Why |
| --- | --- | --- |
| Framework | Next.js 14 (App Router) | Mature, RSC, SSR for the operator UI |
| Language | TypeScript (strict) | Type safety; types generated from contracts |
| Data fetching | TanStack Query | Caching, optimistic updates, ideal for read-heavy dashboards |
| UI primitives | Radix + Tailwind, scaffolded via shadcn/ui | Accessible, headless, copy-not-install |
| Auth | OIDC client (`oidc-client-ts`) | Standards-based, works with any OIDC provider |
| Charts | Recharts | Sufficient for runway / burn / utilization views |
| Real-time updates | Server-Sent Events from gateway | Simpler than WebSockets for one-way dashboard updates |
| Build / deploy | Next.js standalone output → Docker → Kubernetes | Same deployment story as backend |

### Mobile application

| Layer | Choice | Why |
| --- | --- | --- |
| Platform | React Native (Expo) | Cross-platform, reuses TS / React skills with the web app |
| Language | TypeScript | Same |
| Auth | OIDC + PKCE via `expo-auth-session` | Mobile-safe OAuth flow |
| Secure storage | `expo-secure-store` (Keychain / Keystore) | Refresh tokens never in plaintext |
| Biometric | `expo-local-authentication` | Required for approval submission per PRD §32 |
| Push | Expo notifications → APNs / FCM | Notification service emits events; push is one delivery channel |
| Real-time | WebSockets to gateway | Approval-queue updates |
| Data fetching | TanStack Query (same as web) | Code reuse |

Alternative considered for mobile: **Flutter**. Rejected for the small team because RN amortises the TypeScript investment from the web side. Revisit if the mobile team grows and consistency of native widgets becomes a higher priority than code reuse.

## Why this stack

Rationale for the deltas from a "default Go startup stack":

- **`chi` over `fiber`/`echo`/`gin`**: chi sticks to `net/http`, has the smallest API surface, and composes middleware cleanly. Fiber uses fasthttp (incompatible with the broader middleware ecosystem). Gin's API encourages monolithic handlers.
- **`sqlc` over an ORM (`gorm`, `ent`)**: financial code requires explicit, reviewable SQL. ORMs hide query behavior; sqlc generates types from the SQL we write. This matches the "no ORM beyond a query builder" rule carried over from ADR 0001's analysis.
- **PostgreSQL only, no MongoDB / DynamoDB**: financial workloads need ACID, foreign keys, DB-level constraints (notably for [ADR 0004](../adr/0004-database-immutability-enforcement.md)) and Postgres RLS for multi-tenancy.
- **NATS JetStream over Kafka**: target throughput (1M ledger entries/day ≈ 12/s) is well within NATS, operationally simpler. See [ADR 0003](../adr/0003-messaging-nats-vs-kafka.md).
- **REST over gRPC for internal calls**: keeps the contract surface uniform with the public API. Revisit if internal call latency becomes a bottleneck.
- **React Native over native iOS + Android**: small team, code reuse with web, sufficient performance for an approval / dashboard app (no graphics-heavy workloads).

## Three repos, not one

| Repo | Contents | Deployed as |
| --- | --- | --- |
| `trustc-platform` | All Go services, infra, contracts source-of-truth | 8 service images (gateway + 7 services) |
| `trustc-web` | Next.js web app | One web image |
| `trustc-mobile` | Expo / React Native app | iOS + Android builds via EAS |

### Why three repos, not one

- Different toolchains (Go vs npm vs Expo / native build pipelines)
- Different release cadences — services deploy continuously; mobile ships through app stores on a slower clock
- Different reviewer pools
- A mixed-language monorepo's CI complexity outweighs the convenience of "one clone"

### Cross-repo contracts

Cross-cutting types (event envelopes, API DTOs) are defined as **protobuf** in `trustc-platform/contracts/`. Generation:

- `buf generate` produces Go code (committed in `trustc-platform/internal/contracts/`)
- A published npm package `@trustc/contracts` (built from the same `.proto` files in CI) gives the web and mobile repos identical types
- `buf breaking` runs in CI on the contracts directory

This avoids the worst pattern (hand-written client types that drift from the server) while keeping each repo independently buildable.

## Backend repository structure (`trustc-platform`)

See [ADR 0005](../adr/0005-go-workspace-and-build.md) for the rationale.

```
trustc-platform/
├── cmd/                          # service entrypoints (one main.go each)
│   ├── auth/
│   ├── workflow/
│   ├── treasury/
│   ├── ledger/
│   ├── governance/
│   ├── audit/
│   ├── notification/
│   └── gateway/
├── services/                     # per-service domain code, mirrors DDD layers
│   ├── workflow/
│   │   ├── domain/               # entities, value objects
│   │   ├── application/          # use cases, commands, queries
│   │   ├── infrastructure/
│   │   │   ├── postgres/         # repositories, sqlc-generated code, migrations
│   │   │   ├── messaging/        # event publishers, consumers, outbox writer
│   │   │   └── http/             # handlers, DTOs
│   │   ├── policies/             # service-local invariant guards
│   │   └── events/               # event payload definitions specific to this service
│   ├── treasury/...
│   └── ...
├── internal/                     # shared infrastructure (NOT business logic)
│   ├── event/                    # envelope schema, canonical-JSON, signing, validation
│   ├── contracts/                # protobuf-generated Go types
│   ├── crypto/                   # ed25519, sha256 helpers
│   ├── pgx/                      # connection helpers, RLS context setter, migration runner
│   ├── nats/                     # JetStream client, outbox relay
│   ├── otel/                     # tracer / meter / logger setup
│   ├── httpx/                    # required-headers middleware, response envelope, error mapping
│   └── testing/                  # invariant test runners, fixtures, builders
├── contracts/                    # .proto files (source of truth)
│   ├── events/
│   └── api/
├── infrastructure/
│   ├── terraform/                # cloud infra
│   ├── kubernetes/               # service manifests, helm charts
│   └── docker/                   # local dev compose
├── tools/
│   ├── codegen/                  # buf + sqlc invocation wrappers
│   ├── policy-cli/               # author + validate governance policies
│   └── replay/                   # event replay for testing
├── docs/                         # implementation-level docs (operational, runbooks)
├── go.mod
├── go.sum
└── Taskfile.yml
```

Rules:

- **No business logic in `internal/`.** Only protocol-level utilities (event contracts, crypto, transport, observability). Cross-domain types are forbidden — that would re-introduce the "shared god service" anti-pattern (PRD §64).
- **Each service owns its DB schema and migrations.** Migrations live in `services/<service>/infrastructure/postgres/migrations/`. No cross-schema reads at the application layer.
- **`internal/` is the language-level boundary.** Go enforces that nothing outside the module can import these packages — the boundary that the TS stack used Turborepo + pnpm strict mode to approximate.

## Web repository structure (`trustc-web`)

```
trustc-web/
├── app/                          # Next.js App Router
│   ├── (auth)/
│   ├── (operator)/expenses/
│   ├── (operator)/budgets/
│   ├── (investor)/dashboard/
│   └── api/                      # BFF routes if needed
├── components/
├── lib/
│   ├── api/                      # generated client from @trustc/contracts
│   ├── auth/                     # OIDC client setup
│   └── query/                    # TanStack Query setup
├── public/
├── package.json
└── next.config.mjs
```

## Mobile repository structure (`trustc-mobile`)

```
trustc-mobile/
├── app/                          # Expo Router file-based routes
│   ├── (auth)/
│   ├── (tabs)/
│   │   ├── approvals/
│   │   ├── expenses/
│   │   └── settings/
│   └── _layout.tsx
├── components/
├── lib/
│   ├── api/                      # generated client from @trustc/contracts
│   ├── auth/                     # OIDC + secure-store
│   └── biometric/                # local-authentication wrappers
├── app.config.ts                 # Expo config
└── package.json
```

## Per-backend-service folder structure (DDD)

Each backend service follows the same internal layout, mirroring PRD §26:

```
services/<service>/
├── domain/                       # entities, value objects, domain services
├── application/                  # use cases, commands, queries
├── infrastructure/
│   ├── postgres/                 # repositories, migrations, sqlc queries
│   ├── messaging/                # event publishers, consumers, outbox writer
│   └── http/                     # handlers, DTOs, validators
├── policies/                     # service-local invariant guards
└── events/                       # event payload definitions specific to this service
```

## Build & deploy

| Repo | Pipeline (per PR) | Pipeline (per merge to main) |
| --- | --- | --- |
| Backend | `golangci-lint` → `go vet` → `go test` → `govulncheck` → `buf lint` | + build per-service images, push to registry, deploy to dev |
| Web | `eslint` → `tsc --noEmit` → `vitest` | + build standalone Next.js image, push, deploy to dev |
| Mobile | `eslint` → `tsc --noEmit` → `vitest` → Maestro smoke tests | + EAS build (preview channel) |

- Per-tag (`vN.N.N`): backend & web deploy to staging → manual gate → production. Mobile submits to TestFlight / Play internal testing.
- Backend uses rolling deploys; Postgres requires careful migration sequencing
- Web uses blue/green via Kubernetes deployment swap
- Mobile: EAS Update for JS-only changes; full app-store submission for native changes

## Local dev

- `task up` (in `trustc-platform`) brings up Postgres, Redis, NATS via `docker compose`
- `task run/<service>` runs a single service with hot reload (`air`)
- `task test/integration` runs the full integration suite using `testcontainers-go`
- `task seed` populates a fixture organization with sample data
- Web: `pnpm dev` against the local backend
- Mobile: `npx expo start` with the iOS simulator or Android emulator pointed at the local gateway

## Migration policy

- Every PR that touches a schema includes the migration
- Migrations are reversible (down provided), even if rolling back is operationally expensive
- Ledger / audit table migrations require explicit reviewer + replay test
- See [06-ledger.md](./06-ledger.md#migration-safety) for ledger-specific constraints

## Testing strategy

| Layer | Tool | What it covers |
| --- | --- | --- |
| Unit (Go) | stdlib `testing`, table-driven | Pure domain logic |
| Property-based (Go) | `pgregory.net/rapid` | Invariants — ledger balance, FSM transitions |
| Integration (Go) | `testcontainers-go` | Service against real Postgres / NATS |
| Invariant suite (Go) | Dedicated `tests/invariants/` package | Cross-cutting invariants from [01-principles.md](./01-principles.md) |
| Unit (web / mobile) | Vitest + Testing Library | Components, hooks |
| End-to-end (web) | Playwright | Top-level flows against a staging stack |
| End-to-end (mobile) | Maestro | Approval / submission flows on a simulator |
| Replay | Custom Go runner | Rebuild projections from event log, assert match |

Coverage gates: 80% line; 100% on the policy aggregator and ledger validator.

## What is NOT in the stack (and why)

- **An ORM beyond `sqlc`.** ORMs hide query behavior; for financial code we want explicit SQL.
- **GraphQL.** Adds complexity; resource-oriented REST suffices.
- **Event sourcing framework.** Outbox-on-Postgres is simpler; see [ADR 0002](../adr/0002-event-sourcing-vs-outbox.md).
- **A centralized "shared services" library.** Forbidden by the domain isolation rules in [02-domains.md](./02-domains.md).
- **Native iOS + Android instead of RN.** Considered; rejected for team size — see the mobile section above.
- **gRPC for service-to-service.** Considered; we chose REST + the event bus to keep the contract surface uniform with the public API. Revisit if internal call latency becomes a bottleneck.
- **A meta-framework for mobile (e.g. Tamagui's full stack)**. Expo + React Native is enough; an extra abstraction layer is not free.

## See also

- [02-domains.md](./02-domains.md) — boundaries the repo structure enforces
- [15-roadmap.md](./15-roadmap.md) — what gets built first; web and mobile are parallel workstreams that begin once Phase 3 (Workflow Engine) exposes a stable API
- [ADR 0002](../adr/0002-event-sourcing-vs-outbox.md), [ADR 0003](../adr/0003-messaging-nats-vs-kafka.md), [ADR 0005](../adr/0005-go-workspace-and-build.md)
