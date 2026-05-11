---
name: bug-fix-orchestration
description: Investigate, plan, delegate, and verify bug fixes — orchestrator workflow for when users report issues. Investigate root causes yourself, then delegate all code changes to subagents.
version: 1.0
---

# Bug Fix Orchestration

## When to Use

Use when the user reports one or more bugs and you need to fix them. You act as **investigator and orchestrator** — never writing code directly. Pattern: investigate → plan → delegate → verify.

## The Process

### 1. Investigate Before Delegating

**Never delegate blindly.** Understand the root cause first so you can give the subagent precise instructions.

- Read the relevant source files
- Trace the code path from UI action → data layer → backend
- Identify the ONE root cause (many "bugs" are symptoms of one issue)
- Check for hidden dependencies: DB triggers, conditional rendering branches, optional props, environment configs

**Common traps:**
- **React Query mutation invalidation gaps:** When a mutation affects multiple query keys, you MUST invalidate ALL of them. E.g. `useSetTaskTags` invalidated `['taskTags', taskId]` and `['tasks']` but NOT `['workspaceTaskTags']` — so the tags column (powered by `workspaceTaskTags`) never refreshed after toggling tags. Rule: trace ALL consumers of the mutated data and invalidate every query key they use.
- **CSS `position: relative` inside table cells:** If an inline popover/editor inside a `<td>` uses `position: relative`, it expands the row height. Fix: make the `<td>` `relative` and the popover `absolute` so it overlays without affecting layout.
- **No-op callback props:** When a component receives `onClose` but the parent passes `onClose={() => {}}`, clicking close does nothing. Common when the parent doesn't want the child to close independently (drawer vs modal), but confuses users. Always pass the real handler or document why it's intentionally a no-op.
- **Modal callbacks that unconditionally close:** If `onCreated` always closes the modal, "Create & Next" can't work. Fix: add a `keepOpen` parameter to the callback: `onCreated?.(result, false)` for normal create, `onCreated?.(result, true)` for create-and-next. The parent checks `if (!keepOpen) closeModal()`.
- **DB triggers + RLS (RECURRING PATTERN):** Triggers that query RLS-protected tables MUST be `SECURITY DEFINER` or they run as the invoking user and get RLS-filtered (empty) results. Check `prosecdef` column in `pg_proc`. Fix: `ALTER FUNCTION func_name(args) SET security_definer`. This has caused multiple bugs: (1) signup trigger couldn't read workspace tables for new user provisioning, (2) task number trigger got NULL from `max(task_number)` → duplicate key on unique index. Before changing a value that feeds into a trigger, read the trigger source (`SELECT prosrc FROM pg_proc WHERE proname = 'trigger_name'`). A change like `null → 0` can silently disable a trigger that checks `if new.value is null`. **PostgREST vs direct SQL:** After fixing a DB trigger, verify with BOTH PostgREST (the JS client) AND direct SQL via Management API (`POST /v1/projects/{ref}/database/query`). PostgREST may handle NULL differently than raw SQL — a fix that works in direct SQL can still fail through the REST client. See `supabase-development` skill for the full PostgREST NULL pitfall.
- **Conditional rendering:** When adding a component to a page with `isCollapsed ? (A) : (B)` branches, check BOTH branches. Adding to only one means it disappears when the state toggles.
- **Optional props:** If a button calls `onAction?.(arg)` and the parent never passes `onAction`, clicking does literally nothing. Check all call sites.
- **SPA redirect debugging — systematic checklist:** When a page silently redirects to `/` or another route, check in this order: (1) Route definition — is the path registered in the router? (2) **Nested route matching (CRITICAL)** — with React Router v6, a parent `path="/*"` with inner `<Routes>` containing absolute paths like `/admin/changelog` is UNRELIABLE. The catch-all `path="*"` often matches first. **Fix: use the `<Outlet>` pattern** — the parent layout component renders `<Outlet />` instead of `<Routes>`, and child routes are defined as nested `<Route>` elements under the parent. This is the idiomatic RR6 approach and avoids all matching ambiguity. (3) Auth guard timing — does `RequireAuth` pass but the page component's own useEffect clears loading state prematurely when `workspace` is still null? (4) ErrorBoundary — does the component throw during render, causing ErrorBoundary's "Go Home" button? (5) Vercel rewrites — `vercel.json` must have `{ "source": "/(.*)", "destination": "/index.html" }` for SPA routing. (6) Deployment state — the build serving the route must include it; check `mcp_vercel_getDeployments` for the latest READY deployment's commit SHA vs the commit that added the route. (7) Browser cache — stale JS from a previous deploy won't have the new route; test in incognito.
- **Form submission races:** HTML forms without `method` attribute default to GET. In SPAs with catch-all rewrites, this causes a full page reload before React's `onSubmit` fires. Fix: add `method="post"` to the `<form>`.
- **sanitizeErrorMessage for user-facing errors:** Map raw Postgres error codes/patterns to user-friendly messages. Never let DB error strings reach the UI — they expose internal schema details. Use `code` or `message` pattern matching (e.g. `duplicate key`, `invalid input syntax`).
- **Auth signup failures:** When "Database error saving new user" or similar auth errors appear, check `mcp_supabase_get_logs(project_id, service="auth")` FIRST. Auth logs show the exact SQL error from trigger functions (e.g. column name mismatches after Supabase version upgrades). This is faster than guessing from the frontend error message.
- **Edge function "silent success" — PostgREST insert failures:** When an edge function returns 200 but doesn't create records, the insert is likely failing with a PostgREST error (400) that's caught by `if (!insertRes.ok) { skipped++ }` — so the function reports success with `{ created: 0 }`. Common causes: (1) NOT NULL column received null value, (2) RLS policy blocks service_role (unlikely — service_role bypasses RLS natively), (3) missing table grant. Debug: add `console.error(await insertRes.text())` in the else branch, deploy, trigger, check logs. Also check `information_schema.columns` for `is_nullable = 'NO'` columns and ensure your code defaults them.
- **Supabase edge function logs don't show console.log:** The `mcp_supabase_get_logs` API only returns HTTP method, status code, execution time, and timestamp — NOT `console.log` output. To debug edge function logic, you must either: (a) return debug info in the response body, (b) write to a DB table, or (c) use the Supabase Dashboard's real-time log viewer which may show more detail.

