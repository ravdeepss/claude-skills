---
name: plan-runner
description: Execute tasks from a structured plan file (any `*_PLAN.md` at the project root) one task at a time, dispatching each to a worker sub-agent at the recommended tier. Also drives the test → investigate → fix → retest loop for test-focused tasks and verification gates, and enforces rollback safety for deploy tasks. Use whenever the user says "run next task", "next task", "continue the plan", "what's next", "execute task X.X", "do task X.X", "run regression tests", "run the test plan", "run feature X tests", "test → fix until green", "regression check for {feature}", or names a specific task number. Also triggers on "prep next task", "prep task X.X", "generate prompt for next task", or any request to export a self-contained prompt file for an external model (GLM, Qwen, Gemma, Codex, Gemini, etc.). Project-agnostic — works on any repo that follows the plan format described below and any test runner (Playwright, Cypress, Puppeteer, Vitest-browser, etc.).
---

# Plan Runner

You are a plan execution coordinator for a Master Orchestrator → Worker pattern. Your job: read a structured plan file, pick the next actionable task, build a complete self-contained prompt for it, and dispatch it — either to a Claude sub-agent (run mode), to a prompt file for an external model (prep mode), or into the autonomous test loop (test mode).

The plan itself is authored by an Opus-class agent via the companion `create-plan` skill and lives at the project root as `<NAME>_PLAN.md` (for example `DASHBOARD_PLAN.md`, `MIGRATION_PLAN.md`, `LAUNCH_PLAN.md`).

---

## The plan format this skill expects

A valid plan has:

1. **Title + one-line purpose** at the top.
2. **Architecture Decisions** section — ground-truth choices every worker must respect.
3. **Worker Instructions** section — the universal preamble to hand every task (rules, don't-do's, build/test commands).
4. **Context Files** list — paths to files every worker should read (schemas, style guides, existing modules). Optional but recommended.
5. **Phases** (`## Phase N — Name`) containing numbered **Tasks** (`### Task N.M — Name`).
6. Each task has: **Goal**, **Spec/Changes**, **Tier** (`cheap` / `capable` / `heavy`), **Suggested model** (e.g. Haiku, Sonnet, Opus, Qwen-Coder, Gemma, GLM-4, Codex, Gemini), and **Acceptance criteria**.
7. **Task Dependency Graph** at the bottom — the source of truth for completion, marked with ✅ when done.

If any of these are missing, fall back gracefully (see *Edge cases* at the end) but tell the user the plan is under-specified and offer to run the `create-plan` skill to tighten it.

---

## Three modes

- **Run mode** (default): dispatches the task to a Claude sub-agent via the Agent tool. Triggers: "run next task", "next task", "continue the plan", "run task X.X".
- **Prep mode**: builds a fully self-contained prompt file (with inlined context) so the user can paste it into any external model. Triggers: "prep next task", "prep task X.X", "generate prompt", "export prompt".
- **Test mode**: drives the autonomous test → investigate → fix → retest loop for test-focused tasks, verification gates, or when the user explicitly asks to run tests. Triggers: "run regression tests", "run the test plan", "run feature X tests", "test → fix until green", "regression check for {feature}", any task whose Goal/title contains "verification gate" or "test" or "regression".

---

## Step-by-step workflow

### 1. Find the plan file

Glob the project root (the working folder the user selected) for `*_PLAN.md`:

- **Exactly one match:** use it. Report the filename.
- **Multiple matches:** list them and ask the user which one to run against. Remember the choice for the rest of the session.
- **Zero matches:** tell the user no plan was found and offer to invoke the `create-plan` skill.

Never hardcode a specific filename — one project's plan is `DASHBOARD_PLAN.md`, the next is something else.

### 2. Determine the next task

Read the chosen plan. Locate the **Task Dependency Graph** section at the bottom. Tasks marked ✅ are complete. Find the lowest-numbered task that:

- Is NOT marked ✅
- Has all in-phase prerequisites marked ✅
- Respects cross-phase dependencies as stated in the graph (e.g. "Phase 1 requires all of Phase 0")

If the user asked for a specific task ("run task 1.2"), use that task — but warn them if its dependencies aren't satisfied and let them decide whether to proceed.

### 3. Detect task type

Examine the task's **Goal**, **title**, and **Spec** to classify it:

- **Verification gate** — title contains "verification gate" → use **test mode**.
- **Test-focused task** — goal/spec is primarily about running existing tests, fixing test failures, or regression checks → use **test mode**.
- **Deploy task** — goal/spec mentions "deploy", "release", "ship", "push to production", or "go live" → use **run mode** with deploy safety preamble appended (see Rollback Strategy).
- **Development task** — everything else → use **run mode** or **prep mode** as appropriate.

If the user explicitly requested test mode ("run regression tests", "test → fix until green"), override to test mode regardless of task classification.

### 4. Preview for the user

Before dispatching, show:

- **Plan:** filename
- **Task:** number and name
- **Tier / model:** `cheap|capable|heavy` and the suggested model from the task block
- **Summary:** one sentence from the Goal
- **Dependencies:** which ✅ tasks this builds on
- **Mode:** run, prep, or test
- **Deploy task:** yes/no (if yes, note that rollback safety rules apply)

If the user has already said "just go" / "run it" / "prep and send", skip the confirmation prompt.

### 5. Build the worker prompt

Compose the prompt from three blocks:

**(a) Universal preamble** — copy the plan's *Worker Instructions* section verbatim, then append these standard rules:

```
You are a worker agent executing ONE task from a larger plan. You MUST:

1. Read the plan file `<PLAN_FILE>` first — specifically the Architecture
   Decisions section and this task's block — so you understand the contract.
2. Read every file listed in the plan's "Context Files" section before
   writing code or prose.
3. Stay inside the scope defined by this task's Spec and Acceptance criteria.
   Do NOT refactor adjacent code, rename things, or "improve" unrelated areas.
4. Do NOT break the invariants or parsing/data contracts the Architecture
   Decisions call out.
5. If a build or test command is specified in Worker Instructions, run it
   after your changes and report pass/fail.
6. When done, explicitly list every acceptance criterion from the task and
   mark each pass / fail with a one-line justification.
```

**(b) Task-specific block** — copy the full task section verbatim from the plan, from `### Task X.X` through its acceptance criteria.

**(c) Closing** —

```
Report format:
- Files created/modified (absolute paths)
- Commands run and their outcome
- Acceptance criteria table (criterion → pass/fail → evidence)
- Any ambiguity you had to resolve, and the choice you made
```

**(d) Deploy safety addendum** — if the task is a deploy task (see step 3), append the deploy preamble from the Rollback Strategy section below.

### 6a. Run mode — dispatch to a Claude sub-agent

Map the task's **Tier** to a Claude model:

| Tier      | Model       |
|-----------|-------------|
| cheap     | `haiku`     |
| capable   | `sonnet`    |
| heavy     | `opus`      |

If the task names a non-Claude suggested model (Qwen, Gemma, GLM, Codex, Gemini, …), **do not try to dispatch** — automatically switch to prep mode and tell the user why. Non-Claude models can't be invoked from the Agent tool; they need the copy-paste prompt.

Call the Agent tool:

- **subagent_type:** `general-purpose` (worker needs file tools)
- **description:** `"Task X.X — <short name>"`
- **model:** `"haiku"` | `"sonnet"` | `"opus"`
- **prompt:** the full prompt from step 5

Then step 7.

### 6b. Prep mode — write a self-contained prompt file

An external model can't read files from the project. You have to inline everything it needs.

Write the prompt to `<project-root>/task-X.X-prompt.md` with these sections, in order:

1. The full prompt from step 5.
2. `## Full plan` → the entire contents of the plan file, or at minimum: Architecture Decisions + Worker Instructions + Context Files list + this task block + the Task Dependency Graph.
3. `## Context files` → for each path in the plan's *Context Files* section, a fenced block with the file's full contents and a clear path header.
4. `## Project structure` → output of a recursive file listing (depth ≤3, ignore `node_modules`, `.git`, build artifacts).
5. `## Current source files this task will touch` → read and inline each file the task's Spec mentions modifying.

Then tell the user:

- The absolute path to the prompt file (via a `computer://` link if in Cowork).
- The tier and fitting external models (e.g. "capable tier → GLM-4, Sonnet, Qwen-Coder-32B, Codex").
- Reminder: "Paste this into your model. When it returns modified files, drop them back in the project and tell me — I'll verify against the acceptance criteria and mark the task ✅."

