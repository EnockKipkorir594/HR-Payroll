# Workflows and State Diagrams

Phase 0 · Authoritative for state transitions. Any code that changes a payroll run's state must
match the diagram below and update it in the same PR.

## 1. Payroll run lifecycle

```mermaid
stateDiagram-v2
    direction TB

    [*] --> DRAFT : create run (payroll_run:create)

    DRAFT --> CALCULATING : calculate (payroll_run:calculate)
    DRAFT --> CANCELLED : cancel (payroll_run:reverse)

    CALCULATING --> CALCULATED : engine succeeded
    CALCULATING --> CALCULATION_FAILED : engine error
    CALCULATION_FAILED --> CALCULATING : retry

    CALCULATED --> CALCULATING : inputs changed, recalculate
    CALCULATED --> PENDING_APPROVAL : submit (payroll_run:submit)
    CALCULATED --> CANCELLED : cancel (payroll_run:reverse)

    PENDING_APPROVAL --> APPROVED : approve (payroll_run:approve)
    PENDING_APPROVAL --> CALCULATED : reject, return to preparer
    PENDING_APPROVAL --> CANCELLED : cancel (payroll_run:reverse)

    APPROVED --> FINALIZED : finalize (payroll_run:finalize)
    APPROVED --> PENDING_APPROVAL : revoke approval

    FINALIZED --> PAID : mark paid
    FINALIZED --> REVERSED : reverse (payroll_run:reverse)
    FINALIZED --> DRAFT : correction run opened

    REVERSED --> [*]
    PAID --> [*]
    CANCELLED --> [*]

    note right of FINALIZED
        Immutable. No field of a finalized
        run may be updated. Corrections are
        recorded as adjustments against a
        later run, never as edits here.
    end note

    note right of PENDING_APPROVAL
        Segregation of duties: the preparer
        cannot approve their own run.
    end note
```

### Transition rules

| From | To | Guard | Audit event |
| --- | --- | --- | --- |
| — | `DRAFT` | Caller has `payroll_run:create` | `payroll_run.created` |
| `DRAFT` | `CALCULATING` | `payroll_run:calculate`; period not already finalized | `payroll_run.calculation_started` |
| `CALCULATING` | `CALCULATED` | Engine returned a complete, itemised result | `payroll_run.calculated` |
| `CALCULATING` | `CALCULATION_FAILED` | Engine threw; error recorded, run unchanged | `payroll_run.calculation_failed` |
| `CALCULATION_FAILED` | `CALCULATING` | Retry permitted, attempt counter incremented | `payroll_run.calculation_retried` |
| `CALCULATED` | `CALCULATING` | At least one input changed since last calculation | `payroll_run.recalculated` |
| `CALCULATED` | `PENDING_APPROVAL` | `payroll_run:submit`; no validation exceptions | `payroll_run.submitted` |
| `PENDING_APPROVAL` | `APPROVED` | `payroll_run:approve`; **approver ≠ preparer** | `payroll_run.approved` |
| `PENDING_APPROVAL` | `CALCULATED` | `payroll_run:approve` with a rejection reason | `payroll_run.rejected` |
| `APPROVED` | `FINALIZED` | `payroll_run:finalize`; snapshot written in one transaction | `payroll_run.finalized` |
| `APPROVED` | `PENDING_APPROVAL` | `payroll_run:finalize` revoked, with reason | `payroll_run.approval_revoked` |
| `FINALIZED` | `PAID` | `payroll_run:finalize`; settlement recorded | `payroll_run.paid` |
| `FINALIZED` | `REVERSED` | `payroll_run:reverse`; reason mandatory | `payroll_run.reversed` |
| any non-terminal | `CANCELLED` | `payroll_run:reverse`; reason mandatory | `payroll_run.cancelled` |

Terminal states are `PAID`, `REVERSED` and `CANCELLED`. A run in a terminal state has no outgoing
transition except opening a correction run.

## 2. Authentication and token rotation

