# Verification

PhabPhysics serves content that is randomized at runtime and frequently drafted
with LLM assistance. Verification is split into automated checks, browser tests,
independent physics review, and final human approval.

## Fast verification

`npm run verify` runs both builds followed by 24 named check groups. The
current registry contains 3,786 generator registrations, so the default
100-draw sweep inspects 378,600 generated problems.

### Registry and wiring checks

These checks compare the course registry with frontend unit lists, verify that
each referenced guided-problem id is registered, and confirm unit/day
consistency. They catch missing registrations and client/backend drift.

### Generator sweep

For each course, unit, day, difficulty, and generator index, the harness checks:

- a finite numeric answer within a broad magnitude band
- no `undefined`, `NaN`, `Infinity`, `null`, raw template fragments, or
  object stringification in student-facing text
- valid optional graphs, energy bars, free-body diagrams, representations, and
  solution steps
- successful conversion to an answer-free client object

Failures include the full registry location and run number. The number of draws
can be increased locally with `GENERATOR_RUNS`.

This is a sanity and contract harness, not a proof that every answer is correct.
Selected generators also have independent derivation tests, and physics-specific
checks assert invariants such as energy conservation.

### Content contracts

Unit-specific checks enforce authored constraints and compare selected content
with committed baselines. AP Physics 1 Tier-3 content uses explicit
`--require-t3` snapshot gates. Updating a baseline is a separate action so a
content change is visible in review.

### LaTeX escaping

JavaScript silently removes the backslash in a string such as `'\('`. The
result is valid JavaScript but broken math markup. The guided-LaTeX check scans
for this defect before it reaches the browser.

## Browser verification

Puppeteer covers failures that static checks cannot see:

- every guided problem is opened in the production player
- prediction and exploration state transitions are exercised
- Algebra Blocks receives regression and completion checks
- simulation invariants run in the browser
- posted handouts are measured against their print-page budget

These tests were added because valid data and valid JavaScript can still produce
an unusable screen or PDF.

## End-to-end path

The end-to-end suite runs against Firebase Auth, Firestore, Functions, Hosting,
and Storage emulators on a demo project. It performs this workflow:

```text
teacher signup
  → class creation
  → student signup and class join
  → assigned lesson render
  → wrong submission, then correct submission
  → teacher dashboard update
  → teacher answer-key download
  → student answer-key denial
```

No production service or data is used.

## CI jobs

| Job | Failure handling | Typical scope |
|---|---|---|
| `verify` | Fails the workflow | Build, TypeScript, registry checks, generator sweep, derivations, invariants |
| `browser` | `continue-on-error` | Render sweeps, tutor regression, simulation checks |
| `e2e` | Fails the workflow | Full classroom path on Firebase emulators |

The browser job is reported separately because it downloads Chrome and takes
longer than the fast gate. Its non-blocking status is explicit rather than
described as a required check.

## LLM and human review

The model that drafts physics content does not certify it. A separate model
re-derives the result from the stated givens and attempts to find a
counterexample. Review notes record confirmed findings and leave incomplete
findings pending.

Automated checks remain authoritative only for conditions they can evaluate.
They cannot decide whether a prompt is pedagogically useful, whether wording is
ambiguous, or whether an untested derivation is correct. I review changes to
scoring and curriculum before release.

A small public implementation of the generator-sweep and baseline approach is
available in
[`verification-gates`](https://github.com/ahreinhardt/verification-gates).
