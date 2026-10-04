# ADR-0001: Modular monolith in a pnpm monorepo

- **Status:** Accepted
- **Date:** 2026-10-04
- **Phase:** 0
- **Deciders:** Lead engineer

## Context

The build plan mandates a modular monolith in a pnpm monorepo, with the API, web app and background
worker as separate deployable containers rather than separate microservices.

Ten developers will work on this concurrently. Payroll is the most correctness-sensitive domain we
could choose: a mis-scoped query leaks one organization's salaries to another, and a non-reproducible
calculation produces wrong statutory filings.

We must satisfy two opposing pressures: transactionality and type safety *across* module boundaries,
and independent deployability of the web, API and worker tiers.

## Decision

We build a **modular monolith in a pnpm workspace monorepo**.

- **One deployable API process** (`apps/backend`) containing all business modules. Payroll
  calculations, finalization snapshots and approval transitions run inside PostgreSQL transactions
  against one database. This is non-negotiable for correctness and we get it for free with a single
  process.
- **Three deployable containers**, not one: `apps/backend` (API), `apps/worker` (BullMQ consumer),
  `apps/frontend` (Next.js). Each builds to its own image and scales independently.
- **Layering inside the API is mandatory and enforced in review:**
  `routes → middleware → controllers → services → Prisma`.
  Controllers translate HTTP only. Business logic lives in services. Payroll calculation logic
  belongs in neither a controller nor a React component.
- **Modules may not reach into another module's tables directly.** If module A needs module B's
  data, it calls B's service. This preserves the option of extracting a module later, which is the
  entire reason for choosing a modular monolith over a plain one.
- **Shared code lives in `apps/packages/*`** and is imported as a workspace dependency: `config`,
  `database`, `validation`, `contracts`, `test-utils`.

## Consequences

**Easier**

- Cross-module payroll transactions are ordinary local transactions. No saga, no distributed
  consistency problem, no eventual reconciliation.
- Refactoring across modules is a normal IDE operation.
- One deployment artifact for the API means one version to reason about during an incident.
- Local development needs one API process, not six.

**Harder**

- We must actively resist the "just reach into the other module's table" shortcut, or the modular
  boundaries rot and extraction stops being possible. This is a review responsibility.
- The API is a single scaling unit. A heavy report request can starve request handling. Mitigation:
  move report and PDF work to the worker tier, which is exactly what `apps/worker` is for.
- A bug in the API takes down web and API together. Accepted; Cloud Run restarts and the worker
  tier keeps running queued work.

**We now owe**

- A code review rule that rejects business logic in controllers.
- A lint or review convention keeping module boundaries intact.

## Alternatives considered

**Microservices** — Rejected. Distributed transactions around payroll finalization are a serious
correctness risk, and ten students cannot operate ten services. The build plan rules it out, and
independents agree.

**Single deployable container (API + worker + web together)** — Rejected. The worker must scale on
queue depth and the web tier must scale on traffic; they have opposite scaling profiles. Bundling them
means one slow queue stalls HTTP requests.

**Separate repositories per package** — Rejected. Cross-cutting changes to a Prisma migration plus
the service that uses it would need coordinated multi-repo PRs. Monorepo gives us atomic commits for
cross-cutting changes, which matters more at our size than repository isolation.

**npm workspaces instead of pnpm** — Rejected. Slower installs, no strict `node_modules` layout, and
weaker guarantees against phantom dependencies. pnpm's strict layout surfaces undeclared imports as
build errors, which is a security property, not just a performance one.