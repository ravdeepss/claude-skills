# <PROJECT NAME> — <NAME>_TEST_PLAN.md Template

> Fill in every placeholder below. Delete this top comment block when saving.
> File name on disk: `<NAME>_TEST_PLAN.md` at the project root.
> This template is the exact shape `plan-runner` and `test-runner` expect. Do not rename sections.

---

# <Project Title> — Test Plan

> **Purpose:** <one-sentence purpose — what this plan delivers, e.g. "Playwright regression coverage for the <app> web UI, executable by a capable-tier agent under the test→fix→retest loop.">
> Each task is self-contained and designed to be handed to a worker agent.
> Phase 0 must complete before any feature phase starts.

---

## Architecture Decisions

Non-negotiable choices every worker must respect. Fill every line.

- **Target app:** `<path-to-app-or-package>` — built on <stack summary>.
- **Source of truth for features:** `app-spec.json` at `<path>` — drives the coverage matrix below.
- **Test runner:** <Playwright / Cypress / Puppeteer / Vitest-browser / other>, version `<x.y>`. Browser(s): <chromium / chromium+webkit / all>.
- **Mock strategy:** <in-app-mock | route-interception | real-db-ephemeral | hybrid>. See the Mock Layer section for the file path / env flag / fixture location.
- **Auth strategy in tests:** <bypass via stored state | mock-auth-client returns canned session | `/test/login` backdoor | real test user>. Test credentials: <email / password or "N/A — bypass">.
- **Realtime handling in tests:** <EventEmitter + BroadcastChannel | Supabase realtime via docker | out of scope>.
- **Time mock:** <Playwright `clock.install()` / `vi.useFakeTimers()` / native | not used>.
- **Selector policy:** primary = <`getByRole` / `data-testid` / other>. Secondary fallback = <...>. Raw CSS classes are BANNED. `nth-child` is BANNED.
- **Banned APIs (enforced by lint):** `page.waitForTimeout`, `cy.wait(<ms>)`, blanket `setTimeout`-based waits in helpers.
- **Fixture convention:** named fixtures at `<path>/{seed,empty,heavy}.json`. Each test declares which one it uses.
- **Artifact policy:** trace on failure, screenshot on failure, video <always / on failure / off>. Output path: `<test-results/ | cypress/videos/ | other>`.
- **Tagging taxonomy:** `@phase-<N>` + `@<feature-id>` + `@<type>` where type ∈ {smoke, happy, error, edge, realtime}. Tags appear both in task headers below and in the spec files the worker writes.
- **Single-test command:** `<exact command with --grep placeholder>` (e.g. `pnpm test:e2e -- --grep "<fragment>" --reporter=list`).
- **Whole-suite command:** `<exact command>`.
- **Typecheck command:** `<exact command>`.
- **Lint command:** `<exact command>`.
- **Do-not-touch files:** <list, e.g. migrations, generated types, the plan file, `app-spec.json`, out-of-scope subsystem directories>.
- **Budget knobs for `test-runner`:** `MAX_ATTEMPTS=<3>`, `MAX_FIXES_PER_RUN=<10>`, `MAX_WALL_TIME_MIN=<30>`.

---

## Worker Instructions

Every worker agent (Claude sub-agent or external model) receives this section as part of its preamble. Imperatives:

1. Read this plan first — Architecture Decisions and your assigned task block.
2. Read each file listed in Context Files below before writing code.
3. Stay inside your task's Spec and Acceptance Criteria. Do not refactor adjacent code or "improve" unrelated areas.
4. For test-writing tasks, match the Scenario verbatim, use the listed Selectors, use the named Fixture, and tag the test exactly as specified.
5. For bug-fixing tasks (dispatched via `test-runner`), one hypothesis → one file → ≤30 LOC. No batching.
6. Run the Typecheck and Lint commands after any change. A broken typecheck is not progress.
7. Never change files in the do-not-touch list.
8. Never add `waitForTimeout` / `cy.wait(ms)` / blanket sleeps. Use the runner's auto-waiting, `waitForResponse`, or `waitUntil`.
9. Report format: files touched, commands run, and a criterion-by-criterion pass/fail table.