```mermaid
sequenceDiagram
    autonumber
    participant C as Client
    participant A as Express API
    participant D as PostgreSQL

    C->>A: POST /v1/auth/login { email, password }
    A->>A: rate limit (per IP and per email)
    A->>A: bcrypt.compare(passwordHash, password)
    alt credentials invalid
        A-->>C: 401 INVALID_CREDENTIALS
    end
    A->>D: load user + active membership
    alt no active membership
        A-->>C: 403 NO_ACTIVE_MEMBERSHIP
    end
    A->>D: insert RefreshToken { familyId, tokenHash, expiresAt }
    A-->>C: 200 { accessToken (15m), refreshToken (7d), user, organization, role }

    Note over C,A: access token expires
    C->>A: POST /v1/auth/refresh { refreshToken }
    A->>D: find token by hash
    alt token already used (revoked, family intact)
        A->>D: revoke entire family  ← reuse detected
        A-->>C: 401 REFRESH_TOKEN_REUSED
    end
    A->>D: revoke presented token, insert successor in same family
    A-->>C: 200 { new accessToken, new refreshToken }
```

Rules encoded above:

- Access tokens are short-lived (15m) and stateless. Refresh tokens are long-lived (7d), stored
  **hashed**, and rotate on every use.
- Replay of a rotated refresh token revokes its whole family and is audit-logged as
  `auth.refresh_reuse_detected`. This is how token theft is caught.
- A 401 on refresh never reveals whether the email exists.

## 3. Organization switching

A user may hold memberships in several organizations. The active organization is a token claim, not
a request parameter, so it cannot be spoofed per request.

```mermaid
sequenceDiagram
    autonumber
    participant C as Client
    participant A as Express API
    participant D as PostgreSQL

    C->>A: GET /v1/auth/me  (Bearer access token)
    A->>A: verify signature, expiry, algorithm
    A->>D: load memberships for principal
    A-->>C: 200 { user, activeOrganization, role, availableOrganizations[] }

    C->>A: POST /v1/auth/switch-organization { organizationId }
    A->>A: require membership, status ACTIVE
    alt membership missing or suspended
        A->>D: audit auth.switch_organization DENIED
        A-->>C: 403 NOT_A_MEMBER
    end
    A->>D: rotate refresh token (new family)
    A-->>C: 200 { new accessToken carrying orgId, new refreshToken }
```

## 4. Tenant isolation on a data request

```mermaid
sequenceDiagram
    autonumber
    participant C as Client
    participant M as Middleware
    participant S as Service layer
    participant D as PostgreSQL

    C->>M: GET /v1/employees/:id (Bearer token)
    M->>M: verify JWT → principal { userId, organizationId, role, permissions }
    M->>M: requirePermission('employee:read')
    M->>S: handler(principal)
    S->>S: assertTenantScope(record.organizationId, principal.organizationId)
    alt record belongs to another organization
        S->>D: audit employee.read DENIED
        S-->>C: 404 NOT_FOUND  ← 404 not 403, do not confirm existence
    end
    S->>D: query filtered by organizationId
    S-->>C: 200 { data }
```

Two deliberate choices:

- **Authorization is applied in the service layer, not only in middleware.** Middleware cannot see
  which rows a query will touch, so middleware alone is not tenant isolation.
- **A cross-tenant read returns `404`, not `403`.** Returning `403` confirms the record exists,
  which is an enumeration leak.

## 5. Queued job execution

```mermaid
stateDiagram-v2
    direction LR

    [*] --> QUEUED : enqueue with idempotencyKey
    QUEUED --> ACTIVE : worker picks up
    ACTIVE --> DONE : handler succeeds
    ACTIVE --> RETRYING : transient failure
    RETRYING --> ACTIVE : backoff elapsed, attempts remain
    RETRYING --> FAILED : attempts exhausted
    ACTIVE --> FAILED : permanent failure
    DONE --> [*]
    FAILED --> [*]

    note right of ACTIVE
        The handler records the idempotency key
        before performing side effects. A retry
        with a known key is a no-op, so a
        duplicate PDF or email is impossible.
    end note

    note right of RETRYING
        Transient: timeout, connection reset,
        5xx. Permanent: validation failure,
        missing record. Only transient errors retry.
    end note
```

## 6. Phase 2+ workflows

Department, employee, contract and compensation workflows are specified in their own phase. When
they are written, they follow the same shape: a state diagram in this file, transition rules as a
table, guards naming the required permission, and an audit event per transition.