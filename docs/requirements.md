# Requirements — Payroll Management System

Version 1.0 · Phase 0 · Kenya-first

## 1. Purpose

A payroll platform that lets an organization manage employees, compensation and payroll inputs,
calculate payroll using versioned Kenyan statutory rules, route that payroll through a controlled
approval and finalization workflow, and issue payslips and statutory reports.

The system must be **repeatable, auditable, testable, explainable and safe to change**. Where those
five qualities conflict, they win in that order.

## 2. Actors

| Actor | Description | Primary surface |
| --- | --- | --- |
| Platform administrator | Operates the platform across organizations. Not a payroll user. | Admin tooling (out of MVP scope) |
| Organization administrator | Owns one organization. Manages settings, users and roles. | Web app |
| HR manager | Manages employee master data, contracts and compensation. | Web app |
| Payroll operator | Enters payroll inputs and drafts payroll runs. | Web app |
| Payroll approver | Reviews and approves/finalizes payroll. Cannot prepare their own. | Web app |
| Auditor | Read-only access to payroll records and the audit trail. | Web app |
| Employee | Views own payslips and profile. | Web app (self-service) |

Employees never see another employee's data. Approvers never prepare the payroll they approve.

## 3. MVP scope

### In scope

1. **Organization & tenancy** — multi-organization, strict server-side tenant isolation.
2. **Employee master data** — employees, departments, employment contracts, compensation.
3. **Payroll inputs** — attendance, overtime, bonuses, commissions, deductions, adjustments.
4. **Calculation engine** — versioned, effective-dated statutory rules (PAYE, NSSF, SHIF, Housing Levy)
   plus configured earnings and deductions. Reproducible: same inputs and same rule version produce
   the same output.
5. **Payroll run lifecycle** — DRAFT → CALCULATING → CALCULATED → PENDING_APPROVAL → APPROVED →
   FINALIZED → PAID, with explicit cancellation and reversal paths.
6. **Immutability** — a finalized payroll run cannot be edited. Corrections happen only through
   recorded adjustments or reversals against a later run.
7. **Payslips & reports** — PDF payslips stored in GCP Cloud Storage, downloadable through
   authenticated or short-lived signed URLs. Payroll register, PAYE, NSSF, SHIF and P9 reports
   built from finalized data only.
8. **Automation** — BullMQ on Redis for PDF generation, email delivery and recurring schedules.
   No system cron.
9. **Audit** — every sensitive mutation recorded with actor, organization, resource, outcome,
   IP and request ID.
10. **Email** — transactional payslip and notification email through Nodemailer + SendGrid.

### Explicitly out of scope for the MVP

- Multiple currencies per organization (MVP is KES only).
- Contractor/self-employed payroll variants.
- Mobile applications.
- Multi-country statutory packs.
- Payroll approval by more than one approver level.
- Machine learning or statistical forecasting.

### Deferred, but must not be designed out

- Currency and country are stored as organization columns so a future multi-currency MVP needs a
  migration, not a rewrite.
- Roles and permissions are data, not code enums, so new roles need a seed update, not a release.

## 4. Functional requirements

### FR-1 Identity and access

| ID | Requirement |
| --- | --- |
| FR-1.1 | Users authenticate with email and password. Passwords are hashed with bcrypt, cost factor from configuration, minimum 12. |
| FR-1.2 | The API issues short-lived JWT access tokens and long-lived rotating refresh tokens. |
| FR-1.3 | A refresh token is stored only as a hash. Presenting an already-used token revokes the whole token family and forces re-authentication. |
| FR-1.4 | Every request to a protected route carries a valid access token; otherwise `401`. |
| FR-1.5 | Authorization is enforced server-side on every request using role and permission data from the database. The client never decides access. |
| FR-1.6 | A user's access token carries their active organization and role. Switching organization re-issues tokens. |
| FR-1.7 | Cross-organization access attempts return `403` and are audit-logged. |
| FR-1.8 | Login is rate limited per IP and per email to slow credential stuffing. |

### FR-2 Organization and tenancy

| ID | Requirement |
| --- | --- |
| FR-2.1 | All tenant-owned data carries an `organizationId`. |
| FR-2.2 | Every data access is scoped to the caller's active organization. Scoping is applied in the service layer, not only in middleware. |
| FR-2.3 | A user may belong to more than one organization, with an independent role in each. |
| FR-2.4 | Deactivating an organization or membership immediately denies access on the next request. |

### FR-3 Employee master data

| ID | Requirement |
| --- | --- |
| FR-3.1 | An employee belongs to exactly one organization and one department. |
| FR-3.2 | An employee has at most one active employment contract at any time. |
| FR-3.3 | Employee records are never hard-deleted once referenced by a payroll run; they are deactivated. |
| FR-3.4 | Every employee mutation is validated with Zod before reaching a service. |

### FR-4 Compensation