---

## Context Files

Paths (project-root-relative) every worker reads before starting.

- `app-spec.json` — feature catalogue + `testEnvironment` block.
- `<SCHEMA.md or migration path>` — database source of truth for fixture authoring.
- `<mock-layer file path>` — mock implementation (once Phase 0 Task 0.3 completes).
- `<page-object base class path>` — selector conventions (once Phase 0 Task 0.5 completes).
- `<fixtures folder>` — seed/empty/heavy JSON (once Phase 0 Task 0.4 completes).
- `<project-level README or CONTRIBUTING>` — coding conventions.

Keep the list tight — this becomes required reading for every worker.

---

## Coverage Matrix

Every feature from `app-spec.features[]` must have a row. Task IDs here must match the Task headers below and the Task Dependency Graph at the bottom.

| Feature id | Smoke | Happy path | Error path | Edge | Realtime | Total tasks |
|---|---|---|---|---|---|---|
| <feature-a> | <0.?>      | <1.1>      | <1.2>      | <1.3>        | —        | 3 |
| <feature-b> | —          | <2.1>      | <2.2>      | <2.3>        | —        | 3 |
| <feature-c> | —          | <3.1>      | —          | <3.2>        | <5.1>    | 3 |
| <feature-d> | —          | —          | —          | —            | —        | 0 — (out of scope: <reason>) |
| **Total**   |            |            |            |              |          | **<N>** |

Empty cells where the column applies = missing coverage. Fix before calling the plan done.

---

## Mock Layer Spec

Describe the mock layer in enough detail that Phase 0 Task 0.3 can be written by a capable-tier worker without further design decisions.

- **File:** `<path>` (e.g. `apps/web/src/lib/mockClient.ts`).
- **Env flag:** `<NAME>=true` swaps the real client for the mock at module-load time.
- **Fixtures loaded from:** `<path>`, read at init, deep-cloned into an in-memory store.
- **Globals exposed for test introspection:** `window.__mockDb` (the store), `window.__mockRealtime` (event emitter), any other handles the tests call.
- **Auth behavior:** `<describe canned session shape, credentials, any error-path triggers>`.
- **Realtime behavior:** `<EventEmitter wiring, BroadcastChannel for multi-tab, how events are fired from mutations>`.
- **Write semantics:** `<in-memory only / optionally persisted between tests / ephemeral>`.
- **Not-implemented handling:** when an app call hits a method the mock doesn't cover, throw `new Error("mockClient: not implemented: <method>")` — this surfaces as a bucket-D failure for `test-runner`.

---

## Phase 0 — Foundation  (must complete before all other phases)

Every Phase 0 task is `capable` tier unless marked otherwise.

### Task 0.1 — Verify / repair the target app builds and runs

**Goal:** Ensure the app boots past the loading shell and reaches an authenticated dashboard.

**Spec / Changes:** `<list every file/config worker may need to fix; e.g. "ensure apps/web/package.json is complete and parses">`. Do not scope-creep into feature fixes — only what prevents boot.

**Tier:** capable
**Suggested model:** <Sonnet, GLM-4>

**Acceptance criteria:**
1. `<install command>` completes with exit 0.
2. `<dev command>` starts and `<dev URL>` returns HTTP 200 with non-empty HTML body.
3. A browser navigating to `<dev URL>` renders past the initial loading spinner within 5 s (screenshot attached).

---

### Task 0.2 — Install test runner + config + lint

**Goal:** <e.g. "Install Playwright, generate config, add an ESLint rule that bans waitForTimeout.">

**Spec / Changes:** exact package names, exact config file paths, lint rule snippet.

**Tier:** capable
**Suggested model:** <Sonnet, Qwen-Coder-32B>

