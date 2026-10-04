# ADR-0009: Exact version pinning with Dependabot pull requests

- **Status:** Accepted
- **Date:** 2026-10-04
- **Phase:** 1
- **Deciders:** Lead engineer

## Context

The build plan requires that dependency versions be pinned, and that dependency and container checks
run in CI/CD.

Ten developers on a payroll system means an unpinned dependency is a supply-chain exposure and a
reproducibility problem at the same time: two developers on the same commit can produce different
installs, so a bug that reproduces for one person does not reproduce for another. That is expensive
during an incident and it makes security response slow, because nobody can be certain what is
installed.

It also bites in reverse. The build plan quotes dependency documentation generically ("match each
reference to the dependency version pinned in the repository"), which only works if there is a single
pinned version.

## Decision

### Pin exactly, in package.json

- Runtime and development dependencies use **exact versions with no range**: `"7.10.0"`, not `"^7.10.0"`.
- A dependency version changes only through a pull request, reviewed and merged like any other change.
- `pnpm-lock.yaml` is committed and is the authoritative resolution. CI installs with `--frozen-lockfile`,
  which fails if `package.json` and the lockfile disagree.

### Dependabot opens the upgrade pull requests

We do not hand-check for advisories. `.github/dependabot.yml` watches `npm` (the repository root and
`/apps/*`), `github-actions` and `docker`, on a weekly schedule, with:

- `patch` and `minor` updates **grouped into a single pull request** per ecosystem.
- `major` updates opened individually, because a major version can change behaviour.
- `security` updates opened immediately rather than batched.
- Versioning strategy `increase-if-necessary`, so Dependabot widens a range if a dependency requires it.

Grouping matters more than it looks. Ten developers each merging individual Dependabot PRs is a
reliable source of merge conflicts and of unreviewed version drift; one grouped PR per week is one
review.

### CI is the enforcement, not the convention

- `pnpm install --frozen-lockfile` fails on an out-of-sync lockfile.
- `pnpm audit --audit-level=high` runs in CI and is a **required status check**.
- `actions/dependency-review-action` runs on every pull request and blocks a PR that introduces a
  dependency with a known vulnerability or a license on the deny list.
- `gitleaks/gitleaks-action` scans every pull request for secrets.
- Dependabot and `gitleaks` are pinned to commit SHAs, and Dependabot's `github-actions` ecosystem
  keeps those SHAs current.

## Consequences

**Easier**

- Installs are reproducible. "Works on my machine" becomes debuggable rather than mysterious.
- Every version change is reviewed, dated and attributable in git history.
- Upgrades arrive as reviewed pull requests on a schedule, so security patches do not depend on anyone
  remembering to check.
- `--frozen-lockfile` in CI means a developer cannot accidentally install something different.

**Harder**

- Developers cannot `npm install <pkg>` and get an unpinned new package silently; they add the exact
  version.
- Manual dependency bumps create conflicts with Dependabot PRs. Mitigated by batching manual bumps into
  the weekly window.
- A grouped PR touching forty packages is harder to review than forty single-package PRs. Accepted: the
  review question becomes "does CI still pass and did anything change behaviour", which the golden test
  suite answers.

**We now owe**

- Weekly triage of Dependabot PRs, with a named owner.
- Adding `docker` to Dependabot only once the Dockerfiles exist (Phase 11).
- Keeping the license deny list current.
- Re-checking that a dependency's pinned major version still matches the documentation the build plan
  cites, at each phase boundary.

## Alternatives considered

**Caret ranges (`^7.10.0`)** — Rejected. The usual default, and defensible for an application. Rejected
here because patch-level drift changes what two developers install from the same commit, which
undermines both reproducibility and security response.

**Dependabot only, no pinning** — Rejected. Automatic updates with no pin make the lockfile the only
constraint, and `pnpm install` without `--frozen-lockfile` can silently resolve differently.

**Renovate instead of Dependabot** — Rejected for now. Renovate's grouping and auto-merge are better,
and its `dependencyDashboard` gives clearer triage. Rejected because Dependabot is GitHub-native,
works with repository rulesets and required status checks without extra configuration, and has no
separate account or token to manage. Worth reconsidering once automated merge for non-breaking patch
updates is worth the extra configuration.

**A scheduled manual audit (a person runs `pnpm audit` every week)** — Rejected. It depends on someone
remembering, and it produces a report rather than a fix. The automation must propose the fix.

**Snyk or another commercial scanner** — Rejected. Budget, and it duplicates what CodeQL, the
dependency review action and Dependabot already cover for a JavaScript/TypeScript stack. Revisit if the
project is no longer a school project.

**Auto-merge patch updates with no review** — Rejected for now. It would be safe once the golden test
suite covers the calculation engine and the build is green. Deferred deliberately, and it is the first
optimization to turn on after Phase 12.