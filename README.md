# PhabPhysics

**A production AP Physics learning platform — two full courses, ~135k lines, live in classrooms.**
[phabphysics.com](https://phabphysics.com)

> **Engineering case study.** The application source is private (live product, real
> student data). This repo is the architecture write-up: what the system does, the
> problems that were actually hard, and how correctness is enforced. Happy to walk
> through the code directly.

![PhabPhysics practice view](img/practice.png)

---

## What it is

Students sign in with Google, join a class, and work randomized physics problems
across four difficulty tiers plus guided interactive problems with 2D canvas and
Three.js simulations. Teachers get a dashboard for class progress, settings, and
admin-locked answer keys. Every lesson is paired with a print-ready handout.

Two complete courses run from one codebase:

| Course | Scope |
|---|---|
| **AP Physics 1** | Units 0–8 — Math Prep through Fluids |
| **AP Physics C** | Unit 0 + Mechanics 1–7 + Electricity & Magnetism 8–13 |

| | |
|---|---|
| Shipped source | **135,158 lines** (53.6k frontend JS · 81.9k TypeScript) |
| Problem generators | Every unit × day × difficulty, randomized per draw |
| Guided interactives | 26 registries, 13 simulation engines (2D canvas + Three.js) |
| Handouts | 749 HTML sources → 670 print-budgeted PDFs |
| Correctness checks | **42 deterministic checkers**, all blocking |
| CI | 3 jobs — static verify, browser sweep, emulator-backed e2e |

## Stack

**Frontend** Vanilla HTML/CSS/JS (no framework), MathJax/KaTeX, Three.js
**Backend** TypeScript · Express · Firebase Cloud Functions
**Data** Firestore · Cloud Storage (locked PDFs)
**Platform** Firebase Hosting · Firebase Auth (Google) + App Check
**Tooling** Terser · `tsc` · Puppeteer (PDF generation, browser tests) · GitHub Actions

```mermaid
flowchart LR
  subgraph Browser
    A["index.html + app.js<br/><i>practice core</i>"]
    B["practice-ui.js<br/>course-ui.js<br/><i>wrapper layer</i>"]
    C["guided player<br/>2D canvas + Three.js"]
    D["admin.html<br/><i>teacher dashboard</i>"]
  end
  subgraph "Cloud Functions (Express, TS)"
    E["/api/problems<br/>/api/submit<br/>/api/progress"]
    F["/api/classes<br/>/api/auth<br/>/api/handouts"]
    G["problems/index.ts<br/><i>course registry</i>"]
  end
  subgraph Storage
    H[("Firestore<br/>classes · progress<br/>problemInstances")]
    I[("Cloud Storage<br/>locked-handouts")]
  end
  A --> B --> E
  C --> E
  D --> F
  E --> G
  E --> H
  F --> H
  F --> I
```

Deeper systems detail: **[docs/architecture.md](docs/architecture.md)**

---

## The parts that were actually hard

### 1. You cannot hand-check a generated problem

Problems are randomly generated per student, per draw. There is no fixed answer
key to diff against — the correctness surface is the *generator*, not the output.

So the generators are tested as properties, not examples. The harness runs every
registered generator (unit × day × difficulty × index) **100 draws each** and
asserts on the whole output shape: finite numeric answer, physically sane
magnitude, no template artifacts (`undefined` / `NaN` / `[object Object]`) reaching
student-facing text, well-formed optional payloads (graph, free-body diagram,
energy bars), and a clean client conversion.

Physics invariants get their own checkers — `check-energy-invariants.js` and
`gasbox-invariants.js` assert conservation laws hold across randomized runs, so a
generator that quietly violates energy conservation fails CI rather than a quiz.

### 2. Refactoring is not available when the users are live

`app.js` owns scoring and answer-checking for students who are graded on the
output. Rewriting it mid-semester is not a risk worth taking, and "we'll be
careful" is not a control.

So UI changes ship as **wrapper modules that load after it** and read or wrap its
globals without touching scoring logic — `practice-ui.js` (presentation),
`course-ui.js` (course plumbing), `tutor-ui.js` (tutor layer). Load order is
explicit in `index.html`. The wrappers are constrained by contract: `course-ui.js`'s
API wrapper is *add-only* — it may append a `course` parameter and nothing else,
never rewrite an endpoint or a unit/day/difficulty value.

Result: an entire second course shipped with **zero diffs** to the highest-risk
file in the system.

### 3. A second course, on top of live student data

AP Physics C was added to a codebase already holding a year of AP Physics 1
progress records. Constraints: no data migration, no broken clients, no collisions.

- **Namespaced progress keys.** AP1 keeps its original `u{unit}d{day}` Firestore
  keys; Physics C writes `c{unit}d{day}`. Neither can overwrite the other.
- **Absent means legacy.** Any missing `course` field anywhere in the system —
  request body, stored document, localStorage — resolves to `ap1`, so every client
  that predates the course concept keeps working untouched. An *unrecognized*
  course string returns `null` and surfaces as a 400, so typos fail loudly while
  omissions fail safe.
- **Frozen identifiers.** The internal id `apcMech` predates the E&M expansion and
  is now wrong as a description, but it is baked into stored progress keys and
  class settings. It stays. Renaming it would be a migration with no user-visible
  payoff.
- **Shared generators by reference.** Physics C Mechanics imports the same audited
  AP1 generator functions rather than copying them — one source of truth, so a
  correction lands in both courses at once.

### 4. Fixing the class of bug, not the instance

In a JavaScript string literal, `'\('` silently evaluates to `'('`. The backslash
disappears, the LaTeX breaks, and nothing throws — it renders as plain text in a
student's problem.

Finding one instance is a bug fix. The actual fix is `check:guided-latex`, a
blocking check that fails `verify` on the entire defect class. Most of the 42
checkers exist for this reason: each one is a bug that is now structurally unable
to ship again.

### 5. Generated content, deterministic gates

Curriculum content is authored with LLM assistance at a scale no single teacher
could hand-write. That is only defensible if the verification is not also
probabilistic — a hallucinated physics answer here is a production incident with
real students.

So authoring is gated by machinery that does not negotiate: per-unit authoring
contracts, snapshot baselines that must be explicitly re-approved, the 42 static
checkers, and Puppeteer sweeps that render every guided problem in a real browser
because code-level checks repeatedly passed content that was broken on screen.

`npm run verify` chains ~25 of these and must pass before anything ships.

---

## Verification & CI

Three GitHub Actions jobs run on every push:

| Job | Blocking | What it proves |
|---|---|---|
| `verify` | ✅ | Site build smoke test, functions `tsc`, wiring checks, 100-draw generator harness |
| `browser` | — | Puppeteer: guided-problem render sweep, tutor-UI regression, simulation invariants |
| `e2e` | ✅ | Full classroom path against Firebase emulators |

The e2e job drives the real critical path on emulated Auth/Firestore/Storage
(demo project, never production): teacher signs up → creates a class → student
joins → the assigned day renders → a wrong submission then a correct one → the
teacher dashboard reflects it → the locked answer key downloads for the teacher
**and is denied to the student**.

More on the verification model: **[docs/verification.md](docs/verification.md)**

---

## Screens

| Guided problem — live 3D simulation with telemetry |
|---|
| ![Guided problem](img/guided.png) |

| Algebra Blocks — direct-manipulation symbolic solver |
|---|
| ![Algebra Blocks](img/algebra.png) |

**Algebra Blocks** is a DragonBox-style manipulator: students drag terms across the
equals sign, cancel factors, and substitute between equations, so algebraic
rearrangement is performed as a physical action instead of asserted in one line of
scratch work. It writes back to the same progress API as the practice engine.

---

## Author

Alex Reinhardt — physics teacher; MS Software Engineering, RIT.
Design, implementation, physics content, and operations are mine.