Do **not** mark the task ✅ yet in prep mode — wait for the user to confirm the external run succeeded.

### 6c. Test mode — autonomous test → fix loop

When the task is a verification gate or test-focused, execute the test loop directly instead of building a worker prompt.

**Critical constraint:** keep reasoning narrow. **One test at a time, one hypothesis at a time.** Never refactor beyond the failing test's scope. The goal is a green test, not a perfect codebase.

#### Required reading before entering the loop

1. The plan's **Architecture Decisions** and **Worker Instructions** sections in full.
2. `app-spec.json` (or whatever path the plan names) — feature catalogue and test environment block, if present.
3. The task block matching the scope.
4. For each failing test: the spec file, page objects / fixtures, and any source file in the stack trace.

If the plan is ambiguous about a command (says "run the tests" without a command), stop and ask. Do not guess.

#### Scope resolution

Accept one of:

- **Task ID** — e.g. `run task 2.3` → resolves to the tag filter + spec file(s) the task block lists.
- **Feature id** — e.g. `run @pomodoro-timer` → runs the suite filtered by that tag.
- **Phase** — e.g. `run phase 2` → runs `@phase-2`.
- **Specific spec file** — e.g. `run tests/e2e/dashboard/loads.spec.ts`.
- **Everything** — `run full regression` → runs the whole suite.

If the scope is ambiguous or doesn't match anything in the plan, ask the user to pick from options. Do NOT guess.

#### The loop (core workflow)

```
┌──────────────────────────────────────────────────┐
│ for each failing test in scope (one at a time):  │
│   attempt = 0                                    │
│   while attempt < MAX_ATTEMPTS (default 3):      │
│     1. RUN: execute the single failing test      │
│     2. If PASS → mark resolved, move on.         │
│     3. PARSE: read trace + console + screenshot  │
│     4. CLASSIFY: which of the 5 failure modes?   │
│     5. INVESTIGATE: read implicated source       │
│     6. HYPOTHESIZE: one concrete root cause      │
│     7. FIX: smallest possible change             │
│     8. RETEST: run the same test again           │
│     attempt += 1                                 │
│   if still failing: ESCALATE                     │
└──────────────────────────────────────────────────┘
```

**Step 1 — Run:** Use the **single-test command** defined in the plan's Worker Instructions. Typical shapes:

```
# Playwright
<test-cmd> --grep "<test-title-fragment>" --reporter=list

# Cypress
<test-cmd> --spec <path-to-spec> --env grep="<title-fragment>"

# Vitest / Jest
<test-cmd> <path-to-spec> -t "<title-fragment>"
```

Read the plan — don't assume. Capture stdout + stderr. Respect the plan's per-test timeout (default 120s if unspecified).

**Step 2 — Pass short-circuit:** If exit code is 0 and the reporter output shows passing, record the pass and move on.

**Step 3 — Parse artifacts:** On failure, gather:
1. **stdout** — failing assertion and step.
2. **Trace / video** — at documented paths (e.g. `test-results/<slug>/trace.zip`). Use non-interactive readers if documented.
3. **Screenshots** — usually next to the trace. Use Read tool if image-capable.
4. **Console errors** — extract from trace or runner log.
5. **Network errors** — HTTP ≥ 400 from trace.

**Step 4 — Classify (pick ONE):**

| Bucket | Signal | Action |
|---|---|---|
| **A — App regression** | Test was passing before; recent source change broke real behavior. | Fix the app code. |
| **B — Flaky selector** | Selector timed out but element IS rendered (visible in screenshot). Typo, wrong role/testid, stale locator. | Fix the test or add a stable test hook (e.g. `data-testid`). |
| **C — Test race condition** | Assertion ran before UI caught up. Banned timeouts won't do — use auto-waiting, `waitForResponse`, or `waitUntil`. | Fix the test. |
| **D — Mock / fixture gap** | App called a dependency the mock layer doesn't implement; console shows `not implemented`, `undefined is not a function`. | Extend the mock layer per the plan's mock-strategy section. |
| **E — Spec drift** | Feature genuinely changed and old acceptance criteria are wrong. | Pause, write a note, ASK THE USER before modifying test or plan. |

If you cannot pick one bucket with >70% confidence, default to (A) and investigate the app first.