### 2. Write a Plan File

Create `plans/<BUG_NAME>_PLAN.md` with:
- Each bug's root cause (1-2 sentences)
- Exact file paths and line numbers to change
- What the fix should be (specific enough for a non-reasoning model)
- Files that should NOT be changed
- Constraints (tsconfig settings, theme tokens, no external deps)

**Why write a file:** Subagents get clean context. You don't repeat yourself if the subagent times out and you need to re-delegate. Plan persists across context compactions.

### 3. Delegate with Full Context

Give the subagent EVERYTHING it needs:
- Root cause explanation (why the bug exists)
- Exact files and what to change
- Existing components to reuse (e.g. "use ConfirmDialog from ./FormModal")
- Project constraints
- Verification commands (exact commands to run)
- Commit message

**Key:** Include the plan file path so the subagent can read it for full details. But also include the essential info directly in the context — don't make the subagent hunt.

### 4. Verify the Deploy (MANDATORY — do NOT skip)

**ALWAYS check deployment state after pushing. Do not report success until ALL deploy targets are verified READY.**

#### Vercel (apps/web)

```bash
# After git push, wait ~30s then:
mcp_vercel_getDeployments(projectId="prj_v37jQJsTuT4g9QXt1bzGZZaluial", limit=1)
# Confirm state is "READY" and readySubstate is "PROMOTED"
```

#### Cloudflare Workers (apps/mcp-server)

**CRITICAL PITFALL:** Source changes to `apps/mcp-server/src/` have ZERO effect until the worker is deployed. Git push does NOT deploy to Cloudflare. Tests passing locally means nothing if the deployed worker is stale.

```bash
cd apps/mcp-server && npx wrangler deploy --name ike-mcp
# Then VERIFY with an actual MCP tool call:
mcp_ike_get_task(task_id="<known task id>")
# Confirm the response shape matches expectations
```

**This applies to ALL deployable targets in the IKE monorepo:**
- `apps/web/` → Vercel (auto-deploys on git push to tracked branches)
- `apps/mcp-server/` → Cloudflare Workers (manual `wrangler deploy` required)
- `supabase/functions/` → Supabase Edge Functions (manual deploy via MCP or CLI required)
- Database migrations → Applied via `mcp_supabase_apply_migration` (manual, NOT auto)