| ID | Requirement |
| --- | --- |
| FR-4.1 | Compensation is effective-dated. A change creates a new dated record; it does not overwrite history. |
| FR-4.2 | Money is stored with explicit decimal precision and never as a floating-point number. |
| FR-4.3 | Statutory rule versions record their official source, effective-from date and test coverage. |

### FR-5 Payroll inputs and calculation

| ID | Requirement |
| --- | --- |
| FR-5.1 | Payroll inputs are captured per employee per pay period and are immutable once the run is finalized. |
| FR-5.2 | The calculation engine is a pure function of (inputs, compensation, rule version). No database or network access inside a rule. |
| FR-5.3 | Rounding happens once, explicitly, at defined points. Intermediate results keep full precision. |
| FR-5.4 | The engine emits an itemised explanation for every amount it produces. |
| FR-5.5 | Golden tests pin the engine output for known statutory scenarios. |

### FR-6 Run lifecycle and approval

| ID | Requirement |
| --- | --- |
| FR-6.1 | A run moves only through the states defined in [workflows.md](./workflows.md). Invalid transitions are rejected. |
| FR-6.2 | The approver of a run must not be its preparer. |
| FR-6.3 | Finalization writes an immutable snapshot. Later changes are adjustments against a new run. |
| FR-6.4 | Every transition is audit-logged with actor, timestamp and previous/new state. |

### FR-7 Payslips, reports and delivery

| ID | Requirement |
| --- | --- |
| FR-7.1 | Payslip PDFs are generated in the worker and stored in Cloud Storage. |
| FR-7.2 | Payslips are served only through authenticated access or short-lived signed URLs. Bucket objects are not public. |
| FR-7.3 | Reports read finalized data only. |
| FR-7.4 | All queued jobs are idempotent. A retry must not produce a duplicate PDF, email or payroll record. |

## 5. Non-functional requirements

### FR-8 Security

| ID | Requirement |
| --- | --- |
| FR-8.1 | No secret, token or private key is committed to Git. Secret scanning push protection is enabled. |
| FR-8.2 | JWT secrets come from environment configuration supplied by a secret manager in deployed environments. |
| FR-8.3 | Payloads are validated with Zod at the boundary. Unknown fields are rejected. |
| FR-8.4 | Security headers are set on every response. |
| FR-8.5 | Sensitive payroll values are never written to logs. |
| FR-8.6 | Dependency and container scanning runs in CI. |

### FR-9 Reliability and auditability

| ID | Requirement |
| --- | --- |
| FR-9.1 | Payroll-critical writes use database transactions. |
| FR-9.2 | Audit records are append-only. No application code updates or deletes them. |
| FR-9.3 | Health and readiness endpoints exist and reflect real dependency state. |
| FR-9.4 | Structured JSON logs carry a request ID end to end. |

### FR-10 Performance

| ID | Requirement |
| --- | --- |
| FR-10.1 | An employee list query returns within 500ms at 10,000 employees for one organization. |
| FR-10.2 | A 500-employee monthly run calculates within 60 seconds. |
| FR-10.3 | List endpoints are paginated with a hard maximum page size. |

### FR-11 Maintainability

| ID | Requirement |
| --- | --- |
| FR-11.1 | Layering is routes → middleware → controllers → services → Prisma. Business logic never lives in a controller or a React component. |
| FR-11.2 | Shared contracts, validation and configuration live in workspace packages, not duplicated per app. |
| FR-11.3 | Every dependency is version-pinned. |
| FR-11.4 | Lint, typecheck, tests and build all pass before merge. |

## 6. Risks

| Risk | Impact | Mitigation |
| --- | --- | --- |
| Statutory rates change mid-project | Wrong payroll | Effective-dated, versioned rules with recorded sources; recheck before each release |
| Parallel schema edits by 10 developers | Migration conflicts, data loss | Migrations reviewed via CODEOWNERS; only one owner merges schema changes |
| Float arithmetic on money | Incorrect payouts, unreconciled totals | Decimal columns plus a money value object; golden tests |
| Cross-tenant data leak | Legal and reputational harm | `organizationId` on every tenant table, service-layer scoping, tenant-isolation tests, RLS in Phase 10 |
| Secrets committed to a public repository | Credential compromise | Push protection, `.gitignore` rules, gitleaks in CI |
| Puppeteer-sized container | Slow deploys, worker OOM | `@react-pdf/renderer` (see [ADR-0007](./adr/0007-pdf-engine-react-pdf.md)) |

## 7. MVP acceptance criteria

1. An administrator can create an organization, invite users, assign roles and manage employees.
2. Payroll can be calculated using versioned statutory and configured rules, with an itemised explanation.
3. A run can be prepared by one user and approved by a different user, then finalized immutably.
4. Payslips are generated, stored in Cloud Storage and downloadable only by entitled users.
5. Reports are produced from finalized payroll data.
6. Scheduled work runs through BullMQ and Redis, with idempotent retries.
7. Payslip email is delivered through Nodemailer and SendGrid.
8. Deployment to Cloud Run runs from GitHub Actions on an approved release.
9. Every sensitive mutation has an audit record, and cross-tenant access is blocked and logged.