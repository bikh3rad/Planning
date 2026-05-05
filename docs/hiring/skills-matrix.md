# Skills Matrix — trustC Engineering Team

> Who we need, what they must know, and at what depth. Use this alongside the job descriptions when evaluating candidates or planning the team.

## Depth scale

| Level | Meaning |
| --- | --- |
| **Expert** | Can design, debug, and teach; owns decisions in this area |
| **Proficient** | Can build and maintain without hand-holding |
| **Working** | Can contribute with occasional guidance |
| **Awareness** | Understands concepts; not expected to lead |

---

## Role map

| Role | Services they own | Count (v1) |
| --- | --- | --- |
| Backend Engineer — Financial Core | Ledger, Treasury, Governance | 2 |
| Backend Engineer — Platform | Workflow, Auth, Audit, Notification, Gateway | 2 |
| Frontend Engineer | Web (`trustc-web`) | 1 |
| Mobile Engineer | React Native (`trustc-mobile`) | 1 |
| Platform / Infrastructure Engineer | Kubernetes, Terraform, CI/CD, observability | 1 |
| Tech Lead / Architect | Cross-cutting design, ADRs, code review | 1 |

Minimum viable team: **7 engineers** to staff v1 delivery (Phase 1–3 of the roadmap).

---

## Backend Engineer — Financial Core

Owns Ledger, Treasury, and Governance — the three most correctness-critical services.

| Skill | Required depth | Notes |
| --- | --- | --- |
| **Go** | Expert | Clean architecture, interfaces, concurrency primitives |
| **PostgreSQL** | Expert | ACID transactions, triggers, RLS, query planning |
| **Double-entry accounting** | Proficient | Ledger invariants: `sum(debits) == sum(credits)` always |
| **Append-only / immutable data design** | Proficient | No UPDATE/DELETE; corrections via compensating entries |
| **Event-driven architecture** | Proficient | Outbox pattern, NATS JetStream, event envelopes |
| **Optimistic locking / concurrency** | Proficient | Race-free balance updates, version columns |
| **`sqlc`** | Working | Compile-time-checked SQL in repo layer |
| **`pgx` v5** | Working | Postgres-native Go driver |
| **Domain-Driven Design** | Working | Bounded contexts, aggregate design |
| **Idempotency patterns** | Proficient | Every write is safe to retry |
| **Security thinking** | Proficient | Hostile insider model; cannot bypass controls |
| **Google Wire (DI)** | Working | Composition roots per service |
| **Docker / Kubernetes** | Working | Enough to build and deploy a service |
| **OpenTelemetry** | Awareness | Tracing, structured logging |

**Nice to have:** experience with financial systems, ledger databases, or compliance-adjacent work (SOC 2, PCI-DSS awareness).

**Hard disqualifier:** candidates who treat `UPDATE`/`DELETE` as the normal correction mechanism for financial data.

---

## Backend Engineer — Platform

Owns Workflow, Auth, Audit, Notification, and API Gateway — the operational backbone.

| Skill | Required depth | Notes |
| --- | --- | --- |
| **Go** | Expert | Same clean-architecture conventions as Financial Core |
| **PostgreSQL** | Proficient | Multi-tenancy via RLS, migrations, index design |
| **State machine design** | Proficient | Workflow FSM, fund state transitions |
| **Auth / OIDC / JWT** | Proficient | Auth service owns identities, signing keys, sessions |
| **RBAC design** | Proficient | Roles, permissions, actor identity across services |
| **NATS JetStream** | Working | Pub/sub, durable consumers, deduplication |
| **Event-driven architecture** | Working | Outbox pattern, at-least-once delivery |
| **Redis** | Working | Idempotency cache, rate limiting, short-lived state |
| **Protobuf / buf** | Working | Cross-repo contract generation |
| **API design** | Proficient | REST conventions, versioning, idempotency semantics (see `13-api-standards.md`) |
| **Docker / Kubernetes** | Working | Build, deploy, rolling upgrades |
| **OpenTelemetry** | Working | Distributed tracing across service calls |
| **Google Wire (DI)** | Working | Same pattern as Financial Core |

**Nice to have:** experience with notification delivery systems, webhook infrastructure, or approval workflow engines.

---

## Frontend Engineer

Owns `trustc-web` — the operator and investor dashboard.

| Skill | Required depth | Notes |
| --- | --- | --- |
| **TypeScript** (strict) | Expert | All code is strict-mode TS |
| **Next.js 14** (App Router) | Expert | RSC, SSR, routing, API routes |
| **React** | Expert | Hooks, composition, state management |
| **TanStack Query** | Proficient | Data fetching, caching, optimistic updates |
| **Tailwind CSS** | Proficient | Utility-first styling |
| **Radix UI / shadcn** | Working | Accessible headless components |
| **OIDC client** | Working | Auth integration (`oidc-client-ts`) |
| **Server-Sent Events** | Working | Real-time dashboard updates from gateway |
| **Recharts** | Working | Runway / burn / utilization views |
| **REST API consumption** | Proficient | Consumes `@trustc/contracts` generated types |
| **Accessibility (WCAG)** | Working | Operator UI used under time pressure; must be usable |

