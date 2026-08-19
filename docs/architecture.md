# Architecture

This document summarizes the production design of
[PhabPhysics](../README.md). The application source is private.

## Request path

```text
Browser                         Cloud Functions                 Firebase
────────────────────────       ───────────────────────         ──────────────────
practice client        ─────►  /api/problems          ─────►  Firestore
guided player                   /api/submit                     classes
teacher dashboard               /api/progress                   userClasses
Algebra Blocks                  /api/classes                    progress
handout index                   /api/auth                       problemInstances
                                /api/handouts          ─────►  Cloud Storage
                                                               locked answer keys
```

The Express API dispatches problem requests through
`functions/src/problems/index.ts`. That module owns course parsing, the course
registry, unit/day lookup, and progress-key construction.

## Course model

The registry currently has three entries:

| Course id | Public label | Units | Progress prefix |
|---|---|---|---|
| `ap1` | AP Physics 1 | 0–8 | `u` |
| `apcMech` | AP Physics C | 0–13 | `c` |
| `ap2` | AP Physics 2 | 9–15 | `p` |

The AP Physics C id predates the addition of Electricity & Magnetism. It remains
unchanged because stored class and progress records already use it.

AP Physics C imports AP Physics 1 generator functions by reference when the
underlying physics is identical. Calculus-specific Mechanics content and all E&M
content live in the Physics C registry. AP Physics 2 has its own registry.

Course input follows two compatibility rules:

- A missing client value resolves to `ap1`, which preserves clients created
  before courses were introduced.
- An unrecognized client value returns an error. Stored class values are
  normalized separately so one invalid document cannot break every read.

Signed-in students inherit the course from their class and do not select it
themselves. Guests can switch courses in the unit picker; progress is stored
under a separate local-storage key for each course. The teacher dashboard also
has a read-only course preview.

## Client boundary

`public/app.js` owns the original practice flow, answer checking, and scoring.
It is global-state code used by live students, so changes to it are kept small
and reviewed separately.

The main page loads two wrappers after the core:

```text
app.js  →  practice-ui.js  →  course-ui.js
```

- `practice-ui.js` handles presentation and interaction improvements.
- `course-ui.js` owns course selection, course metadata, per-course guest
  progress, and request augmentation.

The course wrapper may append a course value to an API call, but it may not
change the endpoint, unit, day, or difficulty. The initial wrapper integration
changed one line in `app.js`: the guest units request now uses the existing
`apiCall` path so the wrapper can observe it.

Algebra Blocks is a separate client with its own `tutor-ui.js` module. It
reports completion through the same progress API as the practice page.

## Progress and problem instances

Progress is stored in one document per user. Course prefixes prevent collisions:

```javascript
{
  "u4d1": {                         // AP Physics 1
    "easy":  { solved, totalAttempts, lastSolved },
    "medium": { ... },
    "hard":   { ... },
    "boss":   { ... },
    "guided": { solved, totalAttempts, guidedId }
  },
  "c4d1": { ... },                  // AP Physics C
  "p9d1": { ... },                  // AP Physics 2
  "lastActivity": <timestamp>
}
```

Each class document stores its course, enabled modes, and target points. The
practice tiers have fixed weights: easy 1, medium 2, hard 3, guided 4, and boss
5.

Generated problems are stored as active server-side instances with their
answers. The client receives an instance id and an answer-free problem object.
A correct submission deletes the instance; a scheduled function removes stale
instances.

The enrollment link contains a class reference and join timestamp, not the
student's email address.

## Guided problems and simulations

The guided player moves through introduction, optional exploration and
prediction, playback, and scored questions. It supports 2D canvas and Three.js
engines for motion, forces, rotation, oscillation, fluids, thermodynamics,
electricity, magnetism, optics, waves, and modern physics.

Completion returns to the practice page through `postMessage` and a
local-storage handoff, then writes through `/api/progress/guided`.

## Handout pipeline

Student handouts and teacher keys are authored as HTML and rendered to
Letter-size PDF with Puppeteer. Posted student sources run through a print-fit
check that reports overflow against the page budget.

Teacher keys build separately. They are served from Cloud Storage through an
authenticated route, with a bundled fallback for emulator use. The end-to-end
test asserts both sides of that authorization rule: a teacher can download a
key and a student cannot.

## Operations

- A service-status document can put the application into maintenance mode
  without a deploy.
- A counter caps self-service teacher signup.
- Deploys are explicit commands. CI validates changes but does not deploy them.
