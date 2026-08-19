# Verification

How correctness is enforced on [PhabPhysics](../README.md).

The premise: this is a live site where students are graded on the output, and the
content is generated — both randomized at runtime and LLM-assisted at authoring
time. Neither is hand-checkable at scale. So the verification has to be
deterministic even though the content is not.

## `npm run verify` — the gate

One command chains ~25 checks. It must pass before anything ships.

```
build smoke test  →  functions tsc  →  wiring checks  →  per-unit content checks
                                                       →  generator harness
                                                       →  physics invariants
```

42 checker scripts in total. They fall into four kinds:

### Wiring checks
Structural guarantees about how content is registered — that every guided problem
id referenced by a day actually exists, that unit/day tables agree with the
registry, that every course's UI unit list matches its backend registry. These
catch the "shipped a link to nothing" class of failure.

### The LaTeX check
In a JavaScript string literal, `'\('` silently evaluates to `'('`. The backslash
vanishes, the math delimiter breaks, nothing throws, and the student sees raw
markup. `check:guided-latex` fails the build on the whole class.

This is the template for most of the suite: a bug is not considered fixed until
the *category* is structurally unable to recur.

### The generator harness
The core correctness problem. Problems are randomly generated per draw, so there
is no answer key to diff — the surface under test is the generator.

Every registered generator (unit × day × difficulty × index) runs **100 draws** and
each output is asserted on:

- the numeric answer is finite and lands in a physically sane magnitude band
- no template artifacts — `undefined`, `NaN`, `Infinity`, `null`, `[object Object]`
  — appear anywhere in student-facing text
- optional payloads are well-formed: graph descriptors use known types, energy-bar
  modes are valid, free-body diagrams and representations parse
- the client conversion produces a clean object with no answer leakage

Draw count is configurable, so a suspect generator can be hammered with 500+ runs
locally without slowing CI.

### Physics invariants
Domain assertions that no amount of type checking would catch. Conservation of
energy is checked across randomized scenarios; the gas-box simulation is checked
against its own thermodynamic invariants. A generator that produces a numerically
finite, well-formatted, *physically impossible* problem fails here.

## Rendering is part of the definition of done

Code-level checks repeatedly passed content that was broken on screen — correct
data, unusable page. So a browser tier exists: Puppeteer sweeps that render every
guided problem, a tutor-UI regression suite, live simulation invariant runs, and
the handout print-budget check.

The working rule that came out of this: **nothing is done until it has been
rendered and looked at.**

## Snapshot baselines

Per-unit content is pinned to committed baselines. Regenerating content diffs
against the baseline and fails on unexplained drift; changing the baseline is an
explicit, reviewed act. This is what makes LLM-assisted authoring safe to iterate
on — a fleet of agents can rewrite a unit, but it cannot quietly change an answer
that was previously verified.

## CI

| Job | Blocking | Runtime | Scope |
|---|---|---|---|
| `verify` | ✅ | fast | Build, `tsc`, wiring, generators, invariants |
| `browser` | — | ~5 min | Render sweeps, tutor regression, sim invariants |
| `e2e` | ✅ | ~10 min | Full classroom path on Firebase emulators |

The browser job is intentionally non-blocking: it needs a Chrome download and
several minutes, and its signal is worth seeing without holding up the fast gate.

The e2e job is blocking and runs the real critical path against emulated
Auth/Firestore/Storage on a demo project — nothing touches production:

> teacher signs up → creates a class with a course → student joins by code →
> the assigned day renders → a wrong submission, then a correct one →
> the teacher dashboard reflects the result →
> the locked answer key downloads for the teacher **and is denied to the student**

Authorization is asserted, not assumed. The denial case is a test.

## What this buys

Content can be produced far faster than one person could hand-write it, and the
things that would make that reckless — wrong answers, broken math, dead links,
content that renders as garbage, authorization holes — are each held by a check
that runs on every push.