**Nice to have:** experience with financial dashboards, data-heavy UIs, or design systems.

---

## Mobile Engineer

Owns `trustc-mobile` — the approval and expense submission app (iOS + Android).

| Skill | Required depth | Notes |
| --- | --- | --- |
| **React Native** | Expert | Expo managed workflow |
| **TypeScript** | Proficient | Same contracts as web |
| **Expo SDK** | Proficient | Routing, secure store, notifications, local auth |
| **Biometric authentication** | Working | Required for approval submission (PRD §32) |
| **OIDC + PKCE** | Working | `expo-auth-session` mobile OAuth flow |
| **TanStack Query** | Working | Shared data-fetching pattern with web |
| **WebSockets** | Working | Real-time approval-queue updates |
| **EAS Build / Update** | Working | OTA JS updates vs full app-store submissions |
| **Secure storage** | Working | Refresh tokens in Keychain / Keystore, never plaintext |

**Nice to have:** experience shipping to both App Store and Play Store; Playwright / Maestro E2E testing.

**Hard disqualifier:** treating the mobile app as a thin wrapper that "just calls the API" — the app owns biometric auth and secure token storage, both of which are security-critical.

---

## Platform / Infrastructure Engineer

Owns the deployment substrate, CI/CD pipelines, observability stack, and secret management.

| Skill | Required depth | Notes |
| --- | --- | --- |
| **Kubernetes** | Expert | Deployments, services, rolling upgrades, health probes |
| **Terraform** | Expert | All infrastructure as code; no click-ops |
| **Docker** | Proficient | Multi-stage builds, distroless images, image scanning |
| **GitHub Actions** | Proficient | Per-PR and per-merge pipelines for three repos |
| **NATS JetStream** (ops) | Proficient | 3-node cluster setup, stream configuration, monitoring |
| **PostgreSQL** (ops) | Proficient | Connection pooling (PgBouncer), backup/restore, migration sequencing |
| **Redis / Valkey** (ops) | Working | Cluster setup, persistence configuration |
| **OpenTelemetry collector** | Proficient | OTLP pipeline → Tempo / Loki / Prometheus / Grafana |
| **Secrets management** | Proficient | AWS Secrets Manager or Vault; rotation; off-disk |
| **Cloud provider** (AWS / GCP) | Proficient | At least one; cloud-neutral design preferred |
| **Security hardening** | Working | Network policies, pod security, least-privilege IAM |
| **Incident response** | Working | On-call readiness; runbook authoring |

---

## Tech Lead / Architect

Sets technical direction, owns ADRs, reviews cross-cutting changes, unblocks other engineers.

Must have **all of the Backend Engineer — Financial Core** skills at Expert level, plus:

| Skill | Required depth | Notes |
| --- | --- | --- |
| **System design** | Expert | Trade-off analysis, documented in ADRs |
| **Financial systems** | Proficient | Ledger design, accounting principles, audit requirements |
| **Multi-tenancy at the DB layer** | Expert | RLS, tenant isolation, cross-tenant leak prevention |
| **Security architecture** | Expert | Threat modeling, RBAC design, hostile insider model |
| **API design** | Expert | Versioning, backward compatibility, idempotency |
| **Distributed systems** | Proficient | At-least-once delivery, deduplication, ordering guarantees |
| **Cross-functional communication** | Proficient | Must bridge product intent (PRD) to engineering decisions (ADRs) |

---

## Skills the whole team must share

Regardless of role, every engineer joining trustC must internalize:

1. **Append-only financial data** — no mutation of ledger, audit, or approval records.
2. **Idempotency first** — every state-changing operation is safe to replay.
3. **Multi-tenancy discipline** — `organization_id` on every row; no cross-tenant reads.
4. **Audit by default** — every state change produces an event; silence is a bug.
5. **Integrity over convenience** — correctness beats ergonomics when they conflict (PRD §39).

These are checked in code review and in the invariant test suite (`tests/invariants/`).

---

## See also

- [hiring/job-descriptions.md](./job-descriptions.md) — role-specific job posts
- [architecture/14-tech-stack.md](../architecture/14-tech-stack.md) — full stack detail
- [architecture/01-principles.md](../architecture/01-principles.md) — non-negotiables every engineer must know
- [architecture/15-roadmap.md](../architecture/15-roadmap.md) — delivery phases that drive hiring sequencing
