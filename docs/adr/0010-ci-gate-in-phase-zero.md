# ADR-0010: The merge gate ships in Phase 0, not Phase 11

- **Status:** Accepted
- **Date:** 2026-10-04
- **Phase:** 0
- **Deciders:** Lead engineer

## Context

The build plan allocates "GitHub Actions PR CI" to **Phase 11**, eleven phases and roughly 160 working
days after the monorepo is bootstrapped in Phase 0.

The same team of ten developers merges code continuously from Phase 2 onward. Under the plan as
written, ten developers would merge unverified code for eleven phases before any automated check runs.

The plan is not wrong about effort — the full pipeline in Phase 11 (integration suites, Docker builds,
Cloud Run staging and production, rollback) genuinely belongs late. It is wrong about the *merge gate*.
A gate that arrives at Phase 11 can only apply to code merged after Phase 11.

A merge gate is also self-enforcing in a way that is hard to retrofit. Once developers trust a gate,
they stop checking manually. Removing it later to "speed up" a deadline permanently costs more than the
delay it causes, because trust does not come back.

## Decision

**A minimal merge gate lands in Phase 0, immediately after the monorepo bootstrap. The full pipeline
still lands in Phase 11.**

Phase 0 gate — the minimum that makes every merge trustworthy:

1. `pnpm install --frozen-lockfile`
2. `pnpm lint`
3. `pnpm typecheck`
4. `pnpm test:unit`
5. `pnpm build`

Phase 1 adds `pnpm test:integration` against an ephemeral PostgreSQL service container, because
integration tests need a database that does not exist in Phase 0.

Phase 11 adds what genuinely cannot be built before there is something to deploy: Docker image builds,
container scanning, Cloud Run staging and production deployment, deployment status checks, smoke tests
and the rollback procedure.

**These five checks are required status checks in the `main` branch ruleset**, so a pull request
cannot merge until they pass. The ruleset is versioned as JSON in `.github/rulesets/` and applied
through the REST API, so protection is reviewable in a pull request rather than hidden in repository
settings.

Two supporting decisions from the same reasoning:

- **The gate is additive, never removed under deadline.** A failing check is fixed or, in a genuine
  emergency, bypassed deliberately by a named person with the reason recorded. It is never deleted to
  make a build pass.
- **CODEOWNERS, not review-count, is the control for sensitive paths.** `infra/**`,
  `packages/database/prisma/migrations/**`, `.github/workflows/**`, `docs/adr/**` and the auth and
  authorization middleware require the lead engineer's review specifically.

## Consequences

**Easier**

- Every merge from Phase 1 onward is verified. A broken build is caught in the pull request, not in
  someone else's branch two days later.
- The integration suite has eleven phases of usage to mature before Phase 11, so Phase 11 is about
  deploying, not about fixing CI for the first time.
- Vulnerabilities are gated at the moment they are introduced rather than in a periodic scan.
- Ruleset and workflow changes are themselves reviewed, because `.github/workflows/**` is CODEOWNERS
  protected.

**Harder**

- Roughly 10 to 15 minutes of CI per pull request. Developers wait. Mitigated by parallelism, by
  `paths-ignore` for documentation-only changes, and because waiting is cheaper than debugging a shared
  `main`.
- CI becomes infrastructure the whole team depends on from week one, so a broken workflow is a
  team-wide blocker. Mitigated by the workflow being CODEOWNERS protected and by keeping each job
  independently re-runnable.
- Two pipelines to maintain rather than one. Accepted: they are the same checks, extended, not
  duplicated.

**We now owe**

- Keep the workflow fast. If the gate regularly exceeds 15 minutes, caching and parallelism are a
  priority over adding more checks.
- Maintain the service containers the integration job needs.
- Document the deliberate-bypass procedure before the first genuine emergency, not during it.

## Alternatives considered

**Follow the plan exactly — no CI until Phase 11** — Rejected. It is the plan's literal instruction,
and it is the wrong sequencing for a ten-person team. The plan is a task estimate, not a dependency
order, and the build plan itself states that if work slips the schedule should move rather than
prerequisites being skipped. A merge gate is a prerequisite for safe parallel work.

**Gate only on lint and typecheck, add tests later** — Rejected. Tests are the check that catches a
broken tenant filter or a wrong rounding rule, which are precisely the defects that matter here.

**Require review approval instead of a CI gate** — Rejected as a substitute. Review catches some
things and is required too, but it is inconsistent, does not scale to every pull request, and cannot
verify that the tests actually pass.

**Set up the full Phase 11 pipeline in Phase 0** — Rejected. There is nothing to deploy or containerize
in Phase 0. Writing a Docker and Cloud Run pipeline before the first module exists means writing it
against a guess.

**Run CI on every push to `main` rather than on pull requests** — Rejected. Verification after merge
cannot block the merge and leaves `main` briefly broken for everyone.

**Use required workflows from a central repository** — Deferred. A workflow hosted in a repository the
team cannot edit is tamper-proof, which is the strongest available gate. It is deferred because the
project currently has one repository under a personal account and the rule's plan availability there
needs verifying. This is the first hardening to add once the project is hosted under an organisation.