**Acceptance criteria:**
1. `<install command>` exits 0 and adds the runner to `package.json`.
2. `<runner> --version` prints the expected version.
3. Running `<lint>` on a decoy spec containing `waitForTimeout` fails with the configured rule id.

---

### Task 0.3 — Build the mock layer

**Goal:** Implement the mock layer described in the Mock Layer Spec section above.

**Spec / Changes:** file at `<path>`; exports match the shape of the real client; env-flag swap at `<swap file path>`; globals attached to `window` in dev/test mode only.

**Tier:** capable
**Suggested model:** <Sonnet, GLM-4>

**Acceptance criteria:**
1. When `<ENV>=true`, `<app-entry>` loads the mock instead of the real client (verified by a `window.__mockDb` truthy check in dev).
2. A representative happy-path call (e.g. list tasks) returns the seed fixture's rows.
3. An unimplemented method throws the sentinel error listed in the Mock Layer Spec.

---

### Task 0.4 — Fixtures (seed / empty / heavy)

**Goal:** Create named fixtures at `<fixtures path>`: `seed.json`, `empty.json`, `heavy.json`.

**Spec / Changes:** each fixture follows the table schemas documented in `<schema source>`. Content:
- `seed.json` — one user, one workspace, three projects, ~20 tasks (mixed statuses), a few tags, one template.
- `empty.json` — one user, one workspace, zero of everything else.
- `heavy.json` — one user, one workspace, ~10 projects, ~500 tasks (for list-perf and virtualization).

**Tier:** cheap (pure data authoring)
**Suggested model:** <Haiku, Qwen-Coder-7B>

**Acceptance criteria:**
1. Each file is valid JSON and conforms to the schema.
2. Loading `seed.json` into the mock layer and hitting `<route>` renders visible data.
3. `empty.json` renders an empty-state with no console errors.

---

### Task 0.5 — Page-object base classes

**Goal:** One page-object per major route, enforcing the selector policy.

**Spec / Changes:** folder `<tests/e2e/pages/>`, files `<list them — DashboardPage.ts, AuthPage.ts, etc.>`. Each page exposes methods for the actions tests will call. No raw CSS in page-objects.

**Tier:** capable
**Suggested model:** <Sonnet, GLM-4>

**Acceptance criteria:**
1. Each file exports a class with a constructor taking the runner's Page handle.
2. Every method uses selectors matching the policy (`getByRole` / `data-testid`).
3. A smoke test imports `DashboardPage` and asserts the dashboard heading is visible.

---

### Task 0.6 — Auth test helper

**Goal:** One function tests call to get an authenticated session.

**Spec / Changes:** file `<path>`, function signature `signIn(page, { email?, password? })`. Behavior matches the Architecture-Decisions auth strategy.

**Tier:** capable
**Suggested model:** <Sonnet>

**Acceptance criteria:**
1. Calling `signIn(page)` with no args completes in < 2 s using the default test credentials / bypass.
2. After `signIn`, the page is on the authenticated landing route.
3. A test that skips `signIn` and hits an authenticated route is redirected to `/login` (proves the bypass only runs when invoked).

---

### Task 0.7 — Time / realtime harness (if applicable)

**Goal:** Time-mock helper + realtime event-injection helper.

**Spec / Changes:** <list exact helpers, e.g. `installClock()`, `emitRealtime(type, payload)`>.

**Tier:** capable
**Suggested model:** <Sonnet>

**Acceptance criteria:**
1. A demo test uses `installClock()` and `page.clock.runFor(25 * 60_000)` to advance through a 25-minute interval.
2. A demo test calls `emitRealtime("task.inserted", row)` and sees the corresponding UI update.

---

## Phase 1 — <first feature cluster, e.g. Auth>  (requires Phase 0 ✅)

### Task 1.1 — auth: smoke

**Goal:** The auth routes render without console errors.

**Feature under test:** `auth`
**Test type:** smoke
**Spec file:** `tests/e2e/auth/smoke.spec.ts` (NEW)
**Page objects / fixtures used:** `AuthPage`, no fixture (pre-auth).