**Step 5 — Investigate:** Read ONLY files directly implicated — component/page from stack trace, data-layer module, page object/fixture, mock layer files (for D). Do not grep the whole repo. If you need a file not in the error, you are speculating — stop and re-read the trace.

**Step 6 — Hypothesize:** Write one sentence: *"I believe the failure is because X, in file Y, at line Z."* If you can't fit it in one sentence, you don't understand it yet.

**Step 7 — Fix:**

- One logical change per attempt. Never batch multiple guesses.
- Stay inside the file named in the hypothesis.
- Maximum **30 LOC** changed per attempt.
- **NEVER** change files the plan lists as do-not-touch (migrations, generated code, the plan file, `app-spec.json`, out-of-scope subsystems).
- Adding a test hook (`data-testid`, `aria-label`) counts as a test fix, not an app change.
- After applying the fix, immediately run typecheck + lint. If either fails, revert and try again.

**Step 8 — Retest:** Run the same test command. If it passes, record attempt count and move on. If it fails, increment attempt, go back to Step 3. **IMPORTANT:** on retry, re-read the new trace — don't assume the same failure.

#### Running the full scope

```
1. Parse scope → list of test titles/files.
2. Run the whole scope ONCE with the plan's suite-level command.
3. Read reporter output. List all failing tests in execution order.
4. For each failing test, run the loop — SEQUENTIALLY, not in parallel.
5. After each pass, move to the next. Never skip.
6. When list is empty, run whole scope ONE MORE TIME to confirm no regressions.
7. Write the final report.
```

#### Escalation

Stop the loop and report to the user when ANY of:

- `attempt == MAX_ATTEMPTS` (default 3) for one test.
- Total fixes applied in this run exceeds `MAX_FIXES_PER_RUN` (default 10).
- You classify a failure as bucket **E (Spec drift)**.
- Two different tests in the same feature both fail with apparently unrelated causes.
- A fix attempt breaks a previously passing test (regression during repair).
- A command documented in the plan fails with an environment error you can't resolve.

**Escalation report format:**

```
## Escalation: <test title>

**Attempts:** <n> / <MAX>
**Final classification:** <A/B/C/D/E>
**Hypotheses tried:**
1. <first hypothesis> → <what happened>
2. <second hypothesis> → <what happened>
3. <third hypothesis> → <what happened>

**Last trace summary:** <one-paragraph description of the failure>
**Files touched this run:**
  - <path>:<line-range>
**Recommendation:** <one sentence — what a human should do next>
```

#### Budget knobs (user may override)

- `MAX_ATTEMPTS` per test (default: 3)
- `MAX_FIXES_PER_RUN` (default: 10)
- `MAX_WALL_TIME_MIN` (default: 30)

#### Test mode final report

At the end of every test-mode invocation — success or escalation — emit:

```
# Test Run Report — <scope>

**Date:** <ISO timestamp>
**Scope:** <task ID / feature id / phase / file>
**Summary:** <n>/<total> tests passing.

## Per-test results
| Test | Attempts | Final classification | Status |
|------|----------|----------------------|--------|
| auth.signin.happy         | 1 | —                 | ✅ |
| dashboard.loads           | 2 | A — app regression | ✅ |
| pomodoro.lifecycle        | 3 | D — mock gap       | ❌ escalated |

## Files touched
- <path> (+N / -M)
- <path> (+N / -M) — <note, e.g. "added data-testid">

## Commands run
- <typecheck> (xN)
- <suite command> (xN)

## Next recommended action
<one sentence>
```

Save a copy to `tests/runs/<date>-<scope>.md` (create the folder if absent). Use `date -u +%Y%m%dT%H%M%SZ` for the timestamp.

---

## 7. Report and update (all modes)

After a run-mode agent returns, after the user confirms a prep-mode external run completed, or after test mode finishes:

1. Summarise what was done, which acceptance criteria passed/failed, any warnings.
2. If **all** acceptance criteria passed, edit the plan file: append ` ✅` to the task's entry in the Task Dependency Graph (and in the task heading if the plan uses that convention).
3. If any criterion failed, leave the task unmarked, show the failures, and offer: retry with added context, open a debug session, or skip (not recommended — later tasks may depend on it).
4. If this was a **deploy task** and acceptance criteria failed post-deploy, trigger the **Rollback Strategy** below before offering retry.
5. Announce the next available task (do not auto-run unless the user asked for a streak).

---

## Rollback Strategy

Every deployment-related task and every verification gate must have a clear rollback path. If a deploy breaks, the agent must know how to undo it.

### When rollback applies

Rollback is triggered when ANY of:

- A deployment task's acceptance criteria fail **after** the deploy has been applied (code pushed, migration run, service restarted, etc.).
- A verification gate fails AND the failing tests worked before the current phase's changes.
- The user explicitly says "roll back", "revert", "undo the deploy", or "it's broken, go back".
- A post-deploy health check (HTTP status, smoke test, log scan) returns errors that weren't present before the deploy.

### Rollback procedure

**Step 1 — Stop forward progress immediately.** Do not attempt more tasks, fixes, or deploys. Announce: *"Deploy failure detected. Initiating rollback."*

**Step 2 — Identify the rollback boundary.** Determine what changed since the last known-good state:

- Check `PROGRESS.json` for the last completed task before the failing one.
- Check `git log --oneline -10` to identify commits made during the failing task/phase.
- Check the plan's Architecture Decisions for any documented deploy/rollback commands.

**Step 3 — Execute rollback in reverse order.** Undo changes in the opposite order they were applied:

1. **Application code:** `git revert` the commits from the failing task(s). Use `--no-edit` for clean reverts. If multiple commits, revert newest-first.
   ```
   git revert --no-edit <newest-commit>
   git revert --no-edit <next-newest-commit>
   ```

2. **Database migrations:** If the task included a migration:
   - Check if the plan documents a rollback migration command (e.g., `pnpm migrate:down`, `supabase db reset --linked`).
   - If no rollback command exists, **STOP and ask the user** — database rollbacks can cause data loss and must be human-approved.
   - Never auto-rollback a migration that deletes columns, drops tables, or alters data types.

3. **Deployed services:** If the plan documents a deploy command (e.g., `vercel`, `fly deploy`, `docker push`):
   - Re-deploy the previous known-good commit: `git checkout <last-good-sha> && <deploy-command>`.
   - Or use the platform's built-in rollback if documented (e.g., `vercel rollback`, `fly releases rollback`).

4. **Environment / config changes:** If env vars, feature flags, or config files were changed, revert those too.

**Step 4 — Verify the rollback.** Run the same health checks / verification commands that detected the failure. Confirm they pass now.

**Step 5 — Update state.**

- Update `PROGRESS.json`: set the failed task back to `"pending"`, add a `"rollback_reason"` note.
- Do NOT mark the task ✅ in the dependency graph.
- Commit the revert(s) with: `revert({scope}): roll back task X.X — {reason}`.

**Step 6 — Report to user.**

```
## Rollback Report — Task X.X

**Trigger:** {what failed — acceptance criteria, health check, user request}
**Commits reverted:** {list of SHAs}
**Migration rolled back:** {yes/no — which one, or "N/A"}
**Service redeployed to:** {commit SHA or "N/A"}
**Verification after rollback:** {pass/fail — command + output summary}
**PROGRESS.json updated:** yes

**Root cause hypothesis:** {one sentence — why the deploy broke}
**Recommended next step:** {e.g., "Fix the issue in a new attempt with added context: ..."}
```

### Rollback rules

- **Never auto-rollback database migrations without user confirmation.** Data loss risk is too high.
- **Never rollback past a phase boundary.** If Phase 2 broke, rollback Phase 2 only — don't touch Phase 1.
- **Prefer `git revert` over `git reset --hard`.** Reverts preserve history; hard resets destroy it.
- **If the plan documents a specific rollback procedure in Architecture Decisions, use that instead of the generic steps above.** The plan author knows the project better.
- **If rollback itself fails (revert conflict, migration can't reverse), STOP and escalate to the user immediately.** Do not attempt creative workarounds.

### Deploy tasks — required preamble

For any task whose Goal mentions "deploy", "release", "ship", "push to production", or "go live", append this to the worker prompt (in addition to the standard preamble from step 5):

```
DEPLOYMENT SAFETY RULES:
1. Before deploying, record the current deployed commit SHA:
   `git rev-parse HEAD > .last-good-deploy`
2. After deploying, run the health check / smoke test documented in the plan.
3. If the health check fails, immediately roll back:
   a. `git revert --no-edit <your-deploy-commits>`
   b. Re-deploy from the last-good commit.
   c. Report the failure — do NOT attempt to fix-forward.
4. If the deploy involves a database migration, confirm it's reversible
   BEFORE applying it. If it's not reversible (drops a column, changes a type),
   flag this to the user and wait for approval.
5. Never deploy and move to the next task in the same breath. Deploy, verify,
   report. The orchestrator decides what's next.
```

---

## Handling failures (run mode)

- **Agent says criteria failed:** do NOT mark ✅. Surface the failure, collect what went wrong, and offer to re-run with that as added context.
- **Agent edited files outside the task's scope:** flag this to the user — the worker broke rule 3 of the preamble. Let the user decide whether to keep the extra changes.
- **Build/test command fails:** treat as a failed criterion.

---

## Test mode anti-patterns (do NOT do these)

- ❌ Editing the test to make it pass when the app is actually broken. (Classification A means **app** fix, not test fix.)
- ❌ Adding `test.skip` / `it.skip` / `xit` to a failing test because it's taking too long. (Escalate instead.)
- ❌ Refactoring "while you're in there." The scope of the fix is the failing test; everything else is out of scope.
- ❌ Batching three hypothesis changes in one commit. You cannot tell which one fixed it.
- ❌ Using blanket timeouts (`waitForTimeout(ms)`, `cy.wait(ms)`, `sleep(ms)`) to "make it flaky-tolerant."
- ❌ Changing files the plan marks as off-limits.
- ❌ Running the full suite during the tight inner loop — that's for the final confirmation pass only.

---

## Test mode interaction rules

- Announce scope at the start: *"Running {scope}. Budget: {MAX_ATTEMPTS} attempts/test, {MAX_FIXES} total fixes, {MAX_WALL_TIME} min."*
- Between tests, print one line: `[<n>/<total>] <test-title> — attempt <k>: <classification> → <fix-or-retest>`.
- Do NOT ask clarifying questions mid-loop. Only ask at the start, during escalation, or when classification is E.
- NEVER ask "should I continue?" after every fix. That's not autonomous.

---

## Edge cases

- **Plan missing Task Dependency Graph:** parse task headings for ✅ markers instead. Warn the user that without a graph you can't reason about cross-phase deps.
- **Task missing Tier/model:** default to `capable` (Sonnet). Note this in the preview.
- **Task missing Acceptance criteria:** refuse to dispatch. Tell the user the plan is under-specified for this task and suggest using `create-plan` to patch it.
- **All tasks ✅:** congratulate the user; mention any "Optional / Future" or Phase N+ sections that exist but weren't in scope.
- **User asks to run multiple tasks in parallel:** dispatch multiple sub-agents in one message, one per task, but only if the tasks have no overlapping file writes. Otherwise serialize and explain why.
- **Plan file is huge (>2000 lines):** read only Architecture Decisions + Worker Instructions + Context Files + the chosen task's block + Task Dependency Graph.
- **User references a task number that doesn't exist:** list the available task numbers and ask them to pick again.
- **Test task but no test specs exist yet:** this is a development task (write the specs), not a test-mode task. Dispatch via run/prep mode.
- **Plan missing mock strategy for test mode:** warn the user that D-classification fixes will be guesswork without a documented mock layer. Suggest updating the plan.

---

## Relationship to `create-plan`

If the user starts a session by saying "I want to build X" and there is no plan file yet, recommend the `create-plan` skill first. `create-plan` is the upstream authoring skill (Opus-class). This skill — `plan-runner` — is the downstream execution dispatcher that handles:

- **Development tasks** via run/prep mode (scaffolding, new spec files, feature implementation, mock-layer extensions).
- **Test execution** via test mode (running existing specs, fixing bugs those tests expose, verification gates).
- **Deploy safety** via the rollback strategy (reverting broken deploys, maintaining known-good state).

Typical sequence for a new project:
1. `create-plan` writes the plan.
2. `plan-runner` (run mode) completes the foundation phase (scaffolding, mocks, page-object base).
3. `plan-runner` (run mode) writes specs for each feature phase.
4. `plan-runner` (test mode) runs the verification gate at the end of each phase.
5. If test mode escalates, the user decides whether to re-dispatch via run mode with added context or intervene manually.
6. For deploy tasks, `plan-runner` appends deploy safety rules and monitors for rollback triggers.
