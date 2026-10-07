
---

# `dependency-policy.md`

This one directly addresses the problem you originally raised.

```md
# Dependency Policy

## 1. Package Manager

The project uses pnpm.

Developers must not use npm or yarn to install project dependencies.

Use:

```bash
pnpm install
```

##  2. Lockfile

The repository must contain:
```text
pnpm-lock.yaml
```
The lockfile must be committed to Git.

Developers must not manually edit the lockfile.

##  3. Adding Dependencies

Before adding a dependency, ask:

-  Do we actually need it?
-  Does an existing dependency already solve the problem?
-  Is the dependency actively maintained?
-  Does it introduce unnecessary security or maintenance risk?

##  4. Installing Dependencies

Add dependencies using pnpm.

For example:
```bash
pnpm add package-name --filter backend
```

or the appropriate workspace package.

Do not manually modify dependency versions in the lockfile.

## 5. Dependency Versions

Dependency versions should be intentionally selected and reviewed.

Do not randomly upgrade dependencies during feature development.

Dependency upgrades should be separate pull requests whenever practical.

##  6. Runtime Versions

The project defines its Node.js version.

Developers must use the project's declared version.

Check:
```bash
node --version
```
before troubleshooting environment-related problems.

##  7. Security

Dependencies with known serious security vulnerabilities must be investigated and addressed.

Do not ignore security warnings without understanding their impact.

##  8. Production Dependencies

Do not add development-only tools as production dependencies.

Keep runtime dependencies and development dependencies correctly separated.

##  9. Dependency Upgrades

Dependency upgrades should include:

-  Reason for upgrade
-  Compatibility check
-  Test verification
-  Review

Large dependency upgrades should not be mixed with unrelated feature work.