**Scenario (Gherkin-lite):**
  Given a cold browser
  When  I navigate to `/login`
  Then  the sign-in form is visible
  And   the browser console has zero errors

**Selectors required:**
- `getByRole('heading', { name: 'Sign in' })`
- `getByRole('textbox', { name: 'Email' })`
- `getByRole('button', { name: 'Sign in' })`
  (Add `data-testid` to the source component if the role/name locator doesn't resolve; that's a test-fix, not an app change.)

**Fixture / mock setup:** none (pre-auth route).

**Tags:** `@phase-1 @auth @smoke`

**Tier:** cheap
**Suggested model:** <Haiku, Qwen-Coder-7B>

**Acceptance criteria:**
1. The spec file exists at the path above with one `test()` block titled "auth sign-in page renders".
2. `<single-test command with --grep "auth sign-in page renders">` exits 0.
3. The test's tags are exactly `@phase-1 @auth @smoke`.
4. No banned APIs used.

---

### Task 1.2 — auth: happy sign-in

<same macro, fill from the coverage matrix>

---

### Task 1.3 — auth: error path (bad credentials)

<same macro>

---

### Task 1.4 — auth: session persistence edge

<same macro — reload mid-session, expect to land on authenticated route, not `/login`>

---

## Phase 2 — <second feature cluster>  (requires Phase 1 ✅ or parallel, see graph)

<repeat the feature-phase structure above — one task per cell in the coverage matrix>

---

## Phase N — Optional / Future

These are not fully specced. Flesh them out when the scheduled phases are done. Each should correspond to an entry in `app-spec.outOfScopeForRegressionV1` or a testing concern beyond v1.

- **Real-DB smoke lane:** <1–2 sentence description>
- **Accessibility audit:** <1–2 sentence description>
- **Visual regression:** <1–2 sentence description>
- **MCP server tests:** <1–2 sentence description>
- **Billing / Stripe webhooks:** <1–2 sentence description>
- **OAuth providers:** <1–2 sentence description>

---

## Task Dependency Graph

> `plan-runner` and `test-runner` read this section. Append ` ✅` to a task's
> line when it's complete. Keep line format `  N.M — <name>` with optional
> `(requires N.M)` annotation.

Phase 0 (gates everything):
```
  0.1 — verify build
  0.2 — install runner + config + lint   (requires 0.1)
  0.3 — mock layer                        (requires 0.1)
  0.4 — fixtures                          (requires 0.3)
  0.5 — page-object base classes          (requires 0.2)
  0.6 — auth helper                       (requires 0.3, 0.5)
  0.7 — time / realtime harness           (requires 0.5)     (if applicable)
```

Phase 1 (requires all of Phase 0):
```
  1.1 — auth: smoke
  1.2 — auth: happy sign-in               (requires 1.1)
  1.3 — auth: error path
  1.4 — auth: session persistence edge
```

Phase 2 (requires Phase 1 ✅):
```
  2.0 — <feature>: smoke
  2.1 — <feature>: happy
  2.2 — <feature>: error
  2.3 — <feature>: edge
```

Phase 5 (requires Phase 2 ✅):
```
  5.0 — realtime: harness smoke
  5.1 — realtime: create-sync happy
  5.2 — realtime: cross-tab edge          (requires 5.1)
  5.3 — realtime: reconnection edge       (requires 5.1)
```

Phase N (Optional / Future — not scheduled):
```
  N.1 — real-db smoke lane
  N.2 — accessibility audit
  N.3 — visual regression
  N.4 — MCP server tests
```

---

## Notes on tier choices (internal — delete or keep)

- `cheap` → fixture JSON authoring, trivial smoke tests, doc writing.
- `capable` → every non-trivial spec, all page-object work, all mock-layer extensions, all bug-fix dispatch via `test-runner`.
- `heavy` → never. If you're reaching for it, pre-resolve the design decision in Architecture Decisions above.
