# ADR-0008: Migrate the Prisma schema one phase at a time

- **Status:** Accepted
- **Date:** 2026-10-04
- **Phase:** 1
- **Deciders:** Lead engineer

## Context

The build plan's Phase 0 deliverable is a **domain model and ERD covering the whole system**, and
Phase 1 delivers "PostgreSQL + Prisma setup/migrations".

That creates a decision: does the initial migration contain every table for all twelve phases, or only
the tables Phase 1 actually needs?

Ten developers will be writing migrations in parallel from Phase 2 onward. Prisma migrations are
**append-only, sequential and strictly ordered** — two developers generating a migration against the
same schema directory produce two directories with the same timestamp prefix and a merge that requires
manual repair.

The full domain model is documented in [docs/erd.md](../erd.md), so teams in Phases 2 to 7 know the
names, keys and cardinalities they must build against even before those tables exist.

## Decision

**The initial migration contains only Phase 1 tables. Each later phase adds its own migration.**

Phase 1 ships: `Organization`, `User`, `OrganizationMembership`, `Role`, `Permission`,
`RolePermission`, `RefreshToken`, `AuditLog`.

Later phases add their own tables — departments and employees in Phase 2, contracts and compensation
in Phase 3, and so on.

Supporting rules:

1. **The ERD is the contract.** Table names, column names, keys and cardinalities for all phases are
   fixed in `docs/erd.md`. A developer in Phase 4 codes against that contract, so integration in Phase
   6 is a formality rather than a negotiation.
2. **One migration per pull request.** A PR that changes the schema contains exactly one migration
   directory. Two migrations in one PR cannot be reviewed coherently.
3. **`packages/database` is CODEOWNERS-protected.** Only the lead engineer merges a pull request that
   touches `apps/packages/database/prisma/migrations/**`. This is the single most effective
   protection against migration ordering conflicts.
4. **Migrations are hand-reviewable.** A PR adding a `DROP COLUMN` or a non-nullable column to a
   populated table needs an explicit migration plan in the PR description, because PostgreSQL
   operations are not transactional across large tables.
5. **No `prisma migrate dev` in CI against production.** CI validates with `migrate deploy` against an
   ephemeral database, and validates the schema with `prisma validate` and `prisma format --check`.

## Consequences

**Easier**

- Migration ordering conflicts essentially disappear, because one owner merges them and one PR carries
  one migration.
- The initial migration is small enough to review properly, which matters most for the migration that
  shapes every table.
- Teams build against a documented contract rather than guessing, so the ERD does real work.
- A mistaken Phase 2 design is caught in Phase 2, by the people building it, rather than in Phase 1 by
  someone guessing what they need.

**Harder**

- The schema is not complete up front, so cross-phase integration is not proven until those phases run.
  Mitigated by the ERD contract.
- `prisma migrate dev` output must be regenerated rather than reviewed as a whole-graph artifact.
- A developer must read the ERD rather than the schema to discover later-phase tables.

**We now owe**

- The ERD stays current. It is a living contract, not a Phase 0 artifact that is written once.
- A note in each phase's task description pointing at the ERD section that phase implements.

## Alternatives considered

**Migrate the whole domain up front in Phase 1** — Rejected. Ten developers would be guessing at
requirements they do not own, so fields would be wrong and migrations rewritten. The one table that is
worth getting exactly right early is the audit log, because retrofitting audit onto an existing schema
is a backfill, whereas building it in means every mutation since day one is recorded.

**Merge all migrations through a single squashed baseline at each phase boundary** — Rejected. It
guarantees the conflict scenario we are trying to avoid, and squashing applied migrations breaks the
chain that deployed environments are following.

**Database-per-phase schemas** — Rejected. Multiple schemas in one database multiply connection
management and make the tenant story harder to reason about, for no isolation benefit.

**Skip Prisma migrations and use `db push`** — Rejected. `db push` has no migration history, so
environments drift and there is no record of what a deployed schema was. The build plan requires
reviewed migrations.

**Let any developer merge migrations, with conflict resolution left to git** — Rejected. Prisma
migration directories are ordered by timestamp. Concurrent generation is a routine source of a
corrupted migration history, and it fails in staging rather than in review.