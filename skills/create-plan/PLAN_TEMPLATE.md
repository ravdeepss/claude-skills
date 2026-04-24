# {PROJECT NAME} — {NAME}_PLAN.md

> Fill in every placeholder below. Delete this top comment block when saving.
> File name on disk: `{NAME}_PLAN.md` at the project root.
> This template is the exact shape `plan-runner` expects. Do not rename sections.

---

# {Project Title} — Plan

> **Purpose:** {one-sentence purpose — what this plan delivers}
> **Execution model:** {single-agent-sequential | orchestrator-with-N-subagents | human-dispatched}
> **Target agents:** {list agent names and models, e.g., "Hermes (GLM-5.1), Gurleen (Kimi-K2.6)"}
> **Platform:** {Hermes | OpenClaw | Claude Code | Cursor | other}
> **Estimated sessions:** {number of sessions expected to complete this plan}
>
> Each task is self-contained and designed to be handed to a worker agent.
> TDD is enforced: every code task follows write-test → verify-fail → implement → verify-pass → pre-push-check.
> Tasks within a phase are sequential unless the Task Dependency Graph says otherwise.
> Phase 0 must complete before any other phase starts.
> Every phase ends with a verification gate — no advancing with known failures.

---

## Agentic Architecture Configuration

This section tells `plan-runner` how to dispatch tasks.

