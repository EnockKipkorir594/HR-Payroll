# Payroll Platform — Documentation Index

Reference documentation for the Kenya-first payroll management system. Source of truth lives in
this folder; code changes that contradict a document must update the document in the same PR.

| Document | Purpose | Phase |
| --- | --- | --- |
| [requirements.md](./requirements.md) | Product scope, actors, functional and non-functional requirements | 0 |
| [glossary.md](./glossary.md) | Payroll and platform terminology, single source of truth for naming | 0 |
| [rbac-matrix.md](./rbac-matrix.md) | Roles, permissions and who may do what | 0 |
| [workflows.md](./workflows.md) | State diagrams for payroll runs, authentication and tenancy | 0 |
| [erd.md](./erd.md) | Entity relationship diagram for the whole domain | 0 |
| [api-conventions.md](./api-conventions.md) | Request/response envelope, errors, pagination, versioning | 1 |
| [adr/](./adr/) | Architecture Decision Records | 0+ |
| [runbooks/](./runbooks/) | Operational procedures (rollback, backup/restore) | 11 |

## Phase status

| Phase | Name | Status |
| --- | --- | --- |
| 0 | Product discovery & architecture | Complete |
| 1 | Platform foundation | Complete |
| 2 | Organization & employees | Not started |
| 3 | Employment & compensation | Not started |
| 4 | Payroll inputs | Not started |
| 5 | Calculation engine | Not started |
| 6 | Payroll run, approval & finalization | Not started |
| 7 | Payslips & reporting | Not started |
| 8 | Jobs, scheduling & email | Not started |
| 9 | Frontend & self-service | Not started |
| 10 | Security & hardening | Not started |
| 11 | CI/CD & release infrastructure | Partial (PR gate landed; deploy pending) |
| 12 | Final QA & MVP release | Not started |

## Documentation rules

1. Architecture changes require an ADR. See [adr/README.md](./adr/README.md).
2. Schema changes require a reviewed Prisma migration. No direct production edits.
3. Statutory rule changes require the official source and an effective date recorded alongside the code.
4. Every feature PR updates the docs it affects in the same pull request.