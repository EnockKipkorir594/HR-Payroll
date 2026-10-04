# ADR-0005: Tenant isolation enforced in the service layer

- **Status:** Accepted
- **Date:** 2026-10-04
- **Phase:** 1
- **Deciders:** Lead engineer

## Context

This is a multi-tenant payroll system. A single query missing an organization filter discloses one
company's employee salaries, national ID numbers, tax PINs and payslips to another. It is the single
highest-severity defect this system can have, and unlike a calculation bug it is not self-evident:
nothing crashes, the response looks correct.

The build plan requires tenant authorization enforced server-side, and tenant-isolation tests in
Phase 10. We are deciding the mechanism now, in Phase 1, because retrofitting it after fifteen modules
have been written without it means auditing every query in the codebase.

## Decision

### Every tenant-owned table carries `organizationId`

Not just most. Every table whose rows belong to a single tenant has a non-null `organizationId` with
an index. There is no "global" tenant table holding tenant data, so there is no ambiguity about
whether a filter is required.

### Scoping happens in the service layer, not in middleware

Middleware can read the token, the path and the body. It cannot see which rows a query will touch.
Therefore:

- **Middleware** establishes `principal.organizationId` and rejects unauthenticated requests.
- **Services** apply the organization filter to every query, and assert tenant scope on every record
  they load.

The rule for module authors: a service method takes the acting `principal` as its first argument and
is the only place that may build a `where` clause on `organizationId`. There is no unscoped
repository helper exposed to controllers. Reaching for `prisma.<model>.findUnique({ where: { id } })`
from a controller is a review rejection, because `findUnique` cannot filter by tenant.

### Cross-tenant reads return 404, not 403

A request for a record belonging to another organization returns `404 NOT_FOUND`. Returning `403`
confirms the record exists, which is an enumeration oracle: an attacker can distinguish "wrong
organization" from "no such id" and map the ID space. The `DENIED` audit record captures the real
reason internally.

### Identifiers are UUIDs

Primary keys are database-generated UUIDs rather than sequential integers. Sequential IDs leak
cardinality (how many employees an organization has) and make cross-tenant guessing productive. UUIDs
also mean a client cannot infer that record 1042 exists because record 1041 was theirs.

### Defence in depth, staged

1. **Phase 1 (now):** application-layer scoping, `organizationId` columns and indexes, UUID keys,
   `404` on cross-tenant access, `DENIED` audit records.
2. **Phase 10:** PostgreSQL **row-level security** on every tenant table, with the application
   connecting as a role that cannot bypass RLS. This catches the query someone forgot, because the
   database itself returns nothing. Enabling RLS later is a migration, not a rewrite, which is why
   scoping must be correct in the application from day one — the two must agree.
3. **Phase 10:** a dedicated cross-tenant test suite that enumerates every tenant-owned route and
   asserts isolation, so a new module cannot ship without coverage.

### Defence against omission

The structural safeguards, in order of how much they catch:

- No unscoped data-access helper is exported from `@payroll/database`. The scoped client is the only
  export.
- The CI job runs `eslint-plugin-security` and the review checklist requires an explicit statement of
  the tenant filter for any new query touching a tenant table.
- Every service signature starts with `principal`, so the tenant is always in scope at the call site.

## Consequences

**Easier**

- Isolation is testable per service without a running HTTP stack.
- Adding a tenant column later is unnecessary — the constraint is already there.
- The Phase 10 RLS migration is additive.

**Harder**

- Every query needs a `where: { organizationId }`. Verbose, and that verbosity is the feature.
- Services cannot be called without a principal, which makes background jobs and seed scripts slightly
  awkward. The job worker carries a system principal with an explicit organization, so the path is the
  same one.
- `404` for cross-tenant reads makes debugging harder for developers. The audit log resolves it.

**We now owe**

- A review rule that any new `prisma.*.findUnique` outside the database package is rejected.
- RLS policies in Phase 10 that mirror the application scoping exactly.
- A cross-tenant test per module, not only the Phase 10 sweep.

## Alternatives considered

**Database-per-tenant** — Rejected for the MVP. The strongest possible isolation, but it makes
cross-organization reporting, pooled migrations and connection management substantially harder, and
we have no requirement for it. The `organizationId` column plus Phase 10 RLS reaches a strong
security posture without that operational cost.

**Schema-per-tenant** — Rejected. Same operational problem, worse: migrations fan out across schemas.

**Relying on PostgreSQL RLS from Phase 1** — Rejected as the *only* control. RLS is excellent defence
in depth but it is invisible in application code, easy to accidentally bypass with a
`SECURITY DEFINER` function or a privileged connection, and it does not help you reason about a
service method. Doing it in the application first and adding RLS later means the two layers agree and
either alone still holds.

**Middleware-only enforcement** — Rejected. This is the mistake the ADR exists to prevent. Middleware
cannot filter rows it has not queried.

**Scope by user rather than by organization** — Rejected. A user may hold memberships in several
organizations, and payroll is scoped to the employer, not the individual.