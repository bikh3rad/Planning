# ADR 0005 — Go workspace & build tooling

**Status:** Accepted
**Date:** 2026-05-02
**Deciders:** Solution architecture
**Supersedes:** [ADR 0001](./0001-monorepo-tooling.md)

## Context

ADR 0001 chose pnpm + Turborepo for a TypeScript / NestJS monorepo. The implementation language has since been changed to Go. We need a fresh decision for Go workspace structure and build orchestration.

Forces:

- 7 backend services sharing significant internal code (event contracts, postgres helpers, crypto, observability)
- Small team — operationally simple beats theoretically optimal
- Go's standard tooling (`go build`, `go test`, build cache) is already excellent
- Mobile and web apps live in separate repos (see [14-tech-stack.md](../architecture/14-tech-stack.md)); this ADR scopes only the backend
- Cross-language contracts are needed (web + mobile consume them), but the contract layer is solved separately by protobuf + buf

## Decision

Use **a single Go module** with Go's standard build tooling, orchestrated by **Taskfile** (`task`).

Concretely:

- One `go.mod` at the repo root for the entire backend monorepo
- Per-service entrypoints under `cmd/<service>/main.go`
- Shared infrastructure code under `internal/<package>` — Go's `internal/` directory makes these packages unimportable from outside the module, enforcing the boundary at the language level
- Per-service domain code under `services/<service>/` (regular packages)
- `Taskfile.yml` declares build / test / lint / migrate / run tasks per service
- Per-service Dockerfiles `COPY` only what they need; `go build` produces a static binary deployed in a distroless image
- CI uses `go test ./...` with the GitHub Actions Go-build-cache action

Layout (full structure in [14-tech-stack.md](../architecture/14-tech-stack.md)):

```
trustc-platform/
├── cmd/<service>/main.go
├── services/<service>/{domain,application,infrastructure,policies}/
├── internal/{event,crypto,pgx,nats,otel,httpx,contracts,testing}/
├── contracts/                    # .proto source of truth
├── infrastructure/{terraform,kubernetes,docker}/
├── tools/{codegen,policy-cli,replay}/
├── go.mod
└── Taskfile.yml
```

## Alternatives considered

- **Multi-module with `go.work`** (each service has its own `go.mod`)
  - Pros: per-service dep isolation, easier to extract a service to its own repo later
  - Cons: more ceremony for shared internal packages; `gopls` historically slower with workspaces; the `internal/` rule has subtle interactions across multiple modules
  - **Reject for v1** — single-module is simpler at our scale, and "split later" is a directory move plus dep cleanup, not a rewrite
- **Bazel (`rules_go`)**
  - Pros: hermetic, multi-language, excellent caching at scale
  - Cons: enormous operational weight; team has no Bazel experience; same reasoning that rejected it for the TS stack in ADR 0001
  - **Reject**
- **Mage** (Go-native task runner)
  - Pros: tasks written in Go, no extra binary
  - Cons: tasks become Go code that compiles; harder for non-Go contributors to read; slower than declarative YAML for simple cases
  - **Reject** in favor of Taskfile, which is YAML and self-documenting
- **Plain Makefile**
  - Pros: zero dependencies, ubiquitous
  - Cons: cross-platform issues, tab-vs-space quirks, harder to express task dependencies and per-target inputs
  - **Reject** — Taskfile is a low-cost upgrade
- **Polyrepo (one repo per service)**
  - Pros: fully independent deploys, minimal blast radius
  - Cons: shared internal code becomes a publishing problem; cross-cutting changes turn into N PRs; CI / observability config duplicated
  - **Reject** — premature for our team size

## Consequences

### Positive

- New contributors clone one repo, run `task up`, and have the full backend locally
- `go build ./cmd/<service>` produces a deployable binary using the standard build cache
- `go test ./...` runs every test in parallel; CI is fast without extra caching infrastructure
- `internal/` packages cannot be imported by anything outside the module — the language enforces the boundary that ADR 0001 needed Turborepo + pnpm strict mode to approximate
- Shared cross-cutting code lives in one place, versioned with the services that use it

### Negative

- Single `go.mod` means a transitive-dep update affects every service's image
  - Mitigated by `govulncheck` in CI and Go's typically small dep trees
- Splitting a service into its own repo later is a directory move plus dep cleanup, not a one-liner
- Taskfile is one more tool to learn, but it's YAML — shallow curve

### Migration discipline

- **No service writes outside its own service's DB schema.** The single Go module does not relax the schema-isolation rule from [02-domains.md](../architecture/02-domains.md).
- **No business-logic helpers in `internal/`.** Only protocol, transport, and observability utilities. Cross-domain types are forbidden.
- **`go mod tidy` runs in CI** — undeclared deps fail the build.

### Revisit when

- A service's dep set genuinely conflicts with another's (Go's flat dep model handles this fine for internal-only deps; conflicts arise only with C-bound libraries, which we avoid)
- We approach 30+ services or 50+ engineers — at that point, revisit Bazel or polyrepo
- Build / test wall-clock time exceeds 10 minutes despite caching

## Related

- [docs/architecture/14-tech-stack.md](../architecture/14-tech-stack.md)
- [ADR 0001](./0001-monorepo-tooling.md) — superseded
- [ADR 0002](./0002-event-sourcing-vs-outbox.md), [ADR 0003](./0003-messaging-nats-vs-kafka.md), [ADR 0004](./0004-database-immutability-enforcement.md) — unaffected by language choice
