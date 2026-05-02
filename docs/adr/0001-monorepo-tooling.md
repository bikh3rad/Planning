# ADR 0001 — Monorepo tooling

**Status:** Accepted
**Date:** 2026-05-02
**Deciders:** Solution architecture

## Context

trustC will be built as 7+ services in a single repository. We need a tool that:

- Manages workspace dependencies (services depend on shared `libs/`)
- Caches builds and test runs (CI on every PR will be expensive otherwise)
- Supports per-service deployment without rebuilding the world
- Plays well with NestJS, TypeScript, and pnpm
- Is operationally simple — small team, can't afford a full-time build engineer

## Decision

Use **pnpm workspaces + Turborepo**.

- `pnpm` for package management (faster, stricter, smaller node_modules than npm/yarn)
- `turbo` for orchestrating tasks across the workspace with caching
- Per-service Dockerfiles consume the pnpm workspace via `pnpm deploy --prod`

## Alternatives considered

- **Nx** — more powerful generators, deeper TS integration, NestJS plugins. But: heavier conceptual overhead, opinionated project graph, more vendor lock-in. The generators are nice but most of our scaffolding is one-time. Reject.
- **Plain pnpm workspaces** (no Turborepo) — works, but no task graph or caching. CI would re-run everything on every PR. Reject.
- **Bazel** — would handle our scale, but operationally enormous. We do not have the personnel. Reject.
- **Single big package, no workspace** — trivially simple, but couples deploys, prevents per-service versioning, blurs ownership. Reject.

## Consequences

### Positive

- Fast PR feedback: turbo caches lint/test/build by content hash
- Clean per-service Dockerfiles — each app deployable independently
- Familiar tooling for any TypeScript engineer; low onboarding cost
- pnpm's strict mode catches accidental cross-service imports we'd miss with npm

### Negative

- Turborepo is relatively young; we'll be on the upgrade treadmill
- Caching subtleties — incorrect `inputs` declarations cause stale-cache bugs
- pnpm's symlink-heavy node_modules occasionally surprises tools that assume hoisting

### Revisit when

- The repo crosses ~30 services / ~100 packages
- Build times exceed 10 minutes despite caching
- A clear win for Nx emerges (e.g. official NestJS support that materially helps)

## Related

- [docs/architecture/14-tech-stack.md](../architecture/14-tech-stack.md)
