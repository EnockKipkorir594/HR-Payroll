# Roles and Permissions Matrix

Phase 0 · Authoritative for authorization. The seed data in `apps/packages/database/prisma/seed.ts`
is generated from this document — if the two disagree, the seed is wrong.

## 1. Roles

| Role key | Scope | Purpose |
| --- | --- | --- |
| `SUPER_ADMIN` | Platform | Operates the platform. Grants every permission implicitly and is not an employee of any payroll. Never assigned as an employment role. |
| `ORG_ADMIN` | Organization | Owns one organization: settings, users, roles, departments. |
| `HR_MANAGER` | Organization | Owns employee master data, contracts and compensation. |
| `PAYROLL_OPERATOR` | Organization | Prepares payroll: enters inputs, creates and calculates runs. |
| `PAYROLL_APPROVER` | Organization | Reviews, approves and finalizes runs. Cannot prepare what they approve. |
| `AUDITOR` | Organization | Read-only across all records including the audit trail. |
| `EMPLOYEE` | Organization | Self-service: own profile, own payslips, own reports. |

Roles are **data, not code**. `Role` and `Permission` are database tables seeded from this
document, so a new role is a seed update rather than a release.

## 2. Permission catalogue

Permissions are `resource:action`. `🔒` marks a sensitive permission: it is always audit-logged and
is a candidate for step-up authentication.

| Permission | 🔒 | Description |
| --- | :---: | --- |
| `organization:read` | | View organization settings |
| `organization:update` | 🔒 | Change organization settings |
| `department:read` | | List departments |
| `department:create` | | Create departments |
| `department:update` | | Rename or restructure departments |
| `department:archive` | | Archive a department |
| `employee:read` | | View employee records |
| `employee:create` | | Add employees |
| `employee:update` | | Change employee master data |
| `employee:deactivate` | 🔒 | Deactivate an employee |
| `employment:read` | | View employment contracts |
| `employment:create` | | Create employment contracts |
| `employment:update` | | Amend contract terms |
| `employment:terminate` | 🔒 | End an employment contract |
| `compensation:read` | | View compensation |
| `compensation:create` | | Create compensation records |
| `compensation:update` | 🔒 | Change effective-dated compensation |
| `payroll_input:read` | | View payroll inputs |
| `payroll_input:create` | | Record payroll inputs |
| `payroll_input:update` | | Amend payroll inputs before submission |
| `payroll_run:read` | | View payroll runs |
| `payroll_run:create` | | Create a draft run |
| `payroll_run:calculate` | | Trigger calculation |
| `payroll_run:submit` | | Submit for approval |
| `payroll_run:approve` | 🔒 | Approve or reject a run |
| `payroll_run:finalize` | 🔒 | Finalize a run immutably |
| `payroll_run:reverse` | 🔒 | Reverse or cancel a run |
| `payslip:read_own` | | View own payslips |
| `payslip:read_any` | 🔒 | View any employee's payslips |
| `payslip:generate` | | Generate payslip PDFs |
| `report:read` | | View reports |
| `report:generate` | | Produce statutory reports |
| `user:read` | | List users and memberships |
| `user:invite` | | Invite users to the organization |
| `user:deactivate` | 🔒 | Deactivate a user |
| `user:assign_role` | 🔒 | Change a user's role |
| `role:read` | | View roles and permissions |
| `role:manage` | 🔒 | Create or change roles and permissions |
| `audit:read` | | Read the audit trail |

## 3. Grant matrix

`●` granted · `—` not granted

