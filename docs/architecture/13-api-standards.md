# 13 — API Design Standards

> Every HTTP API in trustC must conform to these. Sources: PRD §10, §41.

## Principles

### Required of every API

- **Idempotency support** on every state-changing endpoint
- **Audit metadata** on every request (`X-Actor-ID`, `X-Correlation-ID`)
- **Traceability** — `trace_id` in every response, propagated to logs and audit events
- **Explicit status transitions** — endpoints declare the state they move an entity to; no implicit side effects

### Forbidden

- Implicit mutations triggered by GET requests
- Silent failures (any error must surface a structured response with a code)
- Hidden side effects across resources (changing X also touches Y "for convenience")
- Polymorphic endpoints whose behavior depends on a non-obvious request field

## Required headers (incoming)

| Header | Required on | Notes |
| --- | --- | --- |
| `Authorization` | All authenticated | `Bearer <jwt>` |
| `X-Request-ID` | All | Client-supplied or gateway-generated UUID |
| `X-Correlation-ID` | All state-changing | Propagated across services for one business op |
| `X-Actor-ID` | Internal service-to-service | The originating user, not the calling service |
| `Idempotency-Key` | All state-changing | Client-supplied; persisted with the result |
| `Content-Type` | All with body | `application/json` |
| `Accept` | All | `application/json` |

The gateway rejects requests missing required headers with `400 Bad Request`.

## Standard response envelope

### Success

```json
{
  "success": true,
  "data": { /* endpoint-specific */ },
  "trace_id": "uuid",
  "timestamp": "2026-05-02T10:15:30.123Z"
}
```

### Error

```json
{
  "success": false,
  "error": {
    "code": "BUDGET_EXCEEDED",
    "message": "Request exceeds remaining budget",
    "details": {
      "budget_id": "uuid",
      "remaining": 12000,
      "requested": 50000
    }
  },
  "trace_id": "uuid",
  "timestamp": "2026-05-02T10:15:30.123Z"
}
```

Error codes are stable, documented, and machine-checkable. Client behavior keys off `error.code`, never off `error.message`.

## HTTP status codes

| Range | Meaning |
| --- | --- |
| 200 | Success, with data |
| 201 | Created (state-changing endpoints that produced a new resource) |
| 202 | Accepted (long-running operation initiated) |
| 204 | Success, no body |
| 400 | Client error: malformed request, missing required fields/headers |
| 401 | Unauthenticated |
| 403 | Authenticated but not authorized (RBAC failure) |
| 409 | Conflict: idempotency-key collision with different payload, or version conflict |
| 412 | Precondition failed: governance rejected (decision attached) |
| 422 | Unprocessable: validation error |
| 429 | Rate limited |
| 500 | Server error (real bug, paged) |
| 503 | Dependency unavailable |

`412` is reserved specifically for governance rejections. Clients use it to render the rejection reason cleanly:

```json
{
  "success": false,
  "error": {
    "code": "GOVERNANCE_REJECTED",
    "message": "Policy evaluation rejected this request",
    "details": {
      "decision_id": "uuid",
      "reasons": [ /* governance reasons */ ]
    }
  },
  "trace_id": "uuid",
  "timestamp": "..."
}
```

## URL conventions

- Resource-oriented: `/expenses`, `/wallets/{id}`, `/escrow/{id}/release`
- kebab-case in paths
- `snake_case` in JSON keys
- Plural nouns for collections, singular for actions on instances
- Versioning: `/v1/...` prefix; new versions are introduced when a breaking change is needed, supported in parallel for at least 90 days

## Pagination

Cursor-based, never offset:

```
GET /expenses?limit=50&cursor=eyJpZCI6Li4ufQ==
```

Response:

```json
{
  "success": true,
  "data": {
    "items": [ ... ],
    "next_cursor": "eyJpZCI6Li4ufQ==" | null
  },
  ...
}
```

Reasons: offset pagination is broken under concurrent inserts (which we have a lot of) and slow at large offsets.

## Filtering

- Equality: `?status=approved`
- Multi-value: `?status=approved,executed`
- Range: `?created_at_gte=2026-01-01T00:00:00Z&created_at_lt=2026-02-01T00:00:00Z`
- Free-text search: `?q=...` (only where indexed)

Complex queries belong on POST endpoints, not query strings.

## Idempotency semantics

Per [12-security.md](./12-security.md#idempotency):

- The first request with a new `Idempotency-Key` executes and stores result
- Subsequent requests with the same key + same payload return the original result (HTTP code matches the first response)
- Same key + *different* payload returns `409 Conflict`
- Keys are scoped per `(organization_id, endpoint)` and retained for 24 hours minimum

## Versioning & compatibility

- `/v1/...` is the public version surface
- Internal service-to-service APIs use `/internal/v1/...` and have stricter compatibility rules (must support the previous version for the duration of a rolling deploy)
- Event schemas are versioned independently (see [05-events.md](./05-events.md#schema-versioning-rules))

## Documentation

- Every endpoint is documented in OpenAPI (generated from NestJS decorators where possible)
- Examples include error cases, not just happy paths
- Error code dictionary lives at `/v1/errors` (returns the catalog programmatically) and in `docs/api/errors.md` (when implementation starts)

## Performance targets per route class

| Class | p99 |
| --- | --- |
| Read (single entity) | 50 ms |
| Read (list) | 300 ms |
| Write (simple) | 200 ms |
| Write (governance-gated) | 250 ms |
| Long-running (returns 202) | depends on operation; status endpoint within targets above |

## See also

- [12-security.md](./12-security.md) — auth, RBAC, multi-tenancy at the request boundary
- [05-events.md](./05-events.md) — event envelope (counterpart to API envelope)
