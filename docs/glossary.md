# Glossary

Single source of truth for domain and platform terminology. If a PR introduces a new domain term,
add it here in the same PR.

## Business domain

**Adjustment** — A correction applied to a finalized payroll. Adjustments are recorded against a
later payroll run and always carry a reason. Finalized payroll is never edited in place.

**Basic salary** — The fixed gross monthly earnings component of an employee's compensation.

**Deduction** — An amount withheld from gross earnings: statutory (PAYE, NSSF, SHIF, Housing Levy) or
configured (pension, insurance, advance, disciplinary).

**Earning** — An amount added to gross pay: basic salary, allowance, overtime, bonus or commission.

**Effective-dated** — A record that carries the date from which it applies, rather than being
overwritten. Resolving "what was true on date X" must always be possible.

**Finalized** — A payroll run whose results are immutable. Corrections require adjustments or
reversals. This is the central guarantee of the system.

**Gross pay** — Total earnings before any deductions.

**Net pay** — Gross pay minus all deductions. The amount paid to the employee.

**Pay period** — The window of time a payroll run covers, identified by start and end date and a
pay-cycle label such as `2026-10-MONTHLY`.

**Payroll run** — One calculation and approval cycle for one pay period in one organization.

**Payroll input** — A per-employee, per-period fact that feeds the calculation: attendance days,
overtime hours, bonus, commission, or an adjustment. Inputs are captured separately from
compensation so history stays auditable.

**Reversal** — The cancelling counterpart of a finalized run, modelled explicitly rather than as an
edit.

**Rule version** — An immutable, effective-dated set of statutory or configured calculation rules.
A calculation always records which version it used, so a past run can always be re-explained.

**Statutory rule** — A rule mandated by Kenyan law or regulator (KRA, NSSF, SHA). Must record its
official source and effective date.

## Money and precision

**Minor unit** — The smallest stored denomination. All monetary columns are stored as decimal
strings/numbers in the database and manipulated through the money value object. Floating-point
types are forbidden for money.

**Rounding** — Applied exactly once, at defined points in the calculation, with the mode recorded
alongside the rule version. Intermediate values keep full precision.

## Identity and access

**Active organization** — The organization a request is currently acting within. Carried in the
access token and enforced on every data access.

**Audit log** — An append-only record of a sensitive action: actor, organization, action, resource,
outcome, IP, user agent, request ID and timestamp.

**Membership** — A user's association with an organization, carrying an independent role. A user may
hold several memberships.

**Permission** — A single atomic grant such as `payroll:approve` or `employee:read`. Roles are
bundles of permissions.

**RBAC** — Role-Based Access Control. Authorization is decided from database role and permission
data, never from client-supplied values.

**Refresh token family** — The set of refresh tokens descended from one login. Reuse of a rotated
token revokes the entire family, which detects theft.

**Step-up authentication** — Re-entering credentials before a high-risk action. Tracked as a
permission attribute; enforcement lands with sensitive operations.

## Tenancy

**Tenant** — An organization and all of its data. Tenant isolation is enforced server-side on every
query.

**Tenant-scoped** — A record that belongs to exactly one tenant and carries its `organizationId`.

## Platform

**Monorepo** — A single repository holding all applications and shared packages, managed with pnpm
workspaces.

**Modular monolith** — One deployable API whose internals are separated into modules, rather than a
set of network services. Modules may not bypass each other's service layer.

**Workspace package** — A shared library under `apps/packages/` published to the workspace rather
than to a registry.

**Adapter** — The driver that connects Prisma Client to a database. Prisma 7 requires a driver
adapter for every database.

**Golden test** — A test that pins an exact expected output for a known input, used to prove the
calculation engine has not changed behaviour.

**Idempotency** — A job that can run more than once with the same result and no duplicate side
effects. Mandatory for every queued job.

## State

**DRAFT** — A payroll run being prepared. Mutable.

**CALCULATING** — Calculation in progress.

**CALCULATED** — Calculation complete, awaiting submission.

**PENDING_APPROVAL** — Submitted and awaiting an approver other than the preparer.

**APPROVED** — Approved, awaiting finalization.

**FINALIZED** — Immutable snapshot written.

**PAID** — Settlement recorded.

**CANCELLED / REVERSED** — Terminal states reached through an explicit, audited path.