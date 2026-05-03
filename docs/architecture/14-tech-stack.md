# 14 — Tech Stack & Repository Structure

> Stack choices and how the implementation repos will be laid out. Sources: PRD §17, §25, §26, §55.

trustC ships as three independently-deployable artifacts: a Go backend (~7 services + a gateway), a Next.js web application, and a React Native mobile app. Each lives in its own repository, sharing types via a generated contracts package.

## Stack at a glance

### Backend

The backend is seeded from [`mequq/go-template`](https://github.com/mequq/go-template) and adopts its service-internal conventions wholesale; we replicate them across services within a single Go module per [ADR 0005](../adr/0005-go-workspace-and-build.md).

| Layer | Choice | Why |
| --- | --- | --- |
| Language | Go 1.22+ | Strong static typing, fast builds, simple deployment, stdlib `ServeMux` pattern routing |
| HTTP router | stdlib `net/http.ServeMux` | Go 1.22 pattern matching covers our routing needs; no external router dep |
| DI | **Google Wire** | Compile-time DI; per-service composition root at `cmd/<service>/wire.go` |
| Config | **koanf** | YAML file (`--config` flag) + `APP_`-prefixed env overlay |
| DB driver | `pgx` (v5) + `otelsql` | Postgres-native; `otelsql` traces every query without per-call instrumentation |
| Query layer | `sqlc` | Compile-time-checked SQL types; an additive layer on top of `pgx` |
| Migrations | `golang-migrate` | Up/down migrations; per-service subdirs under `migrations/<service>/` |
| Messaging client | `nats.go` (JetStream) | Official Go client, durable consumers, replay |
| Crypto | `crypto/ed25519`, `crypto/sha256` | stdlib — no third-party deps for security-critical code |
| Cache / locks | Redis 7 / Valkey | Idempotency cache, rate limiting, short-lived state |
| Lifecycle | `app.Controller` registry | Components self-register `Start` / `Shutdown` / `Healthz`; `biz.healthz` fans out concurrently |
| Mocks | **mockery** v2 | Generated from `services/<svc>/internal/biz` into `internal/mocks` (config in `.mockery.yaml`) |
| API contracts | **OpenAPI 3.1** (code-first via **swag**) | Annotations on handlers; spec published to `contracts/openapi/`; UI at `/swagger/` |
| Build tooling | Single Go module + **Taskfile** | See [ADR 0005](../adr/0005-go-workspace-and-build.md) |
| Logging | `log/slog` (stdlib) bridged to OTel logs | Structured, zero-dep, Go 1.21+ |
| Observability | OpenTelemetry Go SDK over OTLP gRPC → Grafana / Tempo / Loki / Prometheus | Vendor-neutral |
| Testing | stdlib `testing` + `testcontainers-go` + `pgregory.net/rapid` + **ramsql** for in-memory tests | Standard idioms; real Postgres / NATS in integration tests; ramsql for fast unit tests |
| Containerization | Docker (multi-stage, distroless final) | Static Go binaries → tiny images |
| Orchestration | Kubernetes | PRD §55 |
| IaC | Terraform | PRD §25 |
| CI | GitHub Actions | Matches the repo host |
| CD | **ArgoCD** | GitOps — syncs cluster to manifests in `infrastructure/kubernetes/` |
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

- **stdlib `net/http.ServeMux` over `chi`/`gin`/`fiber`/`echo`**: Go 1.22's pattern matching covers our needs; `otelhttp` plus a small set of `pkg/middlewares` covers the rest. No router dep to track.
- **Google Wire over manual constructor injection**: with 7 services and a deep dependency graph per service, Wire's compile-time validation pays for the codegen step. Composition root per service at `cmd/<service>/wire.go`.
- **koanf over Viper**: simpler API, cleaner env-overlay semantics, no implicit globals. Adopted from the template.
- **mockery over hand-written mocks or testify mocks**: generated mocks stay in sync with interfaces; CI fails when an interface changes without regenerating.
- **swag (code-first) over OpenAPI-first**: faster to ship while the team is small. Revisit if non-Go consumers need to author the spec.
- **`sqlc` over an ORM (`gorm`, `ent`)**: financial code requires explicit, reviewable SQL. ORMs hide query behavior; sqlc generates types from SQL we write. This is a deliberate addition to the template, which uses raw `pgx`.
- **`otelsql` wrapping `pgx`**: every query gets a span without per-call boilerplate.
- **PostgreSQL only, no MongoDB / DynamoDB**: financial workloads need ACID, foreign keys, DB-level constraints (notably for [ADR 0004](../adr/0004-database-immutability-enforcement.md)) and Postgres RLS for multi-tenancy.
- **NATS JetStream over Kafka**: target throughput (1M ledger entries/day ≈ 12/s) is well within NATS, operationally simpler. See [ADR 0003](../adr/0003-messaging-nats-vs-kafka.md).
- **REST over gRPC for internal calls**: keeps the contract surface uniform with the public API. Revisit if internal call latency becomes a bottleneck.
- **`Taskfile` over `Make + dagger` (the template's choice)**: cross-platform, declarative, avoids Make's tab/space pitfalls. We may layer dagger pipelines on top later for CI parity, but Taskfile is the primary orchestrator. See [ADR 0005](../adr/0005-go-workspace-and-build.md).
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

Cross-cutting API shapes are defined as **OpenAPI 3.1 specs** in `trustc-platform/contracts/openapi/`. Generation:

- `oapi-codegen` produces Go server stubs and request/response types (committed in `trustc-platform/internal/contracts/`)
- `openapi-typescript` produces a published npm package `@trustc/contracts` consumed by web and mobile
- `spectral lint` + `oasdiff breaking` run in CI to catch spec violations and breaking changes

This avoids hand-written client types that drift from the server while keeping each repo independently buildable.

## Backend repository structure (`trustc-platform`)

Seeded from [`mequq/go-template`](https://github.com/mequq/go-template); the template's single-service layout is replicated under `services/<service>/internal/` for each service inside the single Go module. See [ADR 0005](../adr/0005-go-workspace-and-build.md) for the full rationale.

```
trustc-platform/
├── cmd/                          # service entrypoints — one composition root per service
│   ├── auth/{main.go, wire.go, wire_gen.go}
│   ├── workflow/{main.go, wire.go, wire_gen.go}
│   ├── treasury/...
│   ├── ledger/...
│   ├── governance/...
│   ├── audit/...
│   ├── notification/...
│   └── gateway/...
├── services/                     # per-service code; positional `internal/` enforces isolation
│   ├── workflow/
│   │   └── internal/             # only cmd/workflow can import this subtree
│   │       ├── service/
│   │       │   ├── server.go     # HTTP wiring, /metrics, /swagger
│   │       │   ├── handler/      # HTTP handlers (implement service.Handler)
│   │       │   └── dto/          # request / response shapes
│   │       ├── biz/              # use cases (Usecase…, Repository… interfaces)
│   │       ├── repo/             # repository implementations
│   │       ├── datasource/       # service-specific DB / queue clients
│   │       ├── entity/           # domain types
│   │       ├── mocks/            # mockery-generated
│   │       ├── policies/         # service-local invariant guards
│   │       └── events/           # service-local event payload definitions
│   ├── treasury/internal/...
│   ├── ledger/internal/...
│   └── ...
├── internal/                     # cross-service shared infrastructure (no business logic)
│   ├── app/                      # Application, HTTPServer, Controller, KConfig, AppLogger, OTLP
│   ├── event/                    # envelope schema, canonical-JSON, signing, validation
│   ├── contracts/                # oapi-codegen-generated Go types (output of `task generate`)
│   ├── crypto/                   # ed25519, sha256 helpers
│   ├── pgx/                      # pool helpers, RLS context setter, otelsql wiring, migration runner
│   ├── nats/                     # JetStream client, outbox relay
│   ├── otel/                     # tracer / meter / logger setup
│   ├── httpx/                    # required-headers middleware, response envelope, error mapping
│   └── testing/                  # invariant test runners, fixtures, builders, ramsql helpers
├── pkg/middlewares/              # external-importable per-route HTTP middlewares
├── contracts/
│   └── openapi/                  # OpenAPI 3.1 specs — source of truth for all API shapes
├── migrations/                   # per-service: migrations/<service>/000001_*.{up,down}.sql
├── infrastructure/
│   ├── terraform/
│   ├── kubernetes/
│   └── compose/                  # docker-compose includes (monitoring, postgres, redis)
├── tools/
│   ├── codegen/                  # oapi-codegen + sqlc + wire + mockery + swag wrappers
│   ├── policy-cli/               # author + validate governance policies
│   └── replay/                   # event replay for testing
├── docs/                         # implementation-level docs (operational, runbooks)
├── docker-compose.yml            # aggregates infrastructure/compose/* includes
├── Taskfile.yml
├── .mockery.yaml
├── .golangci.yaml
├── go.mod
└── go.sum
```

Rules:

- **No business logic in repo-level `internal/`.** Only protocol-level utilities (event contracts, app lifecycle, crypto, transport, observability). Cross-domain types are forbidden — that would re-introduce the "shared god service" anti-pattern (PRD §64).
- **Each service's `services/<svc>/internal/` is unimportable from any other service** — Go's positional `internal/` rule enforces this at the language level.
- **Each service owns its DB schema and migrations.** Migrations live under `migrations/<service>/`. No cross-schema reads at the application layer.
- **`wire_gen.go` is generated**, never hand-edited. Modify the relevant `wire.go` provider set and run `task generate`.
- **Every datasource, handler, and use case registers its own healthz hook** on the shared `app.Controller` — no central healthz orchestration.

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

## Per-backend-service layout (clean architecture, from the template)

Each backend service follows the template's clean-architecture structure, mirroring PRD §26:

```
services/<service>/internal/
├── service/
│   ├── server.go                 # HTTP wiring + /metrics + /swagger
│   ├── handler/                  # HTTP handlers — implement service.Handler
│   └── dto/                      # request / response shapes
├── biz/                          # use cases: Usecase… interfaces (consumed by handlers)
│                                 #            Repository… interfaces (implemented by repo)
├── repo/                         # repository implementations bound via wire.Bind
├── datasource/                   # service-specific DB / queue clients (most live in repo-level internal/)
├── entity/                       # domain types
├── mocks/                        # mockery-generated mocks for biz interfaces
├── policies/                     # service-local invariant guards
└── events/                       # event payload definitions specific to this service
```

Per-service composition root (`cmd/<service>/wire.go`):

```go
//go:build wireinject
// +build wireinject

package main

func newApp(...) (*app.Application, func(), error) {
    panic(wire.Build(
        app.ProviderSet,                                    // Application, HTTPServer, Controller, KConfig, Logger, OTLP
        internalpgx.ProviderSet, internalnats.ProviderSet,  // shared datasource clients
        workflowdatasource.ProviderSet,                     // service-specific clients (if any)
        workflowrepo.ProviderSet,                           // repo.Bind to biz.Repository… interfaces
        workflowbiz.ProviderSet,                            // use cases
        workflowservice.ProviderSet,                        // HTTP wiring + handlers
    ))
}
```

Adding a handler: implement `service.Handler` (`RegisterHandler(ctx) error` registers routes on the injected `*http.ServeMux`), expose a `New…` provider, append it to the service's `NewServiceList` in `services/<svc>/internal/service/handler/wire.go`, then run `task generate`.

## Build & deploy

| Repo | Pipeline (per PR) | Pipeline (per merge to main) |
| --- | --- | --- |
| Backend | `task generate` (must produce no diff) → `golangci-lint` → `go vet` → `go test` → `govulncheck` → `spectral lint` → `oasdiff breaking` | + build per-service images, push to registry, update image tag in `infrastructure/kubernetes/` |
| Web | `eslint` → `tsc --noEmit` → `vitest` | + build standalone Next.js image, push, update image tag |
| Mobile | `eslint` → `tsc --noEmit` → `vitest` → Maestro smoke tests | + EAS build (preview channel) |

**CD is handled by ArgoCD (GitOps):**

- CI writes the new image tag into `infrastructure/kubernetes/<env>/` and commits; ArgoCD detects the diff and syncs the cluster
- Environments: `dev` (auto-sync on every merge), `staging` and `production` (manual sync gate in ArgoCD)
- Per-tag (`vN.N.N`): CI promotes the tag to staging manifests → ArgoCD syncs staging → manual ArgoCD sync to production. Mobile submits to TestFlight / Play internal testing.
- Backend uses rolling deploys (ArgoCD `RollingUpdate`); Postgres migrations run as an init container / pre-sync hook before pods are replaced
- Web uses blue/green via ArgoCD `Rollout` (Argo Rollouts)
- Mobile: EAS Update for JS-only changes; full app-store submission for native changes

## Common backend tasks

Adapted from the template's Make targets to Taskfile, parameterised by service:

| Task | What it does |
| --- | --- |
| `task generate` | Regenerate Wire DI graphs (every `cmd/<service>`), mockery mocks, OpenAPI specs (swag), oapi-codegen stubs, sqlc queries; `go mod tidy` |
| `task devtools` | One-time: install `golangci-lint`, `gofumpt`, `wire`, `mockery`, `swag`, `oapi-codegen`, `spectral`, `gci`, `sqlc` |
| `task lint` | `golangci-lint run` against the whole tree |
| `task test` | `go test ./...` |
| `task test:integration` | Integration suite using `testcontainers-go` |
| `task test:invariants` | Cross-cutting invariant suite (the `tests/invariants/` package) |
| `task swagger:<service>` | Regenerate `services/<svc>/docs/` from swag annotations |
| `task run:<service>` | Run a single service locally with hot reload (`air`) |
| `task build:<service>` | Build the per-service Docker image |
| `task migrate:<service>` | Apply / rollback golang-migrate against the local DB |

## Local dev

- `task up` brings up Postgres, Redis, NATS, and the monitoring stack (Tempo, Loki, Prometheus, Grafana, OTel collector) via `docker compose`
- `task run:<service>` runs a single service against the local stack
- Each service reads `config.yaml` (copied from `config.example.yaml`) overlaid with `APP_`-prefixed env vars
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
- **gRPC / protobuf.** All APIs (internal and external) use REST + OpenAPI 3.1. This keeps the contract surface uniform, eliminates a separate IDL toolchain, and means a single spec drives both server stubs (`oapi-codegen`) and client types (`openapi-typescript`).
- **A meta-framework for mobile (e.g. Tamagui's full stack)**. Expo + React Native is enough; an extra abstraction layer is not free.
- **Manual mock writing.** Mockery is non-negotiable for the `biz.Repository…` interfaces; hand-written mocks drift.
- **Hand-edited `wire_gen.go`.** Always regenerate via `task generate`; CI fails if the regenerated file differs from committed.

## Deltas from `mequq/go-template`

We adopt the template's choices wholesale except for these deliberate deltas:

| Concern | Template | trustC | Why |
| --- | --- | --- | --- |
| Module structure | Single service per repo | Multi-service in one Go module | 7 services share enough infrastructure that a single module is operationally simpler. See [ADR 0005](../adr/0005-go-workspace-and-build.md). |
| Build orchestration | Make + dagger | Taskfile | Cross-platform, declarative, no Docker-in-Docker for casual local builds |
| Query layer | raw `pgx` | `pgx` + `sqlc` | Compile-time type safety on financial SQL; sqlc is additive |
| Migrations location | `migrations/` (single dir) | `migrations/<service>/` | Per-service ownership of schema |
| API contracts | None (template is single-service) | OpenAPI 3.1 specs in `contracts/openapi/`; `oapi-codegen` → Go; `openapi-typescript` → npm | Single source of truth for all API shapes; no protobuf/gRPC toolchain |
| CD | None defined in template | ArgoCD (GitOps) | Declarative sync from `infrastructure/kubernetes/`; env promotion via manifest branches/dirs |
| Healthz endpoint paths | `/healthz/...` | Same — kept |
| API doc UI | `/swagger/` | Same — kept |
| Config bootstrap | `config.yaml` + `APP_` env | Same — kept |
| Lifecycle pattern | `app.Controller` self-registration | Same — kept (in repo-level `internal/app`) |

## See also

- [02-domains.md](./02-domains.md) — boundaries the repo structure enforces
- [15-roadmap.md](./15-roadmap.md) — what gets built first; web and mobile are parallel workstreams that begin once Phase 3 (Workflow Engine) exposes a stable API
- [ADR 0002](../adr/0002-event-sourcing-vs-outbox.md), [ADR 0003](../adr/0003-messaging-nats-vs-kafka.md), [ADR 0005](../adr/0005-go-workspace-and-build.md)
- Template: [`mequq/go-template`](https://github.com/mequq/go-template) — the seed scaffold