**If ANY deploy target needs a manual step, do NOT report "fixed" until that step is complete AND verified.**

#### Vercel Build Failure Response

If the Vercel build fails:
1. Read the build logs: `mcp_vercel_getDeploymentEvents(deploymentId, limit=20)`
2. Understand the error (TS compile error, missing dep, build script failure)
3. Fix and push again
4. **Do NOT mark the bug as fixed until Vercel shows READY**

Common Vercel failures:
- `.npmrc` with `prefix` setting → remove from git, add to `.gitignore`
- TS errors in `tsc -b` → fix before pushing (Vercel treats all TS errors as fatal)
- Missing env vars → check Vercel dashboard
- Monorepo workspace resolution → check `vercel.json` Root Directory and installCommand
- Migration `NOT NULL` columns added but INSERT paths not swept → Vercel build succeeds but runtime breaks (see deployment protection: may return 401 with `_vercel_sso_nonce` cookie — turn off Deployment Protection for private repos)

```bash
# After git push, wait ~30s then check:
# Use mcp_vercel_getDeployments with app="ike-saas-web" and limit=1
# Confirm state is READY and readySubstate is PROMOTED
```

If the build fails, read the error, understand it, and delegate a fix. Common Vercel failures:
- `.npmrc` with `prefix` setting → remove from git, add to `.gitignore`
- TS errors in `tsc -b` → fix before pushing (Vercel treats all TS errors as fatal)
- Missing env vars → check Vercel dashboard

### 5. Verify Visually (When Possible)

After deployment, use the browser to:
- Navigate to the live app
- Log in (note: browser_click may not trigger React synthetic events — use `browser_console` to dispatch events via JS if needed)
- Reproduce the original bug scenario
- Confirm the fix works

### 6. CRUD Test Mandate for Data-Mutating Features

**CRITICAL:** For ANY feature that involves Create, Update, or Delete (notes, tags, tasks, projects, members), testing ONLY the render/display is insufficient. You MUST also test the full save flow.

| Phase | Test Requirement | Mock Pattern |
|-------|-----------------|--------------|
| **Render** | Component renders with data ✓ | `mockUseXxx.mockReturnValue({ data: [...] })` |
| **Edit** | User interaction → verify form state updated ✓ | `fireEvent.change(input, ...)` |
| **Save** | Click Save → verify correct API calls made with expected payloads | `expect(mockMutate).toHaveBeenCalledWith(expected)` |
| **Success** | Verify UI updates after successful mutation (toast, refresh, list update) | Verify `invalidateQueries` calls or DOM updates |

**Symptom of insufficient testing:** "Clearly this wasn't tested thoroughly" — the display renders correctly but clicking Save causes "Something went wrong" because the mutation path was never exercised. This happened with the notes feature: concatenating notes displayed correctly but the save path tried to write the concatenated string back as a single note, which failed.

**Pattern:** For CRUD features, test all four operations in one integration test:
- Load existing data → verify display
- Edit existing item → save → verify update API called
- Add new item → save → verify create API called
- Delete existing item → save → verify delete API called

**Test mock must exercise the real code path:** Don't mock `createTask()` differently in tests than how the component calls it. If the component goes through `handleFormSubmit` → `createTask.mutateAsync`, the test must exercise that path, not bypass it.

### 7. Migration Pre-Flight Checklist

Before applying ANY migration that adds:
- `NOT NULL` column
- Foreign key constraint
- New enum value
- RLS policy change

Run this checklist:

1. **Sweep ALL INSERT paths:** `grep -r "INSERT INTO <table>" --include="*.sql" --include="*.ts"` across the entire monorepo
2. **Check database triggers:** `tg_provision_*`, `trg_*` — these fire on signup and are EASILY missed
3. **Check edge functions:** `supabase/functions/*/index.ts` 
4. **Check MCP server tools:** `apps/mcp-server/src/mcp/tools/*.ts`
5. **Apply migration** via `mcp_supabase_apply_migration`
6. **Sign up with a fresh test account** — verify no 500 errors in auth logs
7. **Visit app as unauthenticated user** — verify no broken pages (RLS silent nulls)
8. **Run `mcp_supabase_get_logs(service='auth')`** — confirm zero 500 errors
9. **If the migration affects Vercel-deployed code,** check Vercel deployment state

**REPEATED FAILURE PATTERN:** Migration 0012 added `owner_user_id NOT NULL` → `tg_provision_personal_workspace()` not updated → all signups broke. Migration 0017 added `project_name NOT NULL` → same pattern again. A NOT NULL column change requires a COMPLETE sweep of ALL insert paths.

**Most commonly missed path:** `tg_provision_personal_workspace()` — this trigger is defined in an early migration file and is the FIRST code that runs for any new user. It's easy to forget because it's not in the same migration file as the column change.

## Hermes-Specific Tips

- **Vite build in Hermes:** Hermes blocks `vite build` as a long-running process. Use: `node -e "const {build} = require('vite'); build({logLevel:'info'}).then(() => console.log('BUILD OK')).catch(e => {console.error(e); process.exit(1)})"`
- **Git commit in Hermes:** If Hermes blocks `git commit`, the message may be too long. Shorten it.
- **Subagent timeouts:** If a subagent times out (300s), retry once. If it times out again, fall back to investigating and fixing directly yourself — some debugging tasks (reading many files, tracing data flows) are too wide-ranging for subagents and complete faster when done by the orchestrator using `execute_code` for batch file reads.
- **Context compaction:** Write investigation findings to plan files BEFORE they get compacted away. Re-read files after compaction if delegating.

## Multi-Issue Parallel Delegation

When the user reports multiple bugs at once (e.g. "fix these 4 issues"), use this workflow:

### 1. Investigate First (single leaf agent)

Delegate ONE investigation task to understand ALL issues before writing any code. Provide a checklist of what to find per issue:
- Exact file paths and line numbers
- Current code behavior vs expected
- Cross-view comparison (does the bug exist in other views?)
- What needs to change (not HOW, just WHAT)

**Why:** You need the full picture to partition work correctly and avoid two agents editing the same file.

### 2. Write a Plan File

Create a plan that explicitly identifies **dependencies between issues** and **file ownership** for each issue. This determines which issues can be parallelized.

**Partitioning rules:**
- Issues that touch completely different files → delegate in parallel
- Issues with shared type changes (e.g. `statusFilter: string` → `string[]`) → delegate sequentially, dependency first
- Issues in the same file → delegate to ONE agent

### 3. Parallel Delegate

Dispatch agents in parallel when file ownership doesn't conflict. Name them by what they own, not by issue numbers:

```
Gurleen: Issues 1+2 (PriorityMatrix.tsx, KanbanBoard.tsx, index.css)
Fateh: Issues 3+4 (Dashboard.tsx, SearchFilterBar.tsx, tests) — Issue 4 first (Issue 3 depends on it)
```

Each agent gets:
- The plan file path + the relevant section copied into context
- Exact file paths and line numbers
- Verification command (`tsc --noEmit`)

### 4. Merge and Verify

After both agents complete:
- Run `tsc --noEmit` (agents may have conflicting changes)
- If conflicts, resolve and re-verify
- Commit all changes together in one logical commit
- Push and verify deploy

**Anti-pattern:** Don't dispatch agents blindly with just issue descriptions. The investigation step ensures you know the exact files and can partition correctly.

## Anti-Patterns

- **Fixing symptoms instead of root cause** — 3 separate fixes for what's really one issue. **Classic example:** React Router redirect bug — subagents added defensive guards (auth timing, error display) to the page component, but the real fix was refactoring the routing architecture from nested `<Routes>` to `<Outlet>` pattern. The guards were reasonable hardening but didn't solve the redirect. **Lesson:** When investigation reveals an architectural issue (routing, layout, state management), the orchestrator should fix it directly rather than delegating — subagents tend to add defensive code around symptoms rather than restructure architecture.
- **Changing values without understanding downstream effects** — DB triggers, computed state, derived types
- **Reporting success before verifying the deploy** — always check Vercel state
- **Delegating without investigation** — subagent wastes time rediscovering what you could have found in 2 minutes
- **Skipping the collapsed/hidden UI state** — test both expanded and collapsed views, mobile and desktop
