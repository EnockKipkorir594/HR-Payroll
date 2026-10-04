# ADR-0003: JWT access tokens with rotating, hashed refresh tokens

- **Status:** Accepted
- **Date:** 2026-10-04
- **Phase:** 1
- **Deciders:** Lead engineer

## Context

The build plan mandates JWT + bcrypt, following an existing internal pattern as a reference while
never reusing its secrets, keys or environment values.

Payroll is high-value target data: an account that reads the audit log or a finalized payslip exposes
one organization's salaries and identity documents. Our threat model must assume tokens leak — through
a stolen laptop, a shared CI variable, a proxy log, or a team member's browser.

Password storage follows the [OWASP Password Storage Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html).
Token format follows [RFC 7519](https://www.rfc-editor.org/rfc/rfc7519).

## Decision

### Passwords

- Hash with **bcrypt** (via `bcryptjs`, a pure-JS implementation with no native build step), cost
  factor from `BCRYPT_SALT_ROUNDS`, **default 12**.
- Never logged, never returned by any endpoint, never included in an audit `before`/`after` payload.
  A redaction allowlist enforces this.
- Minimum 12 characters, checked against a small common-password list. No composition rules
  (punctuation and case rules push users toward predictable substitutions).
- `password_changed_at` is stored so refresh tokens issued before a password change can be revoked.

### Access tokens

- **Signed JWT, HS256.** A single symmetric secret, `JWT_ACCESS_SECRET`, supplied by environment
  configuration and a secret manager in deployed environments.
- **Lifetime 15 minutes**, configurable via `JWT_ACCESS_TTL`.
- Claims: `sub` (user id), `org` (active organization id), `role` (role key), `jti` (token id), `iat`,
  `exp`. **No email, no name, no permissions list.** Keeping the token small and free of personal data
  limits the blast radius of a leaked token and keeps it cheap to verify on every request.
- **Verification is strict**: the algorithm is pinned to HS256 on verify. Accepting the algorithm
  from the token header is the `alg: none` and HS/RS confusion class of vulnerability and is
  explicitly forbidden.
- `typ` and `kid` are checked. Clock skew tolerance is 5 seconds.

### Refresh tokens

- **Opaque random 256-bit token, stored only as a SHA-256 hash.** The raw token is shown to the client
  exactly once. A database dump yields nothing usable.
- **Lifetime 7 days**, configurable via `JWT_REFRESH_TTL`.
- **Rotating**: every refresh revokes the presented token and issues a successor in the same
  `family_id`.
- **Reuse detection**: presenting an already-revoked token whose family is still active revokes the
  entire family and audit-logs `auth.refresh_reuse_detected`. This is how theft is caught.
- Bound to IP and user agent at issue, recorded for forensics. Not enforced as a match, because mobile
  networks change IP legitimately.
- Changing organization or password starts a **new family**, revoking every prior token.

### Login hardening

- Rate limiting per IP **and** per email, via `express-rate-limit`. Per-email is what actually slows
  credential stuffing against a known address.
- Failed logins increment `failed_login_count`; a short lockout follows. **This is deliberately
  modest**, because aggressive account lockout is a denial-of-service vector against a known admin
  address. Rate limiting is the primary control; lockout is a secondary speed bump.
- Response messages are identical for unknown email and wrong password, so the endpoint is not an
  account-existence oracle.
- Successful logins record `last_login_at` and audit `auth.login`.

### Also

- Password reset and email verification tokens are single-use, hashed, short-lived, and invalidated on
  use. Email delivery arrives in Phase 8; until then the reset endpoint returns the token **only when
  `NODE_ENV !== 'production'`** so the flow is testable. This is a temporary, explicitly gated
  affordance and must be removed when Phase 8 lands.
- Secrets are never committed. `.gitignore` excludes `.env*`, `*.pem`, `*.key`, service-account JSON.
  GitHub secret scanning push protection and a gitleaks CI job are both enabled.

## Consequences

**Easier**

- Access checks are a local signature verification. No database round trip per request.
- Stateless horizontal scaling of the API with no shared session store.
- Token theft is bounded at 15 minutes, and refresh theft is *detectable*.

**Harder**

- Authorization data lives in the token for its lifetime. Revoking a role does not take effect on
  access tokens until they expire. Mitigation: `role:manage` and `user:assign_role` write a
  `tokenVersion` bump, and sensitive operations re-read permissions from the database rather than
  trusting the token claim.
- Refresh rotation means every client must handle a refresh failure by re-authenticating. The web
  client needs a token refresh interceptor.
- Two secrets to manage in production rather than one.

**We now owe**

- Secret rotation procedure. Because HS256 is symmetric, rotation requires coordinated redeployment;
  supporting overlapping verification keys is deferred.
- Removal of the non-production password-reset token exposure once Phase 8 lands. Tracked as a
  Definition-of-Done item for Phase 8.

## Alternatives considered

**Server-side sessions with opaque tokens** — Rejected. Correct and trivially revocable, but requires
shared session state, which means either a Redis dependency in the request path or sticky sessions
on Cloud Run. Reconsider if we ever need instant, guaranteed revocation.

**Long-lived access tokens (24h+)** — Rejected. A leaked token becomes a day-long credential. Fifteen
minutes plus rotation is the better trade.

**Refresh tokens without rotation** — Rejected. Rotation is what gives reuse detection, and without
it a stolen refresh token is indistinguishable from the legitimate one until it expires.

**Storing raw refresh tokens** — Rejected. A read-only database compromise, or a leaked backup,
becomes a complete authentication bypass for every account.

**`argon2`** — Rejected for now. It is the stronger primitive and would be the better choice if a
native build step is acceptable. bcrypt was chosen because the build plan mandates it, the
pure-JS implementation avoids native compilation across ten machines and in CI containers, and cost
factor 12 is well within OWASP guidance. Recorded here so the decision is revisited deliberately if
the constraint is ever lifted.

**An external identity provider (Auth0, Clerk, Firebase Auth)** — Rejected. It removes real work, but
the build plan mandates JWT + bcrypt, and it would put payroll identity data outside our audit trail,
which conflicts with the auditability requirement.