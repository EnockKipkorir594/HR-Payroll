# API Conventions

Phase 1 · Binding for every route. These conventions exist so that ten developers produce one
consistent API without a style debate in review.

## 1. Versioning and prefixes

- All routes are prefixed `/v1`. The version is in the path, so it is visible in logs and unambiguous.
- Feature routers are mounted from `src/routes/index.ts` and nothing else calls `app.use()` for a
  feature. Adding a router in one place keeps the route table greppable.
- Health endpoints sit **outside** `/v1` because orchestrators probe fixed paths:
  `GET /healthz` (liveness) and `GET /readyz` (readiness).

## 2. Response envelope

Every JSON response uses one of two shapes. Never a bare value at the top level.

**Success**

```json
{
  "success": true,
  "data": { "id": "…", "employeeNumber": "KE-0001" },
  "meta": { "requestId": "01JC…" }
}
```

**List**

```json
{
  "success": true,
  "data": [ /* … */ ],
  "meta": {
    "requestId": "01JC…",
    "pagination": { "page": 1, "pageSize": 25, "totalItems": 412, "totalPages": 17 }
  }
}
```

**Error**

```json
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Request body failed validation",
    "details": [
      { "path": "body.email", "message": "Invalid email address", "code": "invalid_string" }
    ]
  },
  "meta": { "requestId": "01JC…" }
}
```

Rules:

- `error.code` is a stable `SCREAMING_SNAKE_CASE` string. Clients branch on `code`, never on `message`.
- `error.message` is human-readable and may change. Never parse it.
- `error.details` is present for validation failures and for domain rule violations. Omitted otherwise.
- `meta.requestId` is present on every response, success or failure. It is the correlation key that
  ties a user report to a log line and an audit record.

## 3. Status codes

| Code | Used when |
| --- | --- |
| `200` | Successful read or update |
| `201` | Resource created; `Location` header set |
| `204` | Successful delete or logout; no body |
| `400` | Malformed request that Zod cannot describe field-by-field |
| `401` | Missing, malformed, expired or invalid token |
| `403` | Authenticated but not permitted |
| `404` | Not found **or** belongs to another organization — see [ADR-0005](./adr/0005-tenant-isolation.md) |
| `409` | Conflict: duplicate unique value, or invalid state transition |
| `422` | Semantically invalid: well-formed but violates a business rule |
| `429` | Rate limited; `Retry-After` header set |
| `500` | Unhandled error. Body never contains internals |
| `503` | Dependency unavailable; readiness failing |

**`403` versus `404`.** A cross-tenant read returns `404`, because `403` confirms the record exists.
A permission failure *within* your own organization returns `403`.

**`409` versus `422`.** `409` means the request conflicts with current state (duplicate email, run
already finalized). `422` means the request is well-formed but semantically wrong (end date before
start date).

## 4. Error codes

Codes are declared once in `src/lib/errors.ts` and are the API contract.

| Code | Status | Meaning |
| --- | --- | --- |
| `VALIDATION_ERROR` | 422 | Zod rejected the input |
| `INVALID_CREDENTIALS` | 401 | Wrong email or password. Deliberately identical for both |
| `TOKEN_EXPIRED` | 401 | Access token expired; client should refresh |
| `TOKEN_INVALID` | 401 | Signature, algorithm or claim check failed |
| `REFRESH_TOKEN_REUSED` | 401 | Rotated token replayed; family revoked |
| `FORBIDDEN` | 403 | Permission check failed |
| `NOT_FOUND` | 404 | Absent, or in another organization |
| `NO_ACTIVE_MEMBERSHIP` | 403 | Authenticated user has no active organization |
| `NOT_A_MEMBER` | 403 | User tried to switch into an organization they do not belong to |
| `CONFLICT` | 409 | Unique constraint or state conflict |
| `INVALID_STATE_TRANSITION` | 409 | Payroll run transition not permitted from the current state |
| `RATE_LIMITED` | 429 | Too many requests |
| `INTERNAL_ERROR` | 500 | Unexpected failure; details go to logs, never to the client |
| `DEPENDENCY_UNAVAILABLE` | 503 | Database or cache unreachable |

