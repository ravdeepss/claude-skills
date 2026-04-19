# <PROJECT NAME> — <NAME>_PLAN.md Template

> Fill in every placeholder below. Delete this top comment block when saving.
> File name on disk: `<NAME>_PLAN.md` at the project root.
> This template is the exact shape `plan-runner` expects. Do not rename sections.

---

# <Project Title> — Plan

> **Purpose:** <one-sentence purpose — what this plan delivers>
> Each task is self-contained and designed to be handed to a worker agent.
> Tasks within a phase are sequential unless the Task Dependency Graph says otherwise.
> Phases can overlap EXCEPT Phase 0, which must complete before any other phase starts.

---

## Architecture Decisions

Bullet list of non-negotiable choices every worker must respect. Examples of things
that belong here:

- **Runtime / deployment model:** e.g. "runs on `file://` protocol, no server required"
- **Source of truth:** e.g. "`workouts.md` is the sole database; the app is read-only"
- **Data / parsing contract:** e.g. "the parser returns `{ goals, muscle_groups, workouts }` — do NOT change the shape"
- **Conventions:** module structure, naming, units (kg vs lb, UTC vs local), directory layout
- **Libraries / dependency policy:** e.g. "vanilla JS + CSS, no external deps"
- **Build step:** e.g. "`python3 build.py` inlines modules into a single `dashboard.html`"
- **Invariants workers must not break:** public API shapes, wire formats, migration files

Keep it short and absolute — if a decision is still open, don't list it here; resolve it first.

---

## Worker Instructions

Every worker agent (whether Claude sub-agent or external model) receives this section as
part of its preamble. Write these as imperatives. Include:

1. Read this plan file first — Architecture Decisions and your assigned task block.
2. Read each file listed in the "Context Files" section below before writing code.
3. Stay within the scope of your task's Spec and Acceptance Criteria. Do not refactor
   adjacent code or "improve" unrelated areas.
4. Units / formats / conventions that apply across every task (e.g. "all weights in kg",
   "dates in ISO 8601", "2-space indent").
5. Build / test commands to run after changes. Example:
   `python3 build.py && python3 -m http.server 8080`
6. File permission step (if the toolchain needs it): `chmod -R 755 .` as the last step.
7. Reporting format: list files touched, commands run, and a criterion-by-criterion
   pass/fail table.

---

## Context Files

Paths (project-root-relative) that every worker should read before starting. If a file
doesn't exist yet, add a Phase 0 task to create it and list the path here anyway with
`(created by Task 0.X)` noted next to it.

- `SCHEMA.md` — data format spec  *(created by Task 0.1)*
- `CONVENTIONS.md` — coding and naming conventions
- `src/js/parser.js` — existing parser that new work extends
- `<add more>`

Keep this list tight — anything listed becomes required reading for every worker,
which costs tokens. Prefer narrow, high-signal files.

---

## Phase 0 — Foundation  (must complete before all other phases)

### Task 0.1 — <short imperative name>

**Goal:** <one sentence — what this task achieves>

**Spec / Changes:**

<Explicit file paths, function signatures, data shapes, examples. Use fenced code
blocks for snippets the worker should produce or match. Call out every "do not touch"
boundary. Include UI copy verbatim.>

**Implementation notes:** <library picks, algorithm hints, pitfalls — optional>

**Tier:** cheap | capable | heavy
**Suggested model:** <e.g. "Haiku, Qwen-Coder-7B" or "Sonnet, GLM-4, Codex">

**Acceptance criteria:**

1. <Observable check 1 — something a human or verifier can confirm>
2. <Observable check 2>
3. <...>

---

### Task 0.2 — <name>

<same structure>

---

## Phase 1 — <Name>  (requires Phase 0 ✅)

### Task 1.1 — <name>

<same structure>

---

## Phase 2 — <Name>  (requires Phase 0 ✅; parallel to Phase 1)

### Task 2.1 — <name>

<same structure>

---

## Phase N — Optional / Future

These are not fully specced. Flesh them out when Phases 0–<last> are done.

- **<Idea 1>:** <1–2 sentence description>
- **<Idea 2>:** <1–2 sentence description>

---

## Task Dependency Graph

> `plan-runner` reads this section to pick the next task. Append ` ✅` to a task's
> line when it's complete. Keep line format `  N.M — <name>` with optional
> `(requires N.M)` annotation.

Phase 0 (foundation, gates everything):
```
  0.1 — <name>
  0.2 — <name>    (requires 0.1)
  0.3 — <name>    (requires 0.1)
```

Phase 1 (requires all of Phase 0):
```
  1.1 — <name>
  1.2 — <name>    (requires 1.1)
  1.3 — <name>    (requires 1.1)
```

Phase 2 (requires all of Phase 0; parallel to Phase 1):
```
  2.1 — <name>
  2.2 — <name>    (requires 2.1)
```

Phase N (Optional / Future — not scheduled):
```
  N.1 — <name>
  N.2 — <name>
```

---

## Notes on tier choices (internal — delete or keep)

- `cheap` → docs, small parser/CSS tweaks, data lookups, simple lookups/renames.
- `capable` → new modules, chart logic, cross-module refactors, UI state.
- `heavy` → architectural work, ambiguous specs, security-sensitive changes. If you
  find yourself tempted to use `heavy`, first ask whether the plan can pre-resolve
  the hard call and leave the worker a mechanical task.
