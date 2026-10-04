# Architecture Decision Records

An ADR captures a decision that is expensive to reverse, the alternatives considered, and what the
decision costs us. They are the reason a teammate in week 12 can understand why the code looks the
way it does.

## Format

Every ADR uses the same sections:

1. **Status** — `Proposed`, `Accepted`, `Deprecated`, `Superseded by ADR-NNNN`.
2. **Context** — the forces that made a decision necessary.
3. **Decision** — what we are doing, stated in the active voice.
4. **Consequences** — what becomes easier, what becomes harder, what we now owe.
5. **Alternatives considered** — what we rejected and why.

## Rules

- One decision per ADR. If the "Decision" section contains "and", consider splitting it.
- Never edit an accepted ADR's Context or Decision. Supersede it with a new ADR and update the
  status line. The history is the point.
- An ADR is written **before** the code. Update its status when the code lands.
- ADRs are reviewed by the lead engineer. `.github/CODEOWNERS` enforces this on `docs/adr/**`.

## Index

| ADR | Title | Status | Phase |
| --- | --- | --- | --- |
| [0001](./0001-modular-monolith-monorepo.md) | Modular monolith in a pnpm monorepo | Accepted | 0 |
| [0002](./0002-monorepo-layout-names.md) | Monorepo layout names differ from the build plan | Accepted | 0 |
| [0003](./0003-authentication-jwt-bcrypt.md) | JWT access tokens with rotating refresh tokens | Accepted | 1 |
| [0004](./0004-rbac-model.md) | Data-driven RBAC with database-backed roles and permissions | Accepted | 1 |
| [0005](./0005-tenant-isolation.md) | Tenant isolation enforced in the service layer | Accepted | 1 |
| [0006](./0006-money-precision-and-rounding.md) | Decimal money with explicit, single-point rounding | Accepted | 0 |
| [0007](./0007-pdf-engine-react-pdf.md) | `@react-pdf/renderer` for payslip PDFs | Accepted | 0 |
| [0008](./0008-phased-schema-migrations.md) | Migrate the schema one phase at a time | Accepted | 1 |
| [0009](./0009-version-pinning-and-dependency-updates.md) | Exact version pinning with Dependabot PRs | Accepted | 1 |
| [0010](./0010-ci-gate-in-phase-zero.md) | Merge gate ships in Phase 0, not Phase 11 | Accepted | 0 |

## Open decisions

Deferred deliberately. Each needs an ADR before the phase that needs it.

| Question | Needed by | Default if undecided |
| --- | --- | --- |
| Cloud Run worker execution model and concurrency settings | Phase 11 | Single worker instance, concurrency 1, session affinity off |
| Payroll run period locking and re-run semantics | Phase 6 | Lock the period on submit; re-run requires reversal |
| Golden-test fixture provenance for statutory rates | Phase 5 | Fixtures cite KRA/NSSF/SHA pages with a retrieval date |
| Report rendering: same PDF engine or a tabular export? | Phase 7 | Same engine for consistency |