---
name: create-test-plan
description: Author a detailed `<NAME>_TEST_PLAN.md` for regression / E2E / integration testing of an existing app, to be executed task-by-task by worker agents under the Master-Orchestrator → Worker pattern and then driven through the red→green loop by the `test-runner` skill. Use when the user says "create a test plan", "write a regression test plan", "plan the E2E tests for this app", "break down testing into tasks for agents", "scaffold a TEST_PLAN.md", "plan Playwright / Cypress coverage for my app", or any request to structure test work so it can be dispatched to cheaper models and then executed against a live codebase. Specialized sibling of `create-plan` — the two share the phase / tier / dependency-graph format, but `create-test-plan` adds a mandatory coverage matrix, per-feature task macro, mock-strategy intake, and explicit coupling to `test-runner` as the execution counterpart.
---

# Create Test Plan

You are a principal-engineer-level **test architect**. Your job: interview the user about an app that exists (or is being built) and needs regression coverage, then emit a `<NAME>_TEST_PLAN.md` at the project root that:

1. `plan-runner` can walk task-by-task to scaffold the suite and author specs.
2. `test-runner` can then execute to get and keep the suite green.

**Why this matters:** the plan you write will mostly be executed by cheaper worker agents (Sonnet, GLM-4, Qwen-Coder, Haiku for narrow doc tasks). Every ambiguity becomes a bug, a flaky test, or a missed feature. Your goal is: *a worker agent, given one task from this plan and the listed Context Files, can either (a) scaffold the right files and write a working spec on first try, or (b) run the test and apply the smallest correct fix without grepping the whole repo.*

## Relationship to `create-plan`

If you've already read `create-plan`: this skill is a specialization of it. Same phase/tier/task/acceptance-criteria DNA. The four things this skill adds:

1. **Mock-strategy intake** — pick one upfront, document it in Architecture Decisions, all downstream tasks respect it.
2. **Coverage matrix** — a mandatory `features × test-types → task-ids` table the plan is incomplete without.
3. **Per-feature task macro** — each feature from `app-spec.json` expands into a predictable cluster of tasks (smoke + happy + error + edge + realtime-if-applicable).
4. **Named test-specific sections** — selector strategy, fixtures, artifacts, tagging taxonomy, CI integration. Not optional, not adjustable per task.

If the user wants a generic build plan, use `create-plan`. If they want tests specifically, use this skill.

## Core principles

1. **`app-spec.json` is the source of truth.** The coverage matrix is built from `features[]`. If no spec exists, Phase 0 Task 0.1 is *"run the `app-spec` skill"* — do not proceed without one.
2. **Pre-resolve the hard stuff.** Mock strategy, selector policy, fixture seeding, test environment reset, auth approach — decide in the plan once, never re-litigated.
3. **Write for a capable-tier worker.** GLM-4 / Sonnet / Codex can write a Playwright spec if you hand them the acceptance criteria, the page-object path, the fixture, and the tag. They cannot design those abstractions under time pressure.
4. **Cover every feature at least once.** The coverage matrix makes missed features visible. If a row has no task id, the plan is not done.
5. **Keep the spec files testable by `test-runner`.** That means stable tags (`@phase-N`, `@<feature-id>`), single-spec-runnable, trace/screenshot on failure, no blanket timeouts.
6. **Phase 0 gates everything.** It builds the test harness. No feature phase runs until Phase 0 is green.

## Workflow

### 1. Intake — ask the user enough to plan well

Before writing anything, ask the four intake groups below using `AskUserQuestion`. Skip questions the user already answered in their initial message.

**Group A — Scope & spec**

- Which app? (repo path / monorepo package)
- Is there a current `app-spec.json`? If not, offer to run the `app-spec` skill first. Don't proceed without one.
- Which feature(s) are in scope — all of `features[]`, a subset, or a phase-by-phase rollout?
- What's **out of scope** for v1 of this test plan? (e.g. MCP server, billing, OAuth, real email delivery) — carry this forward from `app-spec.outOfScopeForRegressionV1` and confirm.
- Any broken features the user already knows about? (Flag them so Phase 0 / early tasks either stabilize or explicitly defer them.)

**Group B — Mock strategy (the decision that colors everything)**

Offer these four options — pick exactly one:

| Strategy | When to pick | Cost |
|---|---|---|
| **in-app-mock** | Backend is a BaaS (Supabase, Firebase, Amplify) and the app goes through a single client lib. Swap the client behind an env flag. Full behavior fidelity, works offline, fastest CI. | One-time mock-layer build. |
| **route-interception** | Backend has a stable HTTP API you own; test framework has good interception (Playwright `page.route`, Cypress `cy.intercept`). Tests own the canned responses. | Per-endpoint mocks in tests. |
| **real-db-ephemeral** | You have a dev DB you can wipe between runs (local Postgres container, SQLite, Supabase branch, `supabase start`). Highest fidelity. | CI infra + seeding step. |
| **hybrid** | Most features mocked, but a smoke lane runs against a real DB. Best of both at higher complexity. | Two configurations to maintain. |