- **Execution model:** {single-agent-sequential | orchestrator-with-N-subagents | human-dispatched}
- **Max concurrent tasks:** {1 for single-agent; N for orchestrator with N subagents}
- **Agent capabilities:** {describe each agent's strengths/weaknesses, e.g., "GLM-5.1: good at code generation but needs very explicit instructions, tends to skip error handling if not reminded"}
- **Session limits:** {e.g., "GLM-5.1 on Hermes: ~60min sessions, ~32K context window"}
- **Provider fallback order:** {e.g., "GLM-5.1 → Kimi-K2.6 → Claude Sonnet"}
- **Task sizing target:** {e.g., "Each task should complete in < 20 min to stay within session limits"}

---

## Session Resume — PROGRESS.json

> This file lives at the project root. Every worker reads it before starting and
> writes it after every task. If a session dies, the next session resumes from here.
> Phase 0 includes a task to create this file. Schema:

```json
{
  "plan": "{NAME}_PLAN.md",
  "started_at": "",
  "execution_model": "{single-agent-sequential | orchestrator-with-N-subagents}",
  "phases": {
    "0": {
      "status": "pending",
      "tasks": { "0.1": "pending", "0.2": "pending" }
    }
  },
  "current_phase": null,
  "current_task": null,
  "last_agent": null,
  "last_updated": "",
  "notes": "",
  "learnings": []
}
```

Workers append to `"learnings"` when they discover gotchas, workarounds, or non-obvious behavior (Quality Ritual Step 6). This list survives session death and helps the next session avoid repeating mistakes.

---

## Architecture Decisions

Non-negotiable choices every worker must respect. Keep it short and absolute — if a decision is still open, resolve it first.

- **Runtime / deployment model:** {e.g., "Next.js on Vercel, Supabase backend"}
- **Source of truth:** {e.g., "`app-spec.json` for features, Supabase migrations for schema"}
- **Data / parsing contract:** {e.g., "TypeScript types generated from Supabase — do NOT hand-write DB types"}
- **Conventions:** {module structure, naming, units, directory layout}
- **Libraries / dependency policy:** {e.g., "use existing deps only, no new npm packages without approval"}
- **Test runner:** {e.g., "Vitest for unit tests, Playwright for E2E"}
- **Mock strategy:** {in-app-mock | route-interception | real-db-ephemeral | hybrid — if applicable}
- **Invariants workers must not break:** {public API shapes, wire formats, migration files, generated types}

---

## Worker Instructions

Every worker agent receives this section as part of its preamble. These are imperatives — follow them exactly.

### Before starting any task

1. Read this plan file first — Architecture Decisions, Agentic Architecture Configuration, and your assigned task block.
2. Read each file listed in the "Context Files" section below before writing code.
3. Read `PROGRESS.json` at the project root to confirm your task is next and no prior task is incomplete.
4. Stay within the scope of your task's Spec and Acceptance Criteria. Do not refactor adjacent code or "improve" unrelated areas.

### Quality Ritual — execute this for EVERY code task

This is the heartbeat of the plan. Every task that produces code follows this ritual, no exceptions.

**Step 1 — Red (write the test, watch it fail):**
  - Write the test specified in the task's TDD Steps.
  - Run it: `{single-test command}`.
  - Confirm it FAILS. If it passes, the feature may already exist — investigate before proceeding.

**Step 2 — Green (implement, watch the test pass):**
  - Implement the feature/fix as specified.
  - Run the test again: `{single-test command}`.
  - Confirm it PASSES (exit 0).

**Step 3 — Build gate (prove nothing is broken):**
  - Run the full verification pipeline:
    ```
    {typecheck command} && {lint command} && {test command} && {build command}
    ```
  - ALL FOUR must pass. If any fail, fix before proceeding. A broken build is never acceptable.

**Step 4 — Self-verify (be your own QA):**
  - Launch the app and manually test the feature you just built, as a real user would.
  - Document what you see: "I navigated to X, clicked Y, saw Z." If something looks wrong, fix it now.

**Step 5 — Commit (with conventional format):**
  - `feat({scope}): {description}` — new feature
  - `fix({scope}): {description}` — bug fix
  - `test({scope}): {description}` — tests only
  - `docs({scope}): {description}` — documentation
  - `refactor({scope}): {description}` — restructuring
  - `chore({scope}): {description}` — maintenance

**Step 6 — Save learnings (capture knowledge immediately):**
  - Did you hit a gotcha, workaround, or non-obvious behavior? Write it down NOW.
  - Save as a comment in the code, a note in PROGRESS.json `"notes"` field, or a memory entry.
  - Do NOT defer this to "later" — learnings not saved immediately are lost when the session dies.

**Step 7 — Update progress:**
  - Update `PROGRESS.json` with the completed task status and timestamp.
  - If this is the last task in a phase, set the phase status to `"completed"`.

### Prohibitions

Do NOT:
- Use `any` types in TypeScript.
- Leave TODO/FIXME comments — either implement it or note it as out of scope.
- Skip error handling — every async call needs a catch/error boundary.
- Use `waitForTimeout` / `cy.wait(ms)` / blanket sleeps in tests.
- Modify files in the do-not-touch list.
- Push code that doesn't build.
- Mark a task complete if ANY step of the Quality Ritual failed.

### Session management

If you're running low on context or approaching a timeout:
1. Finish the current Quality Ritual step if possible (don't leave a half-committed state).
2. Commit your work.
3. Update PROGRESS.json with your current state, what's left, and any notes for the next session.
4. Stop cleanly. Do NOT try to squeeze in one more task.

The next session reads PROGRESS.json and picks up exactly where you left off.

### Conventions

- Units / formats: {e.g., "2-space indent, dates in ISO 8601, all weights in kg"}.
- Reporting format: list files touched, commands run, test results, and a criterion-by-criterion pass/fail table.

---

## Context Files

Paths (project-root-relative) that every worker should read before starting. If a file doesn't exist yet, add a Phase 0 task to create it and note `(created by Task 0.X)` next to it.

- `CONVENTIONS.md` — coding and naming conventions *(created by Task 0.X)*
- `{path}` — {description}
- `{path}` — {description}

Keep this list tight — anything listed becomes required reading for every worker, which costs tokens.

---

## Phase 0 — Foundation  (must complete before all other phases)

> For new projects: includes full bootstrap (repo setup, CI/CD, test runner, state tracking, quality gates).
> For existing projects: verify build, verify test runner, create missing context files, set up PROGRESS.json.

### Task 0.1 — {short imperative name}

**Goal:** {one sentence — what this task achieves}

**Context files to read first:**
- {path} — {why}

**TDD Steps:**

1. **Write the test first:**
   - Test file: `{exact path}`
   - Test description: "{exact test title}"
   - What to assert: {specific assertions}

2. **Verify the test fails:**
   - Run: `{exact test command}`
   - Expected: test fails

3. **Implement the feature:**
   - File(s) to create/modify: {exact paths}
   - Spec / Changes: {detailed specification}
   - Implementation notes: {hints, pitfalls}
   - Do NOT: {explicit prohibitions}

4. **Verify the test passes:**
   - Run: `{exact test command}`
   - Expected: test passes (exit 0)

5. **Pre-push verification:**
   - Run: `{typecheck} && {lint} && {test} && {build}`
   - All must pass.
   - Commit: `{type}({scope}): {description}`

**Tier:** capable
**Suggested model:** {e.g., "Sonnet, GLM-5.1"}

**Acceptance criteria:**
1. {Observable check 1 — a command to run and expected output}
2. {Observable check 2}
3. Pre-push pipeline passes (typecheck + lint + test + build all exit 0)

---

### Task 0.2 — {name}

{same structure}

---

## Phase 1 — {Name}  (requires Phase 0 verification gate)

### Task 1.1 — {name}

{same TDD task structure — every code task follows the Quality Ritual above}

---

### Task 1.V — Phase 1 QA gate  *(implicit — every phase ends with this)*

> **Every phase gets a QA gate automatically.** The planner does not need to justify
> including it. It blocks the next phase. It is never skippable.

**Goal:** Prove all Phase 1 features work end-to-end before anyone touches Phase 2.

**Steps:**
1. Run the full test suite: `{suite command}` — every test must pass.
2. Run typecheck: `{typecheck command}` — must exit 0.
3. Run build: `{build command}` — must exit 0.
4. Manual smoke test: launch the app, walk through every feature delivered in this phase as a real user. Document exactly what you see (screenshots / copy-pasted output / "I clicked X and saw Y").
5. If ANYTHING fails:
   - Fix it NOW. Do not log it for later. Do not move on.
   - After each fix, re-run steps 1-4 from scratch.
   - Only mark this gate complete when everything is green AND the smoke test is documented.
6. Update PROGRESS.json: set phase status to `"completed"`, note any learnings.

**Tier:** capable

**Acceptance criteria:**
1. `{suite command}` exits 0 with all tests passing
2. `{typecheck command}` exits 0
3. `{build command}` exits 0
4. Smoke test documented with specific observations (not "looks fine")
5. PROGRESS.json updated with phase completion

---

## Phase 2 — {Name}  (requires Phase 1 QA gate)

### Task 2.1 — {name}

{same TDD task structure}

---

### Task 2.V — Phase 2 QA gate

{same QA gate structure as Phase 1.V — every phase gets one}

---

## Phase N — Optional / Future

These are not fully specced. Flesh them out when scheduled phases are done.

- **{Idea 1}:** {1–2 sentence description}
- **{Idea 2}:** {1–2 sentence description}

---

## Task Dependency Graph

> `plan-runner` reads this section to pick the next task. Append ` DONE` to a task's
> line when it's complete. Keep line format `  N.M — {name}` with optional
> `(requires N.M)` annotation.
>
> Execution model: {single-agent-sequential | orchestrator-with-N-subagents}
> Max concurrent tasks: {1 | N}

Phase 0 (foundation, gates everything):
```
  0.1 — {name}
  0.2 — {name}    (requires 0.1)
  0.3 — {name}    (requires 0.1)
  0.V — Phase 0 verification gate  (requires all Phase 0 tasks)
```

Phase 1 (requires Phase 0 verification gate):
```
  1.1 — {name}
  1.2 — {name}    (requires 1.1)
  1.V — Phase 1 verification gate  (requires all Phase 1 tasks)
```

Phase 2 (requires Phase 1 verification gate):
```
  2.1 — {name}
  2.V — Phase 2 verification gate  (requires all Phase 2 tasks)
```

Phase N (Optional / Future — not scheduled):
```
  N.1 — {name}
  N.2 — {name}
```

---

> **Note:** PROGRESS.json schema is defined in the "Session Resume" section near the top of this plan.
