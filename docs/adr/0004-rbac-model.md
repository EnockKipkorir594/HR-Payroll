# ADR-0004: Data-driven RBAC with database-backed roles and permissions

- **Status:** Accepted
- **Date:** 2026-10-04
- **Phase:** 1
- **Deciders:** Lead engineer

## Context

The build plan requires an RBAC matrix and RBAC middleware. Payroll approval carries segregation-of-duty
requirements: the person who prepares a run must not be the person who approves it.

With ten developers and a domain that will grow, authorization must be changeable without a
redeploy, and it must be impossible for a controller to quietly grant itself a permission.

The authoritative catalogue is [docs/rbac-matrix.md](../rbac-matrix.md).

## Decision

### Roles and permissions are data

`Role`, `Permission` and `RolePermission` are **database tables**, seeded from the matrix document —
not TypeScript enums or a hardcoded map.

- `Permission.key` is `resource:action`, globally unique, e.g. `payroll_run:finalize`.
- `Permission.is_sensitive` marks permissions that always require an audit record and are candidates
  for step-up authentication.
- `Permission.is_self_scoped` marks read permissions that restrict the caller to their own records
  rather than the whole organization.
- `Role.bypasses_permissions` exists for `SUPER_ADMIN` only. It is the sole permission bypass in the
  system, it cannot be granted through any UI, and actions taken under it are still audited.

The cost of a new role is a seed change and a migration, not a code change and a release.

### Three layers of enforcement

1. **`requireAuth`** verifies the access token and populates `req.principal` with
   `{ userId, organizationId, roleKey, tokenId }`.
2. **`requirePermission('payroll_run:finalize')`** resolves the principal's role to a permission set
   and rejects with `403` when absent.
3. **Service-layer scoping** narrows *which rows* the caller may touch. This is the layer that
   actually prevents cross-tenant and self-scope violations, and it is not optional — see
   [ADR-0005](./0005-tenant-isolation.md).

Permissions are resolved from the database and cached in-process with a short TTL, invalidated by role
and membership changes. The cache is a performance measure only; a stale entry can only widen access
for the duration of the TTL, so the TTL is short and every sensitive permission is re-read from the
database at the service layer rather than trusted from cache.

### Sensible defaults

- **Deny by default.** A route with no `requirePermission` is only reachable if it also has no
  `requireAuth`, and the router asserts at startup that any route reaching a service is behind
  `requireAuth`. Unprotected routes are an explicit, greppable list.
- **Fail closed.** A database error during permission resolution returns `403`, never "allow".
- **No permission checks in the database layer.** Authorization belongs to the service layer where
  business context exists. Row-level security in PostgreSQL is a Phase 10 defence-in-depth addition,
  not the primary control.

### Segregation of duties is code, not grants

Two constraints cannot be expressed as a grant and are enforced explicitly:

1. **Preparer ≠ approver.** `payrollRun.service` rejects approval, finalization or reversal of a run
   by the user who prepared it. The role matrix makes this structural by withholding
   `payroll_run:approve` from `PAYROLL_OPERATOR`, but the service check exists independently because a
   user may hold two roles.
2. **No self-granting.** A caller cannot assign a role they do not themselves hold, and cannot modify
   their own membership. Both are enforced in the user service.

### Audit

Every sensitive-permission action is written to `audit_logs` with actor, organization, role at time of
action, resource, outcome, IP, user agent and request ID. `DENIED` outcomes are recorded too:
repeated denials are an attack signal, and an audit trail that only records successes cannot show an
attempt.

## Consequences

**Easier**

- New roles and permission tuning ship as a seed update.
- The grant matrix is readable by non-engineers, so onboarding does not require reading middleware.
- `403`s become explainable: the log records exactly which permission was missing.

**Harder**

- Every request that reaches a service needs a permission decision, which needs the principal. This is
  friction for the simplest CRUD endpoints.
- Permission resolution touches the database, hence the short-lived cache and its invalidation logic.
- Cache invalidation is a source of subtle bugs: too long a TTL and a revoked role keeps access; too
  short and we lose the benefit. The TTL is therefore short and every sensitive permission is
  re-verified at the service layer, so a stale cache can never authorize a high-risk action.
- Adding a permission means updating three places: the matrix document, the seed, and the route. This
  is deliberate friction; it makes an unreviewed permission grant hard.

**We now owe**

- An authorization test per permission, including a cross-organization case.
- A CI check that the seed and `rbac-matrix.md` agree, added once the matrix grows.

## Alternatives considered

**TypeScript enums and a hardcoded permission map** — Rejected. Every permission change becomes a
release, the matrix and the code can drift apart silently, and adding a role requires touching
application code.

**Full ABAC with policy-as-code (Open Policy Agent, Casbin)** — Rejected for now. More expressive, and
the right answer if we need contextual rules such as "approvers may only approve runs in their own
department". That requirement has not been stated, and OPA adds a runtime dependency and a second
language to learn. The service-layer checks remain the right place for such rules if they arrive, so
this is revisitable without rework.

**Checking permissions only in controllers** — Rejected. Controllers are HTTP translation and are the
first thing a developer routes around under deadline. A service callable from a worker, a script or a
future controller must be safe on its own terms.

**Permissions on the access token** — Rejected. It would remove a database read but makes revocation
impossible until expiry, and puts a large permission list in every token. The token carries only the
role key; resolution happens server-side.