Follow-ups:
- Which test runner? (Playwright / Cypress / Puppeteer / Vitest-browser / something custom.) If they don't know, default to **Playwright + chromium** for web apps.
- What auth behavior in tests? (Skip auth UI via storage state / hit a `/test/login` backdoor / mock the auth client / use a test user.)
- Is time mocking needed? (Pomodoro, due dates, snooze → yes.)
- Realtime/websocket/pub-sub features? (If yes, document how the mock exposes them — usually EventEmitter + BroadcastChannel for multi-tab.)

**Group C — Selectors, fixtures, CI**

- **Selector policy** — `data-testid` first? `getByRole` first? Both, with a priority rule? The plan will enforce this via lint later, so pick one now.
- **Fixture seeding** — per-test beforeEach? One worker-level seed? Named fixtures (seed/empty/heavy) as Playwright fixtures? The plan needs a file path.
- **Artifact policy** — trace on failure only? Video always? Screenshot on failure? What path do they live at? (`test-results/`, `cypress/videos/`, etc.)
- **Tagging taxonomy** — default is `@phase-N` + `@<feature-id>` matching the spec. Confirm or override.
- **CI integration** — will this run on every PR? On a nightly schedule? Both? Is there a `.github/workflows/` file you'll extend, or is this TBD?

**Group D — Budget & phasing**

- Approximate feature count (from `app-spec.json features.length`).
- Target phase breakdown — default is Phase 0 (foundation) + one phase per logical feature cluster (auth / core-crud / views / productivity / realtime / polish / optional). Let the user adjust.
- Worker tier mix — default is ~80% `capable`, ~15% `cheap` (doc/boilerplate), ~0% `heavy` (resolve architecture here, not in a task).
- Budget constraint for `test-runner` runs later — `MAX_ATTEMPTS` per test default 3, `MAX_FIXES_PER_RUN` default 10, `MAX_WALL_TIME_MIN` default 30. User can override.

Don't ask more than 4 questions at a time. Batch them, wait for answers.

### 2. Read `app-spec.json`

Once intake is done, read the full spec. You'll use:

- `features[]` → drives the coverage matrix and the bulk of feature-phase tasks.
- `testEnvironment` (if present) → cross-check against the mock strategy the user picked. Flag mismatches.
- `database.schemaSourceOfTruth` → context file for seed-fixture tasks.
- `frontend.routes[]` → maps to smoke-test tasks ("each route loads without console errors").
- `frontend.keyboardShortcuts[]` → dedicated shortcuts test task.
- `frontend.localStorageKeys[]` → persistence test cases.
- `outOfScopeForRegressionV1` → explicit carve-out in Worker Instructions.

### 3. Draft the plan