## 5. Validation

- Every route with a body, query or params declares a Zod schema in its module's `*.schema.ts` and
  uses `validate({ body, query, params })` middleware.
- **Unknown fields are rejected** (`.strict()`). Silently ignoring an unexpected field is how a client
  bug becomes a silent data loss bug.
- **Query and params are always coerced and bounded.** A `pageSize` above the maximum is an error, not
  a silent clamp, so the client learns its request was wrong.
- Money arrives as a **string** and is parsed by the money value object, never as a JSON number.
  See [ADR-0006](./adr/0006-money-precision-and-rounding.md).

## 6. Pagination

- List endpoints accept `?page` (default 1) and `?pageSize` (default 25, **maximum 100**).
- Responses always carry the full `pagination` object, including `totalItems`, so the client does not
  have to probe for an end.
- Ordering is always explicit. An endpoint with no `orderBy` is a review rejection, because an
  unstable order makes pagination skip and duplicate rows.

## 7. Authentication

- `Authorization: Bearer <accessToken>`.
- Access tokens expire in 15 minutes. The web client refreshes transparently and retries once on
  `TOKEN_EXPIRED`; it must not retry on `TOKEN_INVALID`.
- Switching organization returns a **new token pair**. The old pair is revoked.

## 8. Rate limiting

| Scope | Limit | Applies to |
| --- | --- | --- |
| Global | 300 / 15 min | Every request, per IP |
| Auth | 10 / 15 min | `/v1/auth/login`, per IP |
| Auth | 5 / 15 min | `/v1/auth/login`, per email |
| Password reset | 3 / hour | `/v1/auth/forgot-password`, per IP and per email |

Responses carry standard `RateLimit-*` headers and `429` with `Retry-After`.

## 9. Idempotency

Any `POST` that triggers an asynchronous or externally visible effect accepts an `Idempotency-Key`
header. The key is stored against the job execution record, so a retry with the same key is a no-op.
This is a hard requirement for Phase 8 endpoints and the reason `JOB_EXECUTION.idempotency_key` is
unique in the [ERD](./erd.md).

## 10. Layering

```
routes/index.ts            mount routers, declare required checks
  modules/<feature>/
    <feature>.routes.ts    path, method, middleware chain — nothing else
    <feature>.controller  HTTP translation only: read req, call service, shape response
    <feature>.service      business logic, transactions, tenant scoping
    <feature>.schema.ts    Zod schemas
```

Review rejections, each of which has been observed in practice:

1. Business logic in a controller. A calculation, a rounding decision or a state-transition check in a
   controller means it will be bypassed by the worker or a script.
2. `prisma.*` used outside the service layer. Bypasses tenant scoping.
3. `res.json()` called from a service. Services return values; controllers own the response.
4. An unguarded `await` in a route handler. Every async handler is wrapped in `asyncHandler`, so a
   rejected promise reaches the error middleware instead of hanging the request.
5. A raw `throw new Error()` for an expected failure. Use the `AppError` subclasses so the response
   code and stable error code are correct.
6. Logging a request body on an auth or payroll route.

## 11. Logging and audit

- Structured JSON logs via pino. Every line carries `requestId`, `level`, `msg`, `time`.
- **Never log** request or response bodies on `/auth/*` routes, nor any salary, national ID, tax PIN
  or bank account. A redaction allowlist enforces this; see `src/config/logger.ts`.
- Errors are logged at `error` with the stack; the client receives only `INTERNAL_ERROR`.
- Sensitive mutations are audit-logged through `auditService.record()` with actor, organization,
  resource, outcome and request ID. Audit writes happen **inside** the same transaction as the
  mutation, so a rolled-back mutation leaves no misleading audit entry.

## 12. Adding a route

1. Zod schemas in `<feature>.schema.ts`.
2. `requireAuth`, then `requirePermission`, then `validate`, then the handler.
3. Service method taking `principal` as its first argument, applying tenant scoping.
4. Audit record for anything sensitive, inside the transaction.
5. Tests: at least one success, one validation failure, one permission failure, one cross-tenant case.
6. Update this document if you introduced a new convention or error code.