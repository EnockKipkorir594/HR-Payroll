# ADR-0007: `@react-pdf/renderer` for payslip PDFs

- **Status:** Accepted
- **Date:** 2026-10-04
- **Phase:** 0 (decision), implemented in Phase 7
- **Deciders:** Lead engineer

## Context

The build plan lists "Puppeteer or `@react-pdf/renderer` — choose one for MVP" and lists "select one
PDF engine for MVP" under decisions to lock.

Payslip PDFs are generated in the **BullMQ worker** (`apps/worker`), which deploys to **GCP Cloud Run**
as its own container. The choice therefore determines the worker image size, its cold-start latency,
and its memory profile.

A payslip is a **fixed-layout, tabular document**: employee identity, a period, an earnings table, a
deductions table and statutory totals. It does not need a browser's layout engine, reflow, or CSS
cascade. Reports (payroll register, PAYE, NSSF, SHIF, P9) share that shape.

## Decision

**We use `@react-pdf/renderer` (v4) for all PDF generation: payslips and reports alike.**

Chosen because:

1. **No Chromium binary.** The worker image does not carry a ~400MB browser plus its shared
   libraries. This is the single largest factor in image size, cold-start time and Cloud Run instance
   startup cost.
2. **No sandbox flags.** Chromium in a container needs `--no-sandbox` or a carefully configured seccomp
   profile. Disabling the sandbox to make a container start is a real security downgrade, and it is
   avoidable here.
3. **Better model for the document.** React primitives map directly onto a fixed-layout document:
   `<View>`, `<Text>`, `<Table>` style composition. No HTML/CSS for a document that never reflows.
4. **Predictable resource use.** Rendering runs on a known Node path with a bounded memory profile,
   which matters when a batch job renders 500 payslips in one worker instance.
5. **Shared component model.** The same component tree renders a payslip and a statutory report, so
   report headers and money formatting are written once.

Constraints we accept:

- HTML/CSS is not available. Styling is limited to React Native-style style props. For tabular
  financial documents this is sufficient.
- Complex page-flow features are weaker than a browser. We do not need them.

## Consequences

**Easier**

- Small worker image, fast cold starts, lower Cloud Run cost.
- No browser sandbox to get wrong.
- One rendering model for payslips and reports.
- Deterministic output, which makes PDF content assertions possible in tests.

**Harder**

- We lose the ability to reuse an existing HTML/CSS template if one is ever needed.
- Visual fidelity is limited to the library's primitives, so a client asking for a bespoke branded
  layout is a bigger task than it would be with HTML.
- `@react-pdf/renderer` is a heavier dependency than a template string, and pulls a font stack.

**We now owe**

- Fonts embedded and licensed for the deployment target; font licensing is a real constraint for
  generated PDFs and is not something to discover late.
- A PDF content test that asserts the right employee, period and amounts appear, per the build
  plan's testing requirements.

## Alternatives considered

**Puppeteer** — Rejected. It renders HTML/CSS, which is genuinely better if the document needs
reflow, branding or a web-derived template. It costs roughly 400MB of image size, materially slower
cold starts, and forces a sandbox decision in an environment where the safe default is inconvenient.
We do not need what it uniquely offers.

**Puppeteer with `chrome-headless-shell`** — Rejected. A genuine improvement over full Puppeteer, and
the right answer if we were already committed to HTML. Still a browser, still the sandbox question.

**A PDF library such as PDFKit or pdfmake** — Rejected. No component model, so payslip and report
templates diverge and layout logic gets rewritten per document.

**Server-side HTML rendering then convert** — Rejected. Two rendering systems, two sets of bugs, and
the same Chromium dependency it was meant to avoid.

**Defer the decision to Phase 7** — Rejected. The build plan asks for it to be locked during planning,
and the choice determines the worker image, so it constrains the container work we do before Phase 7.

## Revisit condition

If a stakeholder requires an HTML/CSS-authored payslip template, or branded layouts beyond the library's
primitives, this decision should be revisited with the image-size cost measured rather than assumed.
Puppeteer with `chrome-headless-shell` is the fallback, and the payslip component model is
renderer-agnostic enough that only the render function changes.