| Permission | SUPER_ADMIN | ORG_ADMIN | HR_MANAGER | PAYROLL_OPERATOR | PAYROLL_APPROVER | AUDITOR | EMPLOYEE |
| --- | :---: | :---: | :---: | :---: | :---: | :---: | :---: |
| `organization:read` | ● | ● | ● | ● | ● | ● | — |
| `organization:update` | ● | ● | — | — | — | — | — |
| `department:read` | ● | ● | ● | ● | ● | ● | — |
| `department:create` | ● | ● | ● | — | — | — | — |
| `department:update` | ● | ● | ● | — | — | — | — |
| `department:archive` | ● | ● | ● | — | — | — | — |
| `employee:read` | ● | ● | ● | ● | ● | ● | own only |
| `employee:create` | ● | ● | ● | — | — | — | — |
| `employee:update` | ● | ● | ● | — | — | — | — |
| `employee:deactivate` | ● | ● | ● | — | — | — | — |
| `employment:read` | ● | ● | ● | ● | ● | ● | own only |
| `employment:create` | ● | ● | ● | — | — | — | — |
| `employment:update` | ● | ● | ● | — | — | — | — |
| `employment:terminate` | ● | ● | ● | — | — | — | — |
| `compensation:read` | ● | ● | ● | ● | ● | ● | own only |
| `compensation:create` | ● | ● | ● | — | — | — | — |
| `compensation:update` | ● | ● | ● | — | — | — | — |
| `payroll_input:read` | ● | ● | ● | ● | ● | ● | own only |
| `payroll_input:create` | ● | ● | ● | ● | — | — | — |
| `payroll_input:update` | ● | ● | ● | ● | — | — | — |
| `payroll_run:read` | ● | ● | ● | ● | ● | ● | own only |
| `payroll_run:create` | ● | ● | ● | ● | — | — | — |
| `payroll_run:calculate` | ● | ● | ● | ● | — | — | — |
| `payroll_run:submit` | ● | ● | ● | ● | — | — | — |
| `payroll_run:approve` | ● | ● | — | — | ● | — | — |
| `payroll_run:finalize` | ● | ● | — | — | ● | — | — |
| `payroll_run:reverse` | ● | ● | — | — | ● | — | — |
| `payslip:read_own` | ● | ● | ● | ● | ● | ● | ● |
| `payslip:read_any` | ● | ● | ● | ● | ● | ● | — |
| `payslip:generate` | ● | ● | ● | ● | ● | — | — |
| `report:read` | ● | ● | ● | ● | ● | ● | own only |
| `report:generate` | ● | ● | ● | ● | ● | — | — |
| `user:read` | ● | ● | ● | — | — | — | — |
| `user:invite` | ● | ● | — | — | — | — | — |
| `user:deactivate` | ● | ● | — | — | — | — | — |
| `user:assign_role` | ● | ● | — | — | — | — | — |
| `role:read` | ● | ● | — | — | — | ● | — |
| `role:manage` | ● | ● | — | — | — | — | — |
| `audit:read` | ● | ● | — | — | — | ● | — |

**"own only"** is not a grant in the permission table. It is enforced by
`scopedToSelf` behaviour in the service layer: a caller holding `employee:read` without
`employee:read_any` is restricted to rows where they are the linked employee. No self-scoping
permission key exists, because a client cannot be trusted to omit an ID.

## 4. Segregation of duties

Two constraints cannot be expressed as a grant and are enforced in code:

1. **Preparer ≠ approver.** The user who submits a payroll run cannot be the user who approves,
   finalizes or reverses it. Enforced by `payrollRun.service`, tested in the authorization suite.
   `PAYROLL_OPERATOR` deliberately has no `payroll_run:approve`; `PAYROLL_APPROVER` deliberately has
   no `payroll_run:create`.

2. **Nobody grants themselves a role.** A caller cannot assign a role they do not themselves hold.
   Enforced in the user service and covered by an authorization test.

## 5. Denominator rule

`SUPER_ADMIN` has `isSuperAdmin = true` and bypasses the permission table entirely. This is the only
role that does, it is never granted through a UI, and every action it takes is still audit-logged.
The bypass exists for platform operations such as recovery, not as a convenience.

## 6. Adding a role or permission

1. Update this document.
2. Add the permission key to the catalogue and seed in `apps/packages/database/prisma/seed.ts`.
3. Add it to the grant matrix.
4. If it is sensitive, mark `isSensitive: true` so it is audit-logged.
5. If it is self-scoped, add `scopedToSelf` handling to the relevant service.
6. Open a PR. `.github/CODEOWNERS` requires a review from the lead engineer on this file.