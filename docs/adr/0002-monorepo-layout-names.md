# ADR-0002: Monorepo layout names differ from the build plan

- **Status:** Accepted
- **Date:** 2026-10-04
- **Phase:** 0
- **Deciders:** Lead engineer

## Context

The build plan (v1.1, §2) prescribes this layout:

```
apps/api/  apps/worker/  apps/web/
packages/{database,contracts,validation,ui,email,storage,config,test-utils}/
docs/  infra/
```

The repository already contained a different scaffold:

```
apps/backend/  apps/frontend/  apps/packages/  docs/  infrastructure/
```

At the point of this decision every file in the existing scaffold was a **zero-byte placeholder**, so
nothing of value constrained the choice.

The layout is a contract. Ten developers will import across these paths for twelve phases, and every
divergence from the plan risks an import that resolves on one person's machine and fails on another.

## Decision

**We keep the existing directory names.** The plan's names are treated as a naming preference, not a
hard requirement, and this ADR records the deviation explicitly so nobody has to guess.

Authoritative mapping from plan name to repository path:

| Build plan | This repository | Contents |
| --- | --- | --- |
| `apps/api/` | `apps/backend/` | Express + TypeScript API |
| `apps/worker/` | `apps/worker/` | BullMQ workers |
| `apps/web/` | `apps/frontend/` | Next.js frontend |
| `packages/database/` | `apps/packages/database/` | Prisma schema, client, migrations, seed |
| `packages/contracts/` | `apps/packages/contracts/` | Shared API and domain contracts |
| `packages/validation/` | `apps/packages/validation/` | Shared Zod schemas |
| `packages/config/` | `apps/packages/config/` | Shared tooling configuration |
| `packages/test-utils/` | `apps/packages/test-utils/` | Fixtures and test helpers |
| `packages/ui/` | `apps/packages/ui/` (Phase 9) | shadcn/ui components |
| `packages/email/` | `apps/packages/email/` (Phase 8) | Nodemailer + SendGrid |
| `packages/storage/` | `apps/packages/storage/` (Phase 7) | GCP Cloud Storage |
| `infra/` | `infrastructure/` | Docker and CI configuration |
| `docs/` | `docs/` | Requirements, ADRs, rules, API, runbooks |

Supporting decisions:

- `pnpm-workspace.yaml` includes both `apps/*` and `apps/packages/*` so the nested packages are
  picked up without a flat move.
- Package names are stable and unambiguous regardless of directory depth:
  `@payroll/api`, `@payroll/worker`, `@payroll/web`, `@payroll/database`, `@payroll/validation`,
  `@payroll/contracts`, `@payroll/config`, `@payroll/test-utils`.
- **Imports always use the package name**, never a relative path across a workspace boundary. A
  relative import reaching into `apps/packages/` is a review rejection, because it breaks the build
  order and hides the dependency from pnpm.

## Consequences

**Easier**

- Zero churn on the existing scaffold, which was empty anyway.
- `apps/backend` is the more precise name than `apps/api`: the process serves the API, hosts the
  middleware chain and owns the Prisma client.

**Harder**

- The repository does not match the printed plan. Anyone comparing the two will notice, which is
  exactly why this ADR exists and why the mapping table above is part of the documentation set.
- `apps/packages/` nests packages inside the app glob, which is slightly unusual for a monorepo.
- `infrastructure/` is longer to type than `infra/`. Trivial, and consistency won.

**We now owe**

- Every new document and PR description uses the mapping table, never the plan's bare names.
- Anyone quoting the build plan to a teammate must translate first.

## Alternatives considered

**Rename to match the plan exactly** — Rejected at the lead's decision. The cost is a slightly
unusual `apps/packages/` nesting; the benefit is exact plan conformance. Since both options cost
approximately nothing mechanically, the tie was broken by avoiding churn on a scaffold the team may
already have cloned, and by the lead's stated preference. Revisit if the nesting causes real friction.

**Flat `packages/` at the repository root** — Rejected. This was the cleanest structure and matches
the plan's intent most closely. It was declined because it contradicts the user's explicit decision to
keep the existing layout. Worth revisiting only if the team finds `apps/packages/` confusing in
practice.