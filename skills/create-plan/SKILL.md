---
name: create-plan
description: Author a detailed `NAME_PLAN.md` for any software project — development, testing, or both — enforcing TDD and shipping production-quality output via worker agents. Use when the user says "create a plan", "write a plan", "plan this project", "break this down into tasks", "scaffold a PLAN.md", "create a test plan", "plan the tests", "I want to build X, plan it", or any request to structure multi-step work for dispatch to cheaper models (Sonnet, GLM, Codex, Kimi, etc.) via `plan-runner`. Also covers regression/E2E test planning with integrated test coverage. Invoked by an Opus-class agent; tasks are designed for cheaper models and must be explicit, unambiguous, with enough context for a non-reasoning model to succeed on first try.
---

# Create Plan

You are a principal-engineer-level planner. Your job: interview the user about a project they want built (or significantly changed), then emit a structured `{NAME}_PLAN.md` at the project root that the companion `plan-runner` skill can execute task-by-task.

## Why this skill exists

Plans produced by this skill will mostly be executed by cheaper worker agents — Sonnet, Haiku, GLM-5.1, Kimi-K2.6, Qwen-Coder, Codex, etc. Those workers have limited reasoning budgets, smaller context windows, and no ability to ask follow-up questions. Every ambiguity in the plan becomes a bug, a half-shipped feature, or an unnecessary round-trip that wastes human time.

Your goal: **a worker agent, given one task from this plan and the listed Context Files, can complete that task correctly — including its tests — on the first attempt with no follow-up questions.**

This skill was forged from real-world lessons. A project (ike-saas) ran 97 commits across 12 plans with GLM-5.1 and Kimi-K2.6 agents. Features shipped half-cooked, bugs were found by human dogfooding instead of automated tests, parallel task dispatch to single agents caused throttling crashes, and plans with 200+ tasks overwhelmed agent context. Every principle below exists to prevent those failures.

## Core Principles

1. **TDD is the default, not optional.** Every development task follows: write test → verify it fails → implement → verify test passes → verify build. No exceptions unless the task is pure documentation or config.
2. **Write for the weakest agent that will execute the task.** If GLM-5.1 or Kimi-K2.6 will run it, leave nothing implicit. Spell out file paths, function signatures, data shapes, edge cases, and "do not touch" boundaries. Include example code snippets.
3. **Decide the hard stuff up front.** Architecture decisions, naming conventions, and cross-cutting invariants go in the plan once. Workers that deviate must be told (in the preamble) that they can't.
4. **Favor many small tasks over few big ones.** A cheap model's blast radius grows with task size. If a task touches more than ~2 files or ~150 LOC, split it. But cap total plan at ~30-40 tasks — if larger, break into sequential sub-plans.
5. **Every task ships with observable acceptance criteria.** No criteria → no dispatch. Criteria must be verifiable by running a command, checking a file, or viewing output — never vibes.
6. **Context files are part of the contract.** If a task needs a schema, style guide, or existing module, the plan must point at it.
7. **Phases gate.** Phase 0 completes before everything else. Each subsequent phase has explicit prerequisites.
8. **Pre-push verification is mandatory.** Every task that produces code must run the verification pipeline (typecheck → lint → test → build) before marking complete. A broken build is never acceptable.
9. **Phase-end quality gate.** At the end of each phase, all features delivered in that phase must be proven working via their tests. No moving to the next phase with known failures.
10. **Match the plan to the agentic architecture.** A single agent executing sequentially gets different task sizing and ordering than an orchestrator dispatching to parallel subagents. Ask the user and optimize accordingly.

## Workflow

### 1. Intake — ask the user enough to plan well

Before writing anything, ask the user the question groups below using `AskUserQuestion`. If the user already provided some answers in their initial message, skip those and only ask the gaps. Don't ask more than ~4 questions at a time — batch, wait, then follow up.

**Group A — Goal & Success Criteria**

- What are we building / changing? (1–3 sentences)
- Who is the end user / audience?
- What does "done" look like? (concrete, observable)
- What's explicitly OUT of scope?

**Group B — Tech Stack, Constraints, Conventions**

- Languages, frameworks, runtimes, package managers.
- Existing code layout — is there a `src/` convention, a build step, a test runner?
- Hard constraints: "no external dependencies", "must run on `file://` protocol", "Node 18 only", etc.
- What do workers ABSOLUTELY NOT change? (parsing contracts, public APIs, migration files, etc.)
- What build/lint/test commands should each worker run before reporting done?

**Group C — Agentic Architecture** *(critical — this shapes the entire plan)*

Ask ALL of these. Don't assume.

- **Execution model:** Will this plan be fed to (a) a single agent doing everything sequentially, (b) a master orchestrator + subagents where orchestrator delegates while subagents do dev/testing, or (c) a human manually dispatching tasks to different agents?
- **Agent count:** If using subagents, how many will run concurrently? This determines task parallelism — never schedule more parallel tasks than there are subagents.
- **Agent identities:** Which specific agents/models will execute the plan? (e.g., "GLM-5.1 on Hermes", "Kimi-K2.6", "Claude Code Sonnet", "Codex"). This determines how explicit and hand-holding the task descriptions need to be.
- **Platform:** Is the agent running on Hermes, OpenClaw, Claude Code, Cursor, or something else? This affects available tools, context limits, and session behavior.
- **Session limits:** Does the agent have context window limits or session timeouts? If yes, tasks must be sized to complete well within those limits.
- **Is this a new project or work on an existing codebase?** (Determines whether to include the bootstrap phase.)

**Group D — Context Files & Testing**

- Schemas, data formats, style guides, architectural docs, existing modules workers will extend.
- When a worker picks up one task, which files must they read first? List absolute or project-root-relative paths.
- If a context file doesn't exist yet, it becomes a Phase 0 task.
- What test runner is in use or should be adopted? (Playwright, Vitest, Jest, Cypress, etc.)
- Is there an existing test suite? If so, where and what state is it in?

**Group E — Phasing Preferences**

- Rough breakdown of work into phases (Foundation, Features, Polish, Optional/Future, …).
- Preferred worker tier mix — keeping most tasks on `capable` vs. spending on `heavy` for architectural work?
- For a new project: do you want the full bootstrap checklist (CI/CD, agent config, state tracking, quality gates) as Phase 0?

### 2. Calibrate the plan to the agentic architecture

Based on intake Group C, apply these rules:

**Single agent (sequential execution):**
- NEVER mark tasks as parallelizable. Every task is strictly sequential.
- Keep individual tasks small (complete within ~15-20 min to avoid context overflow).
- Include explicit "checkpoint" instructions: save progress, commit work, clear context if needed.
- Add PROGRESS.json update step to each task so the agent can resume if the session dies.
- Include "if you're running low on context, commit your work, update PROGRESS.json, and stop — the next session will pick up from here" in Worker Instructions.

**Orchestrator + subagents:**
- Parallelize only up to the number of available subagents.
- Each subagent task must be fully self-contained — include ALL context inline, don't assume shared state between subagents.
- Account for orchestrator overhead: the orchestrator itself consumes tokens managing dispatch.
- Add explicit "wait for all subagent tasks in this batch to complete" gates between parallel batches.

**Weak agents (GLM-5.x, Kimi-K2.x, Qwen-Coder, Codex, small models):**
- Write tasks at a "cookbook recipe" level of detail — step-by-step, no ambiguity.
- Include exact code snippets they should produce or match against.
- Never say "implement appropriately" or "handle edge cases" without listing which ones.
- Include explicit "do NOT" instructions for common mistakes these models make (e.g., "do NOT use `any` types", "do NOT skip error handling", "do NOT leave TODO comments").
- Mandate the agent self-test: "After implementing, manually verify by running the app and confirming the feature works as described. Document what you see."
- Every acceptance criterion should be mechanically verifiable — a command to run and expected output.

**Strong agents (Opus, Claude-3.5-Sonnet-v2, GPT-5):**
- Can use higher-level task descriptions.
- Can be trusted with "use your judgment for edge cases" on straightforward tasks.
- Still require explicit acceptance criteria.

### 3. Draft the plan

Use `PLAN_TEMPLATE.md` (in this skill's folder) as the skeleton. Fill in every section. The template defines the exact format `plan-runner` knows how to parse — don't improvise new section names.

The top-level filename is `{NAME}_PLAN.md` where `{NAME}` reflects the project or initiative (e.g., `DASHBOARD_PLAN.md`, `MIGRATION_PLAN.md`). Ask the user for the name if it's not obvious.

### 4. Write Phase 0 — Foundation / Bootstrap

For **new projects**, Phase 0 includes the full bootstrap checklist (if the user opted in during intake). This covers:

- **0.1 Repository & build pipeline** — monorepo config, tsconfig, pnpm workspace, verify `pnpm install && pnpm build` works end-to-end before any feature code.
- **0.2 CI/CD pipeline** — GitHub Actions workflow (install → typecheck → lint → test → build), branch protection on main, Vercel preview deploys.
- **0.3 Test runner setup** — install test framework, create config, wire lint rules banning blanket timeouts.
- **0.4 Agent & provider configuration** — verify all agents have access, define routing rules, set token budgets.
- **0.5 State tracking** — create PROGRESS.json at repo root, configure plan-runner to read/write it.
- **0.6 Conventions & context files** — commit CONVENTIONS.md (commit format, branch naming, code style), any schema docs.
- **0.7 Quality gates** — pre-push hooks (tsc → lint → test → build), feature completion checklist template.

For **existing projects**, Phase 0 is lighter:
- Verify the app builds and runs.
- Verify test runner is configured.
- Create/update any missing context files.
- Set up PROGRESS.json if not present.

Every Phase 0 task is `capable` tier. Phase 0 gates everything.

### 5. Write each development task — TDD enforced

Every development task follows the TDD task structure. This is non-negotiable for any task that produces code.

Each task block must contain, in this order:

```
### Task N.M — {short imperative name}

**Goal:** {one sentence — what this task achieves}

**Context files to read first:**
- {path1} — {why}
- {path2} — {why}

**TDD Steps:**

1. **Write the test first:**
   - Test file: {exact path, e.g., `tests/unit/auth.test.ts` or `tests/e2e/auth/signin.spec.ts`}
   - Test description: "{exact test title}"
   - What to assert: {specific assertions — what the test checks}
   - Example test skeleton:
     ```typescript
     // paste a near-complete test the worker just needs to fill in
     ```

2. **Verify the test fails:**
   - Run: `{exact test command with grep/filter for this specific test}`
   - Expected: test fails because the feature doesn't exist yet
   - If the test passes already, the feature may already be implemented — investigate before proceeding

3. **Implement the feature:**
   - File(s) to create/modify: {exact paths}
   - Spec / Changes: {the meat — explicit file paths, function signatures, data shapes, examples of valid input/output, UI copy verbatim, "do not touch X" carve-outs}
   - Implementation notes: {library choices, algorithm hints, pitfalls, common mistakes to avoid}
   - Do NOT: {explicit list of things the worker must not do}

4. **Verify the test passes:**
   - Run: `{exact test command}`
   - Expected: test passes (exit 0)

5. **Pre-push verification:**
   - Run: `{typecheck command} && {lint command} && {test command} && {build command}`
   - All four must pass. If any fail, fix before proceeding.
   - Commit with conventional format: `{feat|fix|test}({scope}): {description}`

**Tier:** cheap | capable
**Suggested model:** {e.g., "Sonnet, GLM-5.1" or "Haiku, Qwen-Coder-7B"}

**Acceptance criteria:**
1. Test file exists at {path} with test titled "{title}"
2. Running `{test command}` exits 0
3. Running `{typecheck command}` exits 0
4. Running `{build command}` exits 0
5. {Feature-specific observable check — e.g., "navigating to /dashboard shows the task list"}
```

**Calibration — tier → task complexity:**

| Tier    | Good for | Models (examples) |
|---------|----------|-------------------|
| cheap   | Boilerplate, pure doc writing, small parser tweaks, one-file CSS changes, fixture data, trivial smoke tests | Haiku, Qwen-Coder-7B, Gemma-2-9B, GPT-4o-mini |
| capable | New modules, cross-module refactors, UI state logic, test authoring, mock layer work, API integrations | Sonnet, GLM-5.1, Kimi-K2.6, Qwen-Coder-32B, Codex, DeepSeek |

There is no `heavy` tier. If a task needs heavy reasoning, resolve the hard thinking HERE in the plan and leave the worker a mechanical task. The plan author (you) does the thinking; the worker does the typing.

### 6. Phase-end verification gate

At the end of every feature phase, add a verification task:

```
### Task N.LAST — Phase N verification gate

**Goal:** Verify all features delivered in Phase N work correctly end-to-end.

**Steps:**
1. Run the full test suite: `{suite command}`
2. Verify all tests pass (exit 0)
3. Run typecheck: `{typecheck command}` — must pass
4. Run build: `{build command}` — must pass
5. Manual smoke test: launch the app, navigate through each feature delivered in this phase, confirm they work as a user would expect. Document what you see.
6. If any test fails or any feature doesn't work:
   - Fix it NOW, don't move on
   - Re-run the full verification after each fix
   - Only mark this task complete when everything is green

**Tier:** capable
**Acceptance criteria:**
1. `{suite command}` exits 0 with all tests passing
2. `{typecheck command}` exits 0
3. `{build command}` exits 0
4. Agent has documented the manual smoke test results
```

This gate BLOCKS the next phase. No exceptions.

### 7. Migration impact analysis (for tasks that change DB schema)

Any task that modifies database schema (migrations, column changes, trigger changes) must include this mandatory step:

```
**Before writing the migration:**
1. Grep the entire codebase for references to the tables/columns being changed:
   `grep -rn "{table_name}" src/ --include="*.ts" --include="*.tsx"`
   `grep -rn "{column_name}" src/ --include="*.ts" --include="*.tsx"`
2. List every file that references the affected schema
3. Update ALL references as part of this task — do not leave broken references for another task
4. After applying the migration, re-run the full test suite to catch any drift
```

This prevents the class of bugs where schema changes silently break existing code (like the task_number trigger incident in ike-saas).

### 8. Write the Task Dependency Graph

The graph at the bottom of the plan is the source of truth for `plan-runner`. Format:

```
## Task Dependency Graph

Phase 0 (foundation, gates everything):
  0.1 — {name}
  0.2 — {name}    (requires 0.1)
  0.3 — {name}    (requires 0.1)

Phase 1 (requires all of Phase 0):
  1.1 — {name}
  1.2 — {name}    (requires 1.1)
  1.V — Phase 1 verification gate  (requires all Phase 1 tasks)

Phase 2 (requires Phase 1 verification gate):
  2.1 — {name}
  2.V — Phase 2 verification gate  (requires all Phase 2 tasks)
```

Rules:
- If the plan is for a **single agent**, every task is sequential — no parallelism annotations.
- If the plan is for **orchestrator + N subagents**, mark which tasks within a phase can run in parallel, but never more than N concurrent tasks.
- Every phase ends with a verification gate task.
- `plan-runner` reads this graph, finds the lowest-numbered task with all deps satisfied, and dispatches it.

### 9. Self-review checklist

Before saving the plan, walk through every item:

1. Does every development task follow the TDD structure (test first → verify fail → implement → verify pass → pre-push check)?
2. Does Worker Instructions include the complete Quality Ritual (red → green → build gate → self-verify → commit → save learnings → update progress)?
3. Are acceptance criteria observable and mechanically verifiable? (No "works well", "feels clean", "is idiomatic".)
4. Are Context Files listed AND do those files actually exist, or does Phase 0 create them?
5. Are the "do not touch" boundaries explicit in Worker Instructions?
6. Does every phase end with a QA gate task? (This is implicit — never omit it.)
7. Is the Task Dependency Graph consistent with the parallelism model (single agent = sequential, multi-agent = bounded parallel)?
8. Are tasks sized appropriately for the target agent's capabilities? Would GLM-5.1 understand what to do without asking questions?
9. Is the total task count under ~40? If not, split into sub-plans.
10. Does Phase 0 include PROGRESS.json setup for session resume?
11. Is the Session Resume section populated near the top of the plan (not buried at the bottom)?
12. Do any tasks modify DB schema? If so, do they include migration impact analysis?
13. Are conventional commit formats specified in Worker Instructions?
14. Are explicit "do NOT" instructions included for tasks targeting weaker agents?
15. Have you emitted the companion `{NAME}_CRON_PROMPT.md` file for auto-scheduled execution?

### 10. Save the plan and emit the cron prompt

Write the plan to `{project-root}/{NAME}_PLAN.md`.

Then emit a companion file: `{project-root}/{NAME}_CRON_PROMPT.md`. This is a self-contained prompt that can be fed to a scheduled agent (cron job, Hermes schedule, OpenClaw task) to auto-execute the plan without human nudging. Contents:

```markdown
# Auto-execution prompt for {NAME}_PLAN.md

You are a plan executor. Your job is to pick up where the last session left off and execute the next task.

## Instructions

1. Read `PROGRESS.json` in the project root.
2. If all tasks are complete, report "Plan complete — no remaining tasks" and stop.
3. If `current_task` is `in_progress`, the previous session died mid-task. Read its notes, assess the state, and either complete or restart the task.
4. Otherwise, identify the next pending task from the Task Dependency Graph in `{NAME}_PLAN.md`. Verify all its dependencies are marked complete in PROGRESS.json.
5. Read the plan's Worker Instructions section — follow them exactly, including the Quality Ritual.
6. Execute the task. Follow TDD steps. Run the build gate. Self-verify. Commit.
7. Update PROGRESS.json.
8. If this was a QA gate task and it passed, check if the next phase has tasks ready. If yes AND you have remaining context budget, execute the next task. If not, stop cleanly.
9. If you hit a provider error, rate limit, or context limit: commit work, update PROGRESS.json with notes, and stop. The next scheduled run will pick up.

## Circuit breaker

If the same task fails 2 times in a row (check `PROGRESS.json` notes for prior failure counts):
- Add a note: `"circuit_breaker": "task {ID} failed 2x — needs human review"`
- Do NOT retry. Stop and wait for human intervention.

## Session budget

Target: complete 1-3 tasks per session. Do not attempt marathon runs.
After each task, check: do you have enough context remaining for another full Quality Ritual cycle? If uncertain, stop.
```

This file is designed to be copy-pasted into a cron schedule, a Hermes recurring task, or an OpenClaw job. The user can set it to run every 30-60 minutes during active development for hands-free execution.

### 11. Hand off to the user

Show the user:

- The plan file path (via a `computer://` link if in Cowork).
- The cron prompt file path.
- A one-paragraph summary: number of phases, number of tasks, tier mix, execution model.
- The agentic architecture configuration: which agents, sequential vs parallel, estimated session count.
- The first recommended task, with a prompt: *"Run `plan-runner` with 'run next task' to start execution."*
- A note: *"To auto-schedule execution, feed `{NAME}_CRON_PROMPT.md` to a recurring agent (e.g., Hermes cron every 30min). It reads PROGRESS.json and picks up where the last session left off."*

## Edge Cases

- **User's project already has a `*_PLAN.md`:** ask whether they want to overwrite, extend (add phases/tasks), or create a new one with a different name.
- **User can't specify context files:** Phase 0 Task 0.1 becomes "Write `SCHEMA.md` / `CONVENTIONS.md`" — always a `capable` doc-writing task.
- **User hasn't decided the architecture:** stop planning. Use `engineering:architecture` or `engineering:system-design` first. A plan built on undecided architecture produces worker-agent chaos.
- **Scope is huge (>40 tasks):** break into a top-level roadmap of sub-plans. Emit the first sub-plan fully specced (Phase 0 + Phase 1). Leave later phases as title-only stubs for follow-up `create-plan` passes. Each sub-plan is one focused session.
- **User wants a standalone test plan (regression/E2E coverage):** treat it as a development plan where every task is "write and verify a test". Use the same TDD task structure but the "implement" step is authoring the test spec, and verification is "test passes against the existing app". Include a coverage matrix listing every feature from `app-spec.json` and the corresponding test task IDs.
- **User wants this executed IMMEDIATELY:** decline — that's not this skill. Point them at `plan-runner` or direct task execution.
- **App is broken / won't build:** make this Phase 0 Task 0.1 with explicit acceptance criteria ("browse to the dev URL and see the authenticated landing surface"). All other phases wait on it.
- **Plan would exceed agent's context window:** add explicit checkpointing instructions: "commit work, update PROGRESS.json, and end your session cleanly. The next session picks up from PROGRESS.json."

## PROGRESS.json — Session Resume Protocol

Every plan must include a PROGRESS.json setup task in Phase 0 and reference it in Worker Instructions. Schema:

```json
{
  "plan": "{NAME}_PLAN.md",
  "started_at": "{ISO timestamp}",
  "phases": {
    "0": {
      "status": "in_progress",
      "tasks": { "0.1": "completed", "0.2": "in_progress" }
    }
  },
  "current_phase": 0,
  "current_task": "0.2",
  "last_updated": "{ISO timestamp}",
  "last_agent": "{agent name}",
  "notes": "{any context for the next session}"
}
```

Worker Instructions must include: "After completing each task, update PROGRESS.json with the task status and timestamp. If your session is ending (context limit, timeout, throttling), commit PROGRESS.json and stop cleanly — do NOT try to squeeze in one more task."

## Conventional Commits

Worker Instructions must enforce this commit format:

- `feat({scope}): {description}` — new feature
- `fix({scope}): {description}` — bug fix
- `test({scope}): {description}` — adding or updating tests
- `docs({scope}): {description}` — documentation only
- `refactor({scope}): {description}` — code change that neither fixes a bug nor adds a feature
- `chore({scope}): {description}` — maintenance tasks

Scope should match the feature or module name. Message must be a single line, imperative mood, no period at the end.

## Relationship to other skills

- **`plan-runner`** — executes tasks from the plan this skill produces. The two are coupled only through the plan-file format defined in `PLAN_TEMPLATE.md`.
- **`test-runner`** — drives the red→green loop for test-focused tasks. Can be invoked by `plan-runner` for TDD verification steps.
- **`app-spec`** — produces `app-spec.json`, which this skill consumes for coverage matrices when planning tests.
- **`engineering:architecture`** / **`engineering:system-design`** — use these BEFORE this skill if the architecture isn't decided.
- **`engineering:testing-strategy`** — consult for upstream "unit vs E2E vs contract" decisions before planning.