Use `TEST_PLAN_TEMPLATE.md` (in this skill's folder) as the skeleton. Don't invent new section names — `test-runner` parses the template's exact shape.

File name: `<NAME>_TEST_PLAN.md` where `<NAME>` reflects scope (e.g. `REGRESSION_TEST_PLAN.md`, `E2E_TEST_PLAN.md`, `CHECKOUT_E2E_PLAN.md`). Confirm naming with the user.

### 4. Build the coverage matrix (mandatory)

Place this table in the plan just below the Context Files section and above Phase 0. Every row is a feature from `app-spec.json`. Columns are test types:

| Feature id | Smoke | Happy path | Error path | Edge | Realtime | Total tasks |
|---|---|---|---|---|---|---|
| auth          | 1.1 | 1.2 | 1.3 | 1.4 | — | 4 |
| workspaces    | 2.0 | 2.1 | 2.2 | 2.3 | — | 4 |
| task-crud     | — | 2.4 | 2.5 | 2.6 | 5.2 | 4 |
| pomodoro-timer| — | 4.1 | 4.2 | 4.3 (reload) | — | 3 |
| realtime-sync | 5.0 | 5.1 | — | 5.3 (recon) | 5.2 (cross-tab) | 4 |

Rules:
- **Every feature** from `app-spec.features[]` must appear as a row. No omissions without being in `outOfScopeForRegressionV1`.
- **Smoke** = "the feature renders without crashing". Good to have for large features; skip for features that are just one API call (coverage comes from happy).
- **Happy path** = mandatory. Every feature.
- **Error path** = at least one. Validation, server error, permission denied, etc.
- **Edge** = at least one non-obvious case (empty state, max length, unicode, timezone edge, reload mid-operation).
- **Realtime** = only if the feature is in the realtime-sync dependency chain in the spec.
- Columns that don't apply get `—`. Empty cells mean missing coverage — fix before calling the plan done.
- "Total tasks" column is a sanity check.

When you write Phase N tasks below, the task IDs in the table and the task headers must match exactly.

### 5. Write Phase 0 — Foundation

Every test plan's Phase 0 has the same shape. Fill the specifics from intake:

- **0.1** — Verify / repair the target app builds and runs. (If the app is broken, e.g. stuck at a loading spinner, make this task explicit and gate everything on it.)
- **0.2** — Install test runner + framework, create config file, wire lint rule banning blanket timeouts. Include exact versions.
- **0.3** — Build the mock layer (whichever strategy the user picked). Include the file path, the env flag, the shape of the mock.
- **0.4** — Create fixtures / seed data matching the named fixtures (usually `seed.json`, `empty.json`, `heavy.json`). Include the schema source file as context.
- **0.5** — Page-object (or equivalent) base classes, one per major route. Establish the selector policy.
- **0.6** — Auth test helper (how tests sign in — bypass, mock, or real).
- **0.7** (if applicable) — Time mock helper and realtime harness.

Every Phase 0 task is `capable` tier. Never split Phase 0 across phases — it's gated as one.

### 6. Write each feature-phase task using the task macro

For each non-Phase-0 task, use the **mandatory task macro** below. Every task block contains, in this order:

```
### Task N.M — <feature-id>: <short imperative, e.g. "happy-path CRUD">

**Goal:** <one sentence — what this test proves>

**Feature under test:** <feature-id from app-spec.json>
**Test type:** smoke | happy | error | edge | realtime
**Spec file:** tests/e2e/<phase-or-feature-group>/<name>.spec.ts   (NEW — worker creates)
**Page objects / fixtures used:** <paths; these must exist after Phase 0 or be created in this task>

**Scenario (Gherkin-lite):**
  Given <preconditions>
  When  <user action>
  Then  <observable outcome>
  And   <secondary assertion>

**Selectors required:**
  - <role/name or data-testid>, e.g. `getByRole('button', { name: 'Add task' })`, `data-testid="task-row-:id"`
  If a selector is missing in the app source, adding it is IN SCOPE for this task (counts as a test-fix, not an app change).

**Fixture / mock setup:**
  - Which fixture file this test loads (e.g. `__mock-fixtures/seed.json`).
  - Any mock-layer methods the test needs that may not yet be implemented (bucket-D work).

**Tags:** @phase-<N> @<feature-id> @<test-type>   e.g. `@phase-2 @task-crud @happy`

**Tier:** cheap | capable | heavy   (default: capable; cheap only for trivial smoke tests; heavy never.)
**Suggested model:** e.g. "Sonnet, GLM-4, Codex"

**Acceptance criteria:**
  1. The spec file exists at the path above and has one `test()` block with the title implied by "Scenario".
  2. Running `<single-test command from Worker Instructions>` exits 0.
  3. The test's tags match exactly those listed.
  4. The spec uses selectors matching the policy in Architecture Decisions (no raw CSS classes, no `nth-child`).
  5. No `waitForTimeout(ms)` / `cy.wait(ms)` / `sleep()` calls.
  6. (If applicable) Mock-layer additions are in the file named in Architecture Decisions, not ad-hoc inside the spec.
```

Rules for macro use:

- **One test per task.** Do NOT batch "happy + error + edge" into one task. Each gets its own spec block and its own task ID. This is what makes `test-runner` effective — it drives one failing test at a time.
- **Maximum spec size per task:** ~40 LOC. If you need more, split.
- **Never reference "business logic" without giving selectors.** The worker will not invent them.
- **Always include the fixture.** If it's the default seed, say so explicitly (`"uses default seed fixture from Phase 0"`).
- **Acceptance criteria are observable.** No "feels right", no "covers the behavior".

### 7. Write Phase N — Optional / Future

For each item in `app-spec.outOfScopeForRegressionV1`, stub a Phase N entry with a 1-2 sentence description. These are not scheduled; they exist so nobody asks later "why didn't we test X".

Typical Phase N residents: real-DB smoke lane, MCP server, Stripe webhooks, OAuth providers, accessibility audit, visual regression.

### 8. Write the Task Dependency Graph

Same format as `create-plan`. `plan-runner` uses this to pick the next task.

```
Phase 0 (gates everything):
  0.1 — verify build        (blocks all)
  0.2 — install runner      (requires 0.1)
  0.3 — mock layer          (requires 0.1)
  0.4 — fixtures            (requires 0.3)
  0.5 — page objects        (requires 0.2)
  0.6 — auth helper         (requires 0.5, 0.3)
  0.7 — time / realtime     (requires 0.5)

Phase 1 (requires all of Phase 0):
  1.1 — auth: smoke
  1.2 — auth: happy sign-in
  1.3 — auth: error path
  1.4 — auth: session persistence edge

Phase 2 (requires Phase 1):
  2.0 — workspaces: smoke
  2.1 — workspaces: happy
  ...

Phase 5 (requires Phase 2):
  5.0 — realtime: harness smoke
  5.1 — realtime: create-sync happy
  5.2 — realtime: cross-tab edge
  5.3 — realtime: reconnection edge

Phase N (Optional / Future):
  N.1 — real-db smoke lane
  N.2 — accessibility audit
```

Keep the graph accurate as you edit tasks — `plan-runner` trusts it.

### 9. Self-review before handing to the user

Walk through this checklist before saving:

1. **Coverage matrix is complete** — every feature in `app-spec` has at least one task id in the Happy column. Every feature has either ≥1 error or ≥1 edge.
2. **Every task follows the macro** — Goal / Feature / Type / Spec file / Scenario / Selectors / Fixture / Tags / Tier / Model / Acceptance criteria.
3. **Tag format is consistent** — `@phase-N @<feature-id> @<type>`, same in the matrix, the task header, and the spec file.
4. **Acceptance criteria are observable** — no "works correctly", "looks good".
5. **Mock strategy is pre-decided** — every bucket-D acceptance criterion points at the Architecture-Decisions file path, not ad-hoc mocks.
6. **No `heavy` tier tasks** — anything that would need heavy reasoning has been resolved here in the plan.
7. **Phase 0 is truly self-contained** — no task in Phase 0 depends on any feature-phase deliverable.
8. **Dependency graph matches phases** — no orphans, no cycles.
9. **The plan tells `test-runner` what commands to use** — Worker Instructions has suite / single-test / typecheck / lint commands spelled out.
10. **Out-of-scope carve-outs are explicit** — and match `app-spec.outOfScopeForRegressionV1`.

### 10. Save and hand off

Write the plan to `<project-root>/<NAME>_TEST_PLAN.md`. Show the user:

- Path (via `computer://` link in Cowork).
- Summary: number of features covered, number of phases, number of tasks, tier mix.
- The coverage matrix pasted inline so they can eyeball it.
- The first recommended task, plus the prompt: *"Run `plan-runner` with 'run next task' to start scaffolding. Once Phase 0 is done, invoke `test-runner` with 'run phase 1' to validate."*

## Edge cases

- **User has no `app-spec.json`** — stop. Phase 0 Task 0.0 becomes *"run the `app-spec` skill to generate `app-spec.json`"*, and do NOT proceed with the rest of the plan until that file exists. Without it, the coverage matrix is vibes.
- **App is broken / won't run** — make this an explicit Phase 0 task with its own acceptance criterion ("browse to the dev URL and see the authenticated landing surface, not a loading spinner or a blank page"). All other phases wait on it. Do not let a broken build be implicit — a plan that schedules feature tests against an app that can't boot will fail for the wrong reason and teach `test-runner` nothing.
- **User wants only a subset of features tested** — that's fine, but keep the coverage-matrix row for every feature and mark out-of-scope rows with `— (out of scope)`. Don't silently omit them.
- **User picks `real-db-ephemeral` but has no DB reset strategy** — add a Phase 0 task to write a reset script (`scripts/reset-test-db.sh`). Do not use `in-app-mock` + real DB; that's the worst of both.
- **Test runner isn't named** — default to Playwright + chromium for any web app. Say so in Architecture Decisions and move on.
- **Existing tests already exist** — option A: plan a migration task to align them with the new conventions, then build on top. Option B: quarantine them under `tests/legacy/` and write fresh coverage. Let the user decide.
- **Plan would exceed ~40 tasks** — emit Phase 0 + the first feature phase fully specced, and stub later phases (title + coverage-matrix entries only). The user runs `plan-runner` through Phase 1, then you do a follow-up pass to flesh out Phase 2+.

## Relationship to other skills

- **`app-spec`** — produces `app-spec.json`, which this skill consumes. No spec, no plan.
- **`create-plan`** — the generic sibling. If the user wants a non-test build plan, send them there.
- **`plan-runner`** — executes tasks from the plan this skill produces (scaffolding, spec files, mock-layer work).
- **`test-runner`** — drives the red→green loop once specs exist. It reads Architecture Decisions (mock strategy, selector policy, do-not-touch list) directly from this plan. Make sure those sections are airtight — `test-runner` will not ask for clarification mid-loop.
- **`engineering:testing-strategy`** — useful for upstream "should we unit vs E2E vs contract test" discussions. Consult it before using this skill if the user hasn't decided the test-type mix.

Keep the plan small, unambiguous, and exhaustive on coverage. It's the one file the entire test stack below it trusts.
