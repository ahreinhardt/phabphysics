# Architecture

Systems detail for [PhabPhysics](../README.md). Application source is private;
this describes the design, not the implementation.

## Request path

```
Browser                          Cloud Functions               Storage
────────────────────────         ─────────────────────         ──────────────────
index.html + app.js              /api/**  (Express, TS)        Firestore
  + practice-ui.js      ──────►    api/problems       ──────►    classes
  + course-ui.js                   api/submit                    userClasses
guided player (canvas/WebGL)       api/progress                  progress
admin.html (teacher)               api/classes                   problemInstances
handouts.html                      api/auth, api/handouts      Cloud Storage
                                   problems/index.ts             locked handouts
```

Everything course-related funnels through a single module, `problems/index.ts`,
which owns the course registry. That chokepoint is deliberate: adding a course
means registering it in one place, not threading a flag through the codebase.

## Course model

A `courseRegistry` maps a course id to its full unit configuration. AP Physics 1
is the original registry (units 0–8). AP Physics C lives in its own directory
(unit 0 shared, Mechanics 1–7, E&M 8–13) and **imports AP1 generator functions by
reference** wherever the physics is identical — never copies — plus its own
calculus-specific generators. E&M is entirely new work.

Three rules keep the two courses from colliding:

| Concern | Rule |
|---|---|
| Course resolution | Absent or empty course input → `ap1`. Unrecognized string → `null` → 400. |
| Progress keys | `u{unit}d{day}` = AP1 · `c{unit}d{day}` = Physics C |
| Identifiers | Internal ids are frozen once user data references them |

**How a client lands on a course**

| Who | Mechanism |
|---|---|
| Signed-in student | Inherited from their class settings — students never choose |
| Guest | Toggle in the unit-selection modal, persisted to `localStorage`, per-course guest progress |
| Admin | Course picker above the unit cards — view-only preview, re-fetches progress, never writes |

That the admin preview is strictly read-only is enforced by design, not by
convention: the preview path has no write surface at all.

## The wrapper convention

`app.js` is the highest-risk file in the system — it owns scoring and answer
checking for students being graded. It is not edited. UI and feature work ships as
wrapper modules loaded *after* it that read and wrap its globals:

```
index.html load order:  app.js  →  practice-ui.js  →  course-ui.js
```

- `practice-ui.js` — presentation layer
- `course-ui.js` — course plumbing; its API wrapper is **add-only** by contract
  (may append a course parameter; may never rewrite an endpoint, unit, day, or
  difficulty)
- `tutor-ui.js` — the Algebra Blocks tutor layer

This is a deliberate trade: some indirection, in exchange for a blast radius that
never reaches scoring. An entire second course shipped without touching it.

## Data model

Progress is a single document per user, keyed by namespaced day:

```javascript
{
  "u4d1": {                                    // AP Physics 1, unit 4 day 1
    "easy":   { solved, totalAttempts, lastSolved },
    "medium": { ... }, "hard": { ... }, "boss": { ... },
    "guided": { solved, totalAttempts, guidedId }
  },
  "c1d3": { ... },                             // AP Physics C namespace
  "lastActivity": <timestamp>
}
```

Points are weighted by tier (easy 1 → boss 5, guided 4); a day completes when
points meet the class goal. Class documents carry their own settings — course,
required points, which modes are enabled — so a teacher can tune difficulty
without a deploy.

**Active problem instances** are stored server-side with their answers and deleted
on correct submission, with a scheduled function reaping stale documents. Answers
are never resident in the client for an unsolved problem.

**Data minimization.** The user↔class link stores no student email. Enrollment
records carry a class reference and a join timestamp, nothing more.

## Handout pipeline

749 HTML sources → Puppeteer → 670 Letter-size PDFs.

Every page is subject to a **print budget** — a check that fails if content
overflows its page, because a handout that reflows in the browser and then breaks
across a page boundary is worthless in a classroom at 7:50am. Teacher answer keys
build separately and are served from Cloud Storage behind an authorization check;
students are denied, and that denial is asserted in the e2e suite rather than
assumed.

## Operational design

- **Emergency brake.** A service-status document is read on the request path, so
  the site can be put into a maintenance state without a deploy.
- **Capacity gate.** Self-service teacher signup is capped by a counter document —
  growth stays deliberate rather than accidental.
- **Deploys are explicit.** Hosting and functions deploy by command, never
  automatically from a merge. CI is a gate, not a trigger.
