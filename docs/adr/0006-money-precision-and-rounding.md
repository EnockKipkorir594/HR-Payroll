# ADR-0006: Decimal money with explicit, single-point rounding

- **Status:** Accepted
- **Date:** 2026-10-04
- **Phase:** 0 (decision), enforced from Phase 3
- **Deciders:** Lead engineer

## Context

The build plan requires "decimal-safe money handling with explicit precision and rounding".

Payroll amounts are legally and financially exact. An employee shorted 0.01 KES because of a
floating-point artefact is a payroll defect that must still be explained, and statutory filings that
do not sum to the ledger are rejected.

The classic failure is `0.1 + 0.2 === 0.30000000000000004`. Naive float accumulation over a few
thousand employees accumulates error that nobody can explain to an employee or an auditor. The
failure mode is silent: totals still look plausible.

## Decision

### Storage

- All monetary columns are PostgreSQL **`Decimal(19,4)`**, surfaced as `Prisma.Decimal`.
  19 digits with 4 decimal places leaves room for intermediate precision and growth.
- Currency is stored alongside every amount (`char(3)`, `KES` for the MVP).
- **Floating-point types (`Float`, `Double`, JavaScript `number`) are forbidden for money.** This is a
  review rejection, not a lint warning.

### Computation

Money is manipulated through a **value object** in `@payroll/validation` (landed with Phase 3,
contract reserved in Phase 1), never as a bare `Decimal` or `number`. The value object:

- Rejects more than two decimal places on **input** from a client. A client sending 100.005 KES is a
  bug or an attack; it is a validation error, not something to silently round.
- Carries an explicit rounding mode on every operation, defaulting to `ROUND_HALF_UP`.
- Keeps full precision internally. No intermediate rounding.
- Serialises to a **string**, never a JSON number, so a JavaScript client cannot lose precision on
  parse. `"100000.00"` survives; `100000.00` may not.

### Rounding happens exactly once, at named points

Rounding is a **payroll decision with a statutory meaning**, so it is not scattered through the
calculation. It occurs only at points the rule version defines:

1. **Per statutory bracket**, where the law specifies whole units.
2. **Per final payee line**, the amounts actually shown on a payslip and reported to KRA.

Every intermediate step — gross, each deduction, each contribution, net — is carried at full
`Decimal(19,4)` precision. Rounding mode is stored on the `StatutoryRuleVersion` record, so a past run
can always be re-explained with the rules that produced it.

### Reproducibility

A payroll run records the `statutory_rule_version_id` it was calculated under. Given the same inputs,
the same compensation snapshot and the same rule version, the engine produces byte-identical output.
Golden tests pin known statutory scenarios. The engine is a pure function: no database, no clock, no
randomness, no network inside a rule.

### Comparison

Amount equality is exact. `0.1 + 0.2` compared against `0.3` must be true for our purposes, because
both sides are `Decimal`. This removes the entire class of "are these equal?" bugs that require an
epsilon.

## Consequences

**Easier**

- Totals reconcile. The sum of the payslip lines equals the payroll register equals the statutory
  report, because all three are read from `Decimal` columns.
- Amounts are explainable to an employee and defensible to an auditor.
- Reproducibility is testable rather than aspirational.

**Harder**

- `Prisma.Decimal` is not a `number`. Arithmetic requires explicit construction, which is more typing
  and more opportunities for a careless cast.
- JSON responses carry strings, so the web client needs a decimal-aware formatter. Money display must
  not use `Number(value)`.
- Serialization tests are needed to prove no amount leaves the API as a float.

**We now owe**

- A rule that forbids `Number()`/`parseFloat()` on any money path, enforced in review and by
  `eslint-plugin-security` where it can be.
- Golden tests for every statutory rounding rule, each citing its official source and effective date.

## Alternatives considered

**Store money as integer minor units** — Rejected. It is correct and fast, and it is what payment
systems use. It loses expressiveness for intermediate percentages and rates (a rate of 7.5% cannot be
expressed in minor units), forcing awkward scaling constants throughout the calculation. `Decimal` is
the better fit once percentages and prorations are in play.

**Use JavaScript `number` with rounding at each step** — Rejected. This is the actual bug we are
avoiding. It fails silently and the error is not reproducible.

**`big.js` or `decimal.js` directly at every call site** — Rejected. Correct primitives, but
scattering rounding calls invites the failure. Centralizing in a value object makes "how is this
rounded?" answerable by reading one file.

**Rounding only at the end** — Rejected as the *only* rounding. Statutory brackets genuinely do round
at defined intermediate points, and modelling that honestly is better than approximating it.

**Half-even (banker's rounding) instead of half-up** — Rejected. Half-even is statistically superior
for aggregation and is the right default in many financial systems. But statutory payroll amounts are
not aggregations of samples; they are individual entitlements, and half-up matches the convention
Kenyan statutory guidance describes. Weights are stored on the rule version, so this is revisitable
per rule without a code change.