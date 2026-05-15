---
name: plan-runner
description: Execute tasks from a structured plan file (any `*_PLAN.md` in the `plans/` folder) one task at a time, dispatching each to a worker sub-agent at the recommended tier. Also drives the test → investigate → fix → retest loop for test-focused tasks and verification gates, and enforces rollback safety for deploy tasks. Checks `plans/PLANS_REGISTRY.json` to skip completed plans. Use whenever the user says "run next task", "next task", "continue the plan", "what's next", "execute task X.X", "do task X.X", "run regression tests", "run the test plan", "run feature X tests", "test → fix until green", "regression check for {feature}", or names a specific task number. Also triggers on "prep next task", "prep task X.X", "generate prompt for next task", or any request to export a self-contained prompt file for an external model (GLM, Qwen, Gemma, Codex, Gemini, etc.). Project-agnostic — works on any repo that follows the plan format described below and any test runner (Playwright, Cypress, Puppeteer, Vitest-browser, etc.).
---

# Plan Runner

You are a plan execution coordinator for a Master Orchestrator → Worker pattern. Your job: read a structured plan file, pick the next actionable task, build a complete self-contained prompt for it, and dispatch it — either to a Claude sub-agent (run mode), to a prompt file for an external model (prep mode), or into the autonomous test loop (test mode).

The plan itself is authored by an Opus-class agent via the companion `create-plan` skill and lives in the `plans/` folder as `<NAME>_PLAN.md` (for example `plans/DASHBOARD_PLAN.md`, `plans/MIGRATION_PLAN.md`, `plans/LAUNCH_PLAN.md`).

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

**First**, check `plans/PLANS_REGISTRY.json` if it exists. This is the source of truth for which plans exist and their status. Filter out any plan with `"status": "completed"` — completed plans are never re-executed.

Then glob `plans/` for `*_PLAN.md`:

- **Exactly one active (non-completed) match:** use it. Report the filename.
- **Multiple active matches:** list them (with status from registry) and ask the user which one to run against. Remember the choice for the rest of the session.
- **Zero matches (or all completed):** tell the user no active plan was found and offer to invoke the `create-plan` skill.
- **No `plans/` folder exists:** check project root for legacy `*_PLAN.md` files (backwards compatibility). If found, suggest migrating them to `plans/`.

Never hardcode a specific filename — one project's plan is `DASHBOARD_PLAN.md`, the next is something else.

### 1b. Load model routing config

After finding the plan file, check the project root (and the workspace folder) for `hermes-model-config.json`.

**If found:**
- Read it. This overrides the default tier→model mapping in step 6a.
- Use its `plan_tier_mapping` to resolve tiers: `tier_cheap` → the configured cheap model, `tier_worker` → the configured worker model, etc.
- Use its `routing.quick_reference` for task-nature-specific routing (e.g., a frontend build task routes to the `react_nextjs_build` model, not just the generic tier model).
- Use its `routing.roles` for role-specific routing when the task's description matches a known role (e.g., QA tasks → `qa_engineer.primary`).
- Check the plan's **Agentic Architecture Configuration** section for any user-specified model overrides — these take precedence over the config file.

**If not found:**
- Use the default tier→model mapping from step 6a (cheap→haiku, capable→sonnet, heavy→opus).
- Check if the plan's task block has a "Suggested model" field and use that as a hint.

**Override priority (highest to lowest):**
1. User explicitly says "use X model for this task" in the current conversation
2. Plan's Agentic Architecture Configuration section lists model overrides
3. **Task's "Suggested model" field in the plan — THIS IS THE PLAN AUTHOR'S INTENT. Use it directly. Do NOT resolve through tier mapping.**
4. `hermes-model-config.json` routing (by task nature, then by tier) — only when the task has no explicit Suggested model
5. Default tier→model mapping (cheap→haiku, capable→sonnet, heavy→opus) — only as last resort

**⚠️ Silent routing bug:** If a task has both `**Tier:** cheap` and `**Suggested model:** GLM 4.7`, resolving by tier (`tier_cheap` → gemma-4-31b per HERMES.md) silently overrides the plan author's explicit model choice. The plan author may use tiers that don't match HERMES.md's tier names (e.g., `capable` has no HERMES.md equivalent). ALWAYS check for `Suggested model` first — it IS the resolution.

### 1c. Crash recovery — check dispatch log

**Before dispatching any task, check whether the previous session crashed.** Read `plans/{NAME}_DISPATCH_LOG.jsonl` if it exists:

1. Read the last 5 lines of the dispatch log.
2. If the last line is `"action":"dispatched"` with no matching `"action":"completed"` or `"action":"failed"` for the same task:
   → **CRASH DETECTED.** The orchestrator died after dispatching but before recording the result.
3. Run **GitHub-first recovery** (see Dispatch Log & Crash Recovery section):
   - `git fetch origin && git log origin/main --since="<dispatched_at>" --oneline`
   - Grep commit messages for the task scope (conventional commit prefix, feature keyword)
   - If commit found → subagent completed, mark task code-complete, write `"action":"recovered"` to log
   - If no commit AND `dispatched_at` > 15 min ago → subagent timed out, write `"action":"re-dispatched"` to log, dispatch fresh
   - If no commit AND `dispatched_at` < 15 min ago → subagent may still be running, wait (or check running processes)
4. If the last line is `"action":"session_ended"` → clean resume, no recovery needed.
5. If the log doesn't exist → first run of this plan, no recovery needed.

**Always write `"action":"session_started"`** to the dispatch log at the beginning of every session.

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
- **Tier / model:** `cheap|capable|heavy` and the resolved model (from user override → plan config → hermes-model-config.json → task block → default mapping, in priority order)
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

1. Read `.app-spec/app-spec.json` (or `.app-spec/APP_SPEC_SUMMARY.md` for quick orientation)
   FIRST — it contains the full codebase architecture, file paths, schema,
   conventions, and feature details. This eliminates the need to explore
   the repo structure.
2. Read the plan file `<PLAN_FILE>` — specifically the Architecture
   Decisions section and this task's block — so you understand the contract.
3. Read every file listed in the plan's "Context Files" section before
   writing code or prose.
4. Stay inside the scope defined by this task's Spec and Acceptance criteria.
   Do NOT refactor adjacent code, rename things, or "improve" unrelated areas.
5. Do NOT break the invariants or parsing/data contracts the Architecture
   Decisions call out.
6. If a build or test command is specified in Worker Instructions, run it
   after your changes and report pass/fail.
7. When done, explicitly list every acceptance criterion from the task and
   mark each pass / fail with a one-line justification.
8. Do NOT read, write, or modify `PROGRESS.json`, `DISPATCH_LOG.jsonl`,
   or the plan file's Task Dependency Graph. The orchestrator owns all
   state files — report your results in your response and the orchestrator
   will update state after verifying your work.
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

**Hermes-specific delegation:** When dispatching via Hermes `delegate_task`, read `references/hermes-subagent-dispatch.md` for the full config guide — model name verification, timeout management, and common subagent pitfalls (wrong model = silent fallback, config restart requirement, ignored screenshot lists).

Map the task's **Tier** to a Claude model:

**Default tier→model mapping** (used when no `hermes-model-config.json` is loaded and no model override applies):

| Tier      | Model       |
|-----------|-------------|
| cheap     | `haiku`     |
| capable   | `sonnet`    |
| heavy     | `opus`      |

**If `hermes-model-config.json` was loaded in step 1b**, resolve the model from the config only if the task has NO explicit `**Suggested model:**` field. The plan's explicit model recommendation ALWAYS wins — see Step 1b priority rule #3.

When the config IS the authority (no plan-level model specified):
- Use `plan_tier_mapping` for tier-based resolution (e.g., `tier_cheap` might map to `gemma-4-31b`, `tier_worker` to `glm-5.1`).
- Use `routing.quick_reference` if the task nature is identifiable (e.g., `test_generation` → the configured QA model).
- If the resolved model is a non-Claude model (anything not haiku/sonnet/opus): **on Claude Code, switch to prep mode** — non-Claude models can't be invoked from the Agent tool there. **On Hermes, dispatch directly via `delegate_task`** — Hermes supports all OpenAI-compatible models (GLM-5.1, DeepSeek, Kimi, Gemma) as subagents. Always read `references/hermes-subagent-dispatch.md` for Hermes-specific config (provider, model ID verification, timeout management). Include the resolved model name in the preview so the user knows which model was chosen.

**User overrides:** If the user explicitly names a model in the current conversation ("use Sonnet for this one", "dispatch to GLM-5.1"), respect that. Dispatch directly on Hermes (all OpenAI-compatible models supported via `delegate_task`). On Claude Code, non-Claude models require prep mode — export a prompt file the user can paste into the target model.

If the task names a non-Claude suggested model (Qwen, Gemma, GLM, Codex, Gemini, …) — whether from the config or the plan — **on Hermes, dispatch directly** (all are supported). On Claude Code, switch to prep mode and export a prompt file.

**Before dispatching**, write a dispatch record to `plans/{NAME}_DISPATCH_LOG.jsonl`:

```jsonl
{"ts":"<ISO timestamp>","task":"X.X","model":"<resolved model>","action":"dispatched","goal":"<one-line goal from task>"}
```

Then call delegate_task:

- **goal:** the full prompt from step 5
- **context:** include the task's Goal, file paths, and architecture decisions
- **model:** the resolved model from tier/config/override
- **toolsets:** `['terminal', 'file']` (standard worker toolset)

**After the subagent returns**, write the outcome to the dispatch log:

```jsonl
{"ts":"<ISO timestamp>","task":"X.X","model":"<model>","action":"completed","commit":"<sha>","ac_total":4,"ac_passed":4}
```

or on failure:

```jsonl
{"ts":"<ISO timestamp>","task":"X.X","model":"<model>","action":"failed","error":"<one-line error summary>"}
```

The dispatch log is **append-only** — never modify existing lines. Atomic appends prevent corruption on crash.

**Push PROGRESS.json immediately** after every state change — never batch. A stale PROGRESS.json is a crash waiting to happen.

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
- The tier and recommended model. If `hermes-model-config.json` was loaded, name the specific model from the config (e.g., "capable tier → glm-5.1 per hermes-model-config.json"). Otherwise, suggest fitting external models (e.g., "capable tier → GLM-4, Sonnet, Qwen-Coder-32B, Codex"). If the user specified a model override, use that instead.
- Reminder: "Paste this into your model. When it returns modified files, drop them back in the project and tell me — I'll verify against the acceptance criteria and mark the task ✅."

Do **not** mark the task ✅ yet in prep mode — wait for the user to confirm the external run succeeded.

### 6c. Test mode — autonomous test → fix loop

When the task is a verification gate or test-focused, execute the test loop directly instead of building a worker prompt.

**Critical constraint:** keep reasoning narrow. **One test at a time, one hypothesis at a time.** Never refactor beyond the failing test's scope. The goal is a green test, not a perfect codebase.

#### Required reading before entering the loop

1. The plan's **Architecture Decisions** and **Worker Instructions** sections in full.
2. `.app-spec/app-spec.json` (or whatever path the plan names) — feature catalogue and test environment block, if present.
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
- **NEVER** change files the plan lists as do-not-touch (migrations, generated code, the plan file, `.app-spec/app-spec.json`, out-of-scope subsystems).
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

## 6d. MANDATORY: Push, Test & Deploy Gate (after EVERY task, before marking complete)

**CRITICAL:** A task is NOT complete just because code was written and committed locally. The plan runner MUST execute ALL THREE stages before marking a task "code-complete":

```
┌─────────────────────────────────────────────────────────┐
│ Stage 1: PUSH — git push all committed changes           │
│ Stage 2: TEST — run the full test suite, confirm no     │
│           regressions (pre-existing failures unchanged)  │
│ Stage 3: DEPLOY — if the task touches deployed surface   │
│           (Vercel web, MCP worker, edge functions),      │
│           deploy and verify the deployment is READY      │
└─────────────────────────────────────────────────────────┘
```

**Stage 1 — PUSH (mandatory):**
```bash
cd <repo> && git status  # Check for uncommitted changes
git add -A && git commit -m "feat(scope): task X.X — description"
git pull --rebase origin main && git push origin main
```
Never move past Stage 1 with an unpushed local branch.

**Stage 2 — TEST (mandatory):**
Run the full test suite. Accept ONLY these failure patterns:
- Pre-existing failures (documented in the plan's "Known Pre-existing Failures" section, or verified as unchanged from before the task)
- Flaky tests (random failures, not deterministic)

Any NEW deterministic failure is a BLOCKER. Do not proceed to Stage 3. Investigate and fix, then re-run the suite. Record: `N/M passing (K pre-existing failures)`.

**Stage 3 — DEPLOY (conditional, but REQUIRED when applicable):**
If the task touches ANY of:
- `apps/web/` → Vercel auto-deploys on push. Verify deployment went READY with `mcp_vercel_getDeployments`.
- `apps/mcp-server/` → Deploy with `npx wrangler deploy --name ike-mcp`. Verify with `curl https://ike-mcp.ravdeepss.workers.dev/health`.
- `supabase/functions/` → Deploy with `npx supabase functions deploy <name> --project-ref <ref>`. Verify logs show no errors.
- Supabase migrations → Already applied via `mcp_supabase_apply_migration`, but verify with a schema query.

After deploy, run a quick smoke test: load the app, check the health endpoint, or curl the function.

**PITFALL — "Code-complete" without push+deploy:** The cron job completed 12 tasks with commits sitting locally unpushed, the Vercel deployment was stale from a prior commit, and the edge function was never called because the deploy was outdated. PROGRESS.json said "complete" but the feature wasn't live. This is the #1 most damaging pattern in autonomous plan execution.

**Reporting format after Stage 3:**
```
Task X.X: <name>
  Commit: <sha> (pushed to main)
  Tests: N/M passing (K pre-existing unchanged)
  Deploy: Vercel READY / MCP worker deployed / Edge function deployed / N/A
  Smoke test: <result>
```

Only after ALL three stages pass, proceed to mark the task code-complete in PROGRESS.json.

---

## 7. Report and update (all modes)

After a run-mode agent returns, after the user confirms a prep-mode external run completed, or after test mode finishes:

1. Summarise what was done, which acceptance criteria passed/failed, any warnings.
2. If **all** acceptance criteria passed:
   - **Hermes-specific:** Subagents work on local clones — files are NOT pushed to GitHub automatically. After verifying the subagent's output, push all changed files to the feature branch via `mcp_github_push_files`. Read `references/hermes-subagent-dispatch.md` for the full post-subagent checklist.
   - **Per-task:** Mark the task as **code-complete** in PROGRESS.json with `"status": "code-complete"` and the commit SHA. 
   - **Record the model used** in `task_models`: `"task_models": { "X.Y": "model-name" }` (e.g., `"0.1": "glm-4.7"`, `"5.1": "deepseek-v4-pro"`). This enables post-hoc auditing of whether the plan's `**Suggested model:**` was actually respected. If the PROGRESS.json doesn't have a `task_models` key yet, create it.
   - Do NOT mark it `"complete"` and do NOT append ✅ in the Task Dependency Graph yet. Code-complete means the feature branch has the code and tests pass — but it hasn't been reviewed or merged.
   - **Do NOT create a PR yet.** PRs are created in batch at the end of the phase (see Phase PR Review Gate below). Creating per-task PRs floods the repository with tiny diffs and burns review cycles. The only exception is when the plan explicitly states per-task PRs, or when the task spans multiple repos that need cross-coordination.
   - Commit and push PROGRESS.json to the feature branch after each task update so progress is durable.
3. If any criterion failed, leave the task unmarked, show the failures, and offer: retry with added context, open a debug session, or skip (not recommended — later tasks may depend on it).
4. If this was a **deploy task** and acceptance criteria failed post-deploy, trigger the **Rollback Strategy** below before offering retry.
5. Announce the next available task (do not auto-run unless the user asked for a streak).
6. **Cron nudge setup:** When kicking off a long-running plan, offer to set up a recurring cron job that auto-nudges the plan forward. Use `cronjob` with action=create, the plan-runner skill, and a schedule like `every 2h`. Include the repo name, plan file name, and model routing config path in the prompt. Circuit breaker: if the same task fails 2x, the cron should stop and wait for human review.

---

## Phase PR Review Gate (MANDATORY at end of every phase)

**When:** After ALL tasks in a phase are code-complete (pushed to feature branches with passing tests), but BEFORE the phase verification gate (Task N.V) is run and BEFORE the phase is marked complete.

**DO NOT SKIP THIS GATE.** A phase where code sits unmerged on feature branches while PROGRESS.json says "complete" is a lie. Future sessions will waste time discovering unmerged work, chasing phantom regressions, or building on code that doesn't exist on main.

### Step G1 — Verify all tasks are code-complete

1. Read PROGRESS.json. Confirm every task in the current phase has `"status": "code-complete"` (or `"complete"` if already merged).
2. If any task is still `"pending"` or `"in_progress"`, do NOT proceed. Complete those tasks first.
3. List all feature branches used by this phase (`git branch -a | grep feat/`).

### Step G2 — Create batch PRs (one per repo)

For each repo that this phase touched:

1. Identify all feature branches that need merging into `main` (or the plan's target branch).
2. If multiple feature branches exist for one repo, merge them into a single phase branch first:
   ```bash
   git checkout main && git pull origin main
   git checkout -b feat/phase-{N}
   git merge feat/{task-branch-1} feat/{task-branch-2} ...
   git push origin feat/phase-{N}
   ```
3. Create ONE pull request per repo using `mcp_github_create_pull_request`:
   - `head`: `feat/phase-{N}` (or the individual feature branch if only one)
   - `base`: `main` (or the plan's target branch)
   - `title`: `"Phase {N}: {phase name}"` — e.g., `"Phase 2: Multi-tenancy"`
   - `body`: Summarize what this phase delivers, list all tasks, and link to PROGRESS.json

### Step G3 — Wait for CI

For each PR:
1. Check CI status using `mcp_github_get_pull_request_status`.
2. If CI fails: read the failure logs, identify the cause, and either:
   - Fix trivially (missing semicolon, lint error) — commit the fix, push, re-check.
   - Assign a fixer subagent if the fix is non-trivial. Use the same model that wrote the failing code.
   - Do NOT mark the phase complete with failing CI.
3. If CI passes: proceed to review.

### Step G4 — Assign reviewer subagent

For each PR, dispatch a review subagent with the `github-code-review` skill:

```yaml
Model routing for reviews:
  First pass:  gemma-4-31b (cheap, catches 80% of issues — linting, security patterns, obvious bugs)
  Escalation:  deepseek-v4-pro (when gemma flags something complex, or for architecture-sensitive code)
```

**Reviewer prompt must include:**
- The PR number, repo, and diff summary
- The phase's Architecture Decisions from the plan (the reviewer MUST enforce these)
- A directive: "Review this PR for: (1) correctness against the acceptance criteria in the plan, (2) security issues (injection, exposed secrets, missing auth), (3) adherence to architecture decisions, (4) test coverage for new code. Use `mcp_github_create_pull_request_review` with event=APPROVE if clean, or REQUEST_CHANGES with inline comments if issues found."

### Step G5 — Review → Fix Loop

```
┌─────────────────────────────────────────────┐
│                                             │
│  Review returned?                           │
│   ├─ APPROVE → Merge PR (Step G6)           │
│   └─ REQUEST_CHANGES →                      │
│       ├─ Orchestrator reads review comments │
│       ├─ Assign fixer subagent              │
│       │   (model = original task's model)   │
│       ├─ Fixer pushes changes to PR branch  │
│       ├─ CI re-checked                      │
│       ├─ Re-assign reviewer                 │
│       └─ Loop until APPROVE or circuit break│
│                                             │
└─────────────────────────────────────────────┘
```

**Fixer subagent context:** Provide the review comments, the PR diff, and the task's original specification. The fixer should address each inline comment and push to the PR branch.

**Circuit breaker:** If the same category of issue is flagged 3+ times across review cycles, or 3 full review→fix→review cycles pass without resolution, **STOP and escalate to the user.** Do not let two AIs argue in circles. Report:
- The PR link
- The repeating issue category
- The review comments from the last cycle
- Recommendation for human intervention

### Step G6 — Squash-merge PR

Once all PRs for the phase are approved:
1. Squash-merge each PR using `mcp_github_merge_pull_request` with `merge_method: "squash"`.
2. Pull main locally: `git checkout main && git pull origin main`.
3. Clean up: delete the phase branch locally and on remote.

### Step G7 — Update PROGRESS.json and plan

Only NOW — after all PRs are merged to main — do the final marking:
1. Edit the plan file: append ` ✅` to every task in the phase's Task Dependency Graph entry.
2. Update PROGRESS.json: set all phase tasks to `"complete"`, set `"current_phase"` to the next phase number.
3. Commit and push BOTH files directly to main:
   ```bash
   git add PROGRESS.json plans/{NAME}_PLAN.md
   git commit -m "docs: complete Phase {N} — all PRs merged"
   git push origin main
   ```
4. If `plans/PLANS_REGISTRY.json` exists: increment `completed_tasks`, update `last_updated`, `last_agent`, and `current_phase`.

### Step G8 — Run Phase Verification Gate

Now run the phase verification gate task (N.V) against the **merged main branch** — not against feature branches. This gate proves the merged code works end-to-end.

1. Checkout main: `git checkout main && git pull origin main`
2. Run the verification commands from the Task N.V block in the plan
3. If verification passes: the phase is truly complete. Announce the next phase.
4. If verification fails: escalate to fixer subagent, loop until green (same circuit breaker rules apply).

**Phase is NOT complete until G1–G8 all pass.** Do NOT announce "Phase N complete" or dispatch tasks from Phase N+1 until every PR is merged and the verification gate passes on main.

## 8. Post-plan completion

When **all tasks in all phases** are marked ✅ (the plan is complete):

**Step A — Update the registry:**
- Set this plan's status to `"completed"` in `plans/PLANS_REGISTRY.json`
- Set `completed_at` to the current ISO timestamp
- This prevents the plan from being picked up again by future runs

**Step B — Refresh app-spec:**
1. **Check if `.app-spec/app-spec.json` exists.**
2. **If it exists:** invoke the `app-spec` skill in **Mode C (POST-PLAN REFRESH)**. Pass context:
   - The plan filename that just completed
   - A summary of what changed (new features added, schema migrations run, new routes/components, etc.) — derived from the completed tasks' descriptions and acceptance criteria
   - The list of files created/modified during plan execution (from `plans/{NAME}_PROGRESS.json` or the task reports)
3. **If it does NOT exist:** invoke the `app-spec` skill in **GENERATE mode**. A completed plan means there's now real code to document — this is a good time to create the spec.
4. **Report to the user:** *"Plan complete and marked as such in PLANS_REGISTRY.json. App-spec has been refreshed to reflect all changes made during plan execution. {N} features updated, {M} new entries added."*

This ensures the spec stays current as the codebase evolves through plan execution. The next plan or agent session will have an accurate picture of the codebase without re-scanning.

**Do NOT skip this step.** Stale app-specs cause downstream agents to make wrong assumptions. The refresh is cheap (Mode C only scans changed files) compared to the cost of agents working with outdated context.

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

- Check `plans/{NAME}_PROGRESS.json` for the last completed task before the failing one.
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

- Update `plans/{NAME}_PROGRESS.json`: set the failed task back to `"pending"`, add a `"rollback_reason"` note.
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

## Updating plan files on GitHub

Plan files are often 50–100KB+. The `mcp_github_create_or_update_file` tool requires inlining the full file content as a function parameter — for files over ~50KB, this frequently truncates. **Prefer the terminal+git approach for plan file updates:**

```bash
cd /opt/data/<repo> && git pull origin main && \
  cp /tmp/<updated-plan>.md plans/<PLAN>.md && \
  git add plans/<PLAN>.md && git commit -m "docs: mark task X.X ✅" && \
  git push origin main
```

This avoids the content-inlining size limit entirely. Use the local clone (created by subagent dispatch or `git pull`) rather than re-cloning each time.

For files under ~30KB, `mcp_github_push_files` or `mcp_github_create_or_update_file` are fine. If a push returns a 200 but the file size drops dramatically (e.g., 46KB → 6KB), content was truncated — revert and use terminal+git instead.

## Handling failures (run mode)

- **Agent says criteria failed:** do NOT mark ✅. Surface the failure, collect what went wrong, and offer to re-run with that as added context.
- **Agent edited files outside the task's scope:** flag this to the user — the worker broke rule 3 of the preamble. Let the user decide whether to keep the extra changes.
- **Build/test command fails:** treat as a failed criterion.
- **Vision-dependent task but model lacks vision:** See `references/ocr-extraction-pipeline.md` for the easyocr-based fallback pipeline. This converts image extraction into a text-parsing problem when the subagent model can't process images.
- **Subagent timeout (no response after 600s):** Check the dispatch log for the task's `dispatched_at` timestamp. Run GitHub-first recovery: `git fetch && git log origin/main --since="<dispatched_at>" --oneline`. If commits matching the task scope exist, the subagent completed — mark recovered. If no commits found after >15 min, re-dispatch. See Dispatch Log & Crash Recovery section for full procedure.

---

## Dispatch Log & Crash Recovery

The dispatch log (`plans/{NAME}_DISPATCH_LOG.jsonl`) is the event-sourced source of truth for every action taken by the orchestrator. PROGRESS.json is a human-readable summary — the dispatch log is the authoritative record.

### Schema (append-only JSONL)

```jsonl
{"ts":"2026-05-15T05:30:00Z","task":"2.3","model":"deepseek-v4-pro","action":"session_started","agent":"hermes"}
{"ts":"2026-05-15T05:30:05Z","task":"2.3","model":"deepseek-v4-pro","action":"dispatched","goal":"Add project member invite endpoint"}
{"ts":"2026-05-15T05:34:12Z","task":"2.3","model":"deepseek-v4-pro","action":"completed","commit":"abc123f","ac_total":4,"ac_passed":4}
{"ts":"2026-05-15T05:35:00Z","task":"2.4","model":"gemma-4-31b","action":"dispatched","goal":"Wire invite UI to endpoint"}
{"ts":"2026-05-15T05:35:47Z","task":"2.4","model":"gemma-4-31b","action":"failed","error":"subagent timeout at 600s"}
{"ts":"2026-05-15T05:36:00Z","task":"2.4","model":"glm-5.1","action":"re-dispatched","goal":"Wire invite UI to endpoint [retry with larger model]"}
{"ts":"2026-05-15T05:42:00Z","task":"2.4","model":"glm-5.1","action":"completed","commit":"def456g","ac_total":3,"ac_passed":3}
{"ts":"2026-05-15T05:42:10Z","task":"2.5","model":"deepseek-v4-pro","action":"dispatched","goal":"Add member list endpoint"}
{"ts":"2026-05-15T05:42:15Z","task":"—","model":"—","action":"session_ended","reason":"clean"}
```

**Fields by action type:**

| Action | Required fields | Optional fields |
|--------|----------------|-----------------|
| `session_started` | `ts`, `agent` | `task` (if resuming mid-task) |
| `session_ended` | `ts`, `reason` | `tasks_completed` |
| `dispatched` | `ts`, `task`, `model`, `goal` | — |
| `completed` | `ts`, `task`, `model`, `commit` | `ac_total`, `ac_passed` |
| `failed` | `ts`, `task`, `model`, `error` | — |
| `re-dispatched` | `ts`, `task`, `model`, `goal` | — |
| `recovered` | `ts`, `task`, `commit` | — |

### When to write each entry

| Action | Exactly when |
|--------|-------------|
| `session_started` | First action after finding the plan (step 1) |
| `dispatched` | Immediately BEFORE calling `delegate_task` (step 6a) |
| `completed` | After verifying subagent output passes all acceptance criteria + push+test+deploy gate (step 6d) |
| `failed` | After subagent returns with error, timeout, or failed criteria |
| `re-dispatched` | When re-dispatching a previously timed-out task |
| `recovered` | When GitHub recovery finds commits from a task that was `dispatched` but never `completed`/`failed` |
| `session_ended` | Last action before exiting (clean shutdown) |

### Crash recovery procedure

Run this at the start of EVERY session (step 1c):

```
1. Read last 5 lines of DISPATCH_LOG.jsonl (tail -5)
2. If last line is "session_started" with no matching "session_ended":
   → CRASH DETECTED
3. Find the most recent "dispatched" entry that has no matching "completed", 
   "failed", or "recovered" entry for the same task
4. Run GitHub-first recovery:
   a. git fetch origin
   b. git log origin/main --since="<dispatched_at>" --oneline
   c. grep for conventional commits matching the task scope
      (e.g., "feat(sharing): add member invite" for task about member invites)
   d. IF commit found:
      - Read commit diff, verify against task acceptance criteria
      - Write "recovered" to dispatch log
      - Mark task code-complete in PROGRESS.json
      - Push PROGRESS.json immediately
   e. IF no commit AND dispatched_at > 15 min ago:
      - Subagent definitely timed out or crashed
      - Write "re-dispatched" to dispatch log
      - Re-dispatch with identical context
   f. IF no commit AND dispatched_at < 15 min ago:
      - Subagent may still be running
      - Wait or check running processes
      - Only re-dispatch after 20 min staleness
5. If last line is "session_ended" → clean resume, no recovery needed
6. If dispatch log doesn't exist → first run, no recovery needed

ALWAYS write "session_started" before dispatching any task.
```

### Why PROGRESS.json isn't enough for recovery

PROGRESS.json tells you a task is `"in_progress"` but can't tell you:
- Was a subagent actually dispatched? (no — the orchestrator could have crashed while building the prompt)
- If dispatched, what model? (need this to know where to look for commits)
- Did the subagent push code? (PROGRESS.json is local — the commit could be on GitHub)
- How long ago was it dispatched? (need timestamps for the 15-min staleness heuristic)

The dispatch log answers all four questions. PROGRESS.json remains the human-readable summary; the dispatch log is the authoritative event source.

### Owner rule

Only the orchestrator (plan-runner) writes to PROGRESS.json and DISPATCH_LOG.jsonl. Subagents MUST NOT touch either file (enforced via Worker Instructions rule 8). This eliminates merge conflicts from parallel subagents and ensures state transitions are always sequential and consistent.

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

## pnpm Monorepo Execution Pitfalls

When executing plan tasks in a pnpm workspace monorepo, watch for these gotchas:

### Stub scripts (`echo ok`)

Phase 0 scaffolding typically creates placeholder scripts: `"test": "echo ok"`, `"build": "echo ok"`, etc. These make the root-level `pnpm test` pass but silently do nothing. **Before running TDD, check the target package's `package.json`** — if scripts are stubs, replace them with real commands (e.g., `"test": "vitest run"`, `"typecheck": "tsc --noEmit"`) and add the required devDependencies. Otherwise the RED step (test should fail) passes vacuously and you get a false positive.

### Missing generated files

Plans often reference files "created by Task X.X" (e.g., `db-types.ts` generated from Supabase). If the file doesn't exist at runtime — even though the plan says it was created — **create a minimal stub** so barrel exports compile. This happens when:
- A prior task claimed to create the file but didn't commit it
- The file is generated externally (Supabase MCP) and the generation wasn't run
- The session died before committing

Use the smallest valid stub (e.g., `export type Database = {}` for db-types). Flag it in the report and PROGRESS.json learnings so the next session knows to regenerate it.

### Adding devDependencies

When a task requires adding a devDependency (e.g., vitest), edit `package.json` and then **always run `pnpm install`** before running tests. The dep isn't available until pnpm resolves it into `node_modules/`. Forgetting this causes confusing "module not found" errors that look like import path issues but are actually missing packages.

### Concurrent agent pushes (multi-agent repos)

When multiple agents or cron jobs work on the same repo in parallel, the remote accumulates commits between your `git pull` and `git push`. This causes rejected pushes: `"Updates were rejected because the remote contains work that you do not have locally."`

**Rule: ALWAYS `git pull --rebase origin main` immediately before every `git push`.** No exceptions — even if you just pulled 5 minutes ago. Another cron job or agent may have pushed in the interim.

```bash
# Safe push pattern for multi-agent repos:
git pull --rebase origin main && git push origin main
```

If rebase produces conflicts:
1. Read the conflict markers — usually PROGRESS.json or plan dependency graph
2. Accept the upstream version (it has the latest state from other agents)
3. Re-apply your changes on top
4. Push again

**Stash unrelated changes** before pulling to avoid mixing work from different tasks:
```bash
git stash push -m "task X.X changes"
git pull --rebase origin main
git stash pop
```

### `write_file` linter false positives with tsconfig extends

The `write_file` tool has a built-in TypeScript linter that **does not resolve `extends` chains** in `tsconfig.json`. In a monorepo where child tsconfigs use `"extends": "../../tsconfig.base.json"`, the linter applies only the local tsconfig fields — missing `lib`, `moduleResolution`, and other inherited options. This produces false errors like:

### Orchestrator environment pnpm version incompatibility

The orchestrator's Node.js version may not support the pnpm version locked in the project's `pnpm-lock.yaml`. For example, Node.js v20 can't run pnpm ≥ 10 (requires `node:sqlite` built-in module added in v22). Running `pnpm install` fails with `ERR_UNKNOWN_BUILTIN_MODULE: node:sqlite`.

**Workaround:** Use `npx` with the exact pnpm version the lockfile was built with:
```bash
npx --yes pnpm@10.33.2 install --frozen-lockfile
npx --yes pnpm@10.33.2 --filter @fitlog/mcp-server test
```

`npx` downloads and runs the correct pnpm binary regardless of the system Node.js version — pnpm bundles its own Node.js. Do not globally `npm install -g pnpm` — npm may resolve the version tag incorrectly (e.g., `pnpm@8` resolves to pnpm 10.x because npm treats the version prefix as a semver range).

```
error TS2583: Cannot find name 'Map'. Do you need to change your target library?
error TS2705: An async function or method in ES5 requires the 'Promise' constructor.
error TS2307: Cannot find module 'rollup/parseAst' (from third-party node_modules)
```

**How to handle:**
1. **Trust `tsc --noEmit` over the write_file linter.** Always verify with the actual TypeScript compiler via terminal.
2. **Ignore write_file lint errors** that reference `ES5`, `Map`, `Set`, `Promise`, or `rollup/parseAst` — these are almost always false positives from missing base config.
3. **Real errors still surface** via `tsc --noEmit` and `vitest run`. If those pass, the write_file linter was wrong.
4. **Pre-existing errors in the same_tool_failure_warning loop** (count=3+) are a signal that the linter is stuck, not that your code is broken. Continue writing files and verify at the end with `tsc`.

This is especially common with Hono, Zod, and other libraries that use modern TS features in their type declarations.

---

## Edge cases

- **PROGRESS.json says complete but code isn't deployed/merged:** In multi-repo projects or when tasks span multiple sessions, PROGRESS.json can become stale — claiming a task is ✅ when the PR was never created or merged. ALWAYS verify by checking: (a) is the commit on the target branch? (b) does the remote have it? (c) was a PR actually merged? If PROGRESS.json is out of sync, fix it and push the correction. Trust git history over PROGRESS.json.
- **Plan missing Task Dependency Graph:** parse task headings for ✅ markers instead. Warn the user that without a graph you can't reason about cross-phase deps.
- **Task missing Tier/model:** default to `capable`. Resolve the model from `hermes-model-config.json` (if loaded) using `plan_tier_mapping.tier_worker`, otherwise default to Sonnet. Note this in the preview.
- **Task missing Acceptance criteria:** refuse to dispatch. Tell the user the plan is under-specified for this task and suggest using `create-plan` to patch it.
- **All tasks ✅:** mark the plan as `"completed"` in `plans/PLANS_REGISTRY.json`, congratulate the user; mention any "Optional / Future" or Phase N+ sections that exist but weren't in scope.
- **User asks to run multiple tasks in parallel:** dispatch multiple sub-agents in one message, one per task, but only if the tasks have no overlapping file writes. Otherwise serialize and explain why.
- **Plan file is huge (>2000 lines):** read only Architecture Decisions + Worker Instructions + Context Files + the chosen task's block + Task Dependency Graph.
- **User references a task number that doesn't exist:** list the available task numbers and ask them to pick again.
- **Test task but no test specs exist yet:** this is a development task (write the specs), not a test-mode task. Dispatch via run/prep mode.
- **Plan missing mock strategy for test mode:** warn the user that D-classification fixes will be guesswork without a documented mock layer. Suggest updating the plan.

---

## Relationship to `create-plan` and `app-spec`

If the user starts a session by saying "I want to build X" and there is no active plan file in `plans/`, recommend the `create-plan` skill first. `create-plan` is the upstream authoring skill (Opus-class). This skill — `plan-runner` — is the downstream execution dispatcher that handles:

- **Development tasks** via run/prep mode (scaffolding, new spec files, feature implementation, mock-layer extensions).
- **Test execution** via test mode (running existing specs, fixing bugs those tests expose, verification gates).
- **Deploy safety** via the rollback strategy (reverting broken deploys, maintaining known-good state).
- **Spec maintenance** — after plan completion, marking it complete in `plans/PLANS_REGISTRY.json` and refreshing `.app-spec/app-spec.json` so the next plan/agent session has current context.

Typical sequence for a new project:
1. `app-spec` generates `.app-spec/app-spec.json`, `.app-spec/APP_SPEC_SUMMARY.md`, and `.app-spec/DEPENDENCY_GRAPH.md`.
2. `create-plan` reads the app-spec and writes the plan to `plans/{NAME}_PLAN.md`, registers it in `plans/PLANS_REGISTRY.json`.
3. Human completes Phase GATE (provides credentials, tokens, config).
4. `plan-runner` (run mode) completes the foundation phase (scaffolding, mocks, page-object base).
5. `plan-runner` (run mode) writes specs for each feature phase.
6. `plan-runner` (test mode) runs the verification gate at the end of each phase.
7. If test mode escalates, the user decides whether to re-dispatch via run mode with added context or intervene manually.
8. For deploy tasks, `plan-runner` appends deploy safety rules and monitors for rollback triggers.
9. **After all tasks complete**, `plan-runner` marks the plan `"completed"` in `plans/PLANS_REGISTRY.json` and invokes `app-spec` in POST-PLAN REFRESH mode to update the spec with all changes made during execution.

This creates a virtuous cycle: `app-spec` → `create-plan` → `plan-runner` → `app-spec` (refresh) → next `create-plan` session has fresh context. Completed plans are never re-executed thanks to the registry.
