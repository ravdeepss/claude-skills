---
name: pre-deploy-qa-gate
description: Mandatory QA gate to run before reporting any bug as fixed. Deploy verification, runtime smoke test, migration pre-flight, CRUD save-path testing, and edge-case test table. Prevents bugs from reaching Rav.
version: 1.0.0
metadata:
  hermes:
    tags: [qa, deploy, verification, smoke-test, migration, crud, testing]
    related_skills: [bug-fix-orchestration, systematic-debugging, ike-saas-development, test-driven-development]
---

# Pre-Deploy QA Gate

## When to Use

Run this gate **before reporting any bug as "fixed"** in the IKE SaaS project or any project with deployable targets. This fills the gap between "tests pass locally" and "works in production" — which is where every bug in the April-May 2026 post-mortem slipped through.

## The 7 Gates

Complete ALL 7 gates before telling the user a bug is fixed. Skip none.

---

### Gate 1: Deploy Everything That Changed

**Rule:** If code was changed in a deployable target, it must be deployed AND verified.

| Target | Deploy Command | Verification |
|--------|---------------|-------------|
| `apps/web/` | `git push` (Vercel auto-deploys) | `mcp_vercel_getDeployments(projectId="prj_v37jQJsTuT4g9QXt1bzGZZaluial", limit=1)` → confirm `state: "READY"` |
| `apps/mcp-server/` | `cd apps/mcp-server && npx wrangler deploy --name ike-mcp` | Call `mcp_ike_get_task(task_id="<known task id>")` → verify response shape |
| `supabase/functions/` | Deploy via MCP or CLI | Call the function URL, verify 200 response |
| Database migrations | `mcp_supabase_apply_migration` | `mcp_supabase_execute_sql` to verify column/constraint exists |

**PITFALL — MCP server not auto-deployed:** Git push does NOT deploy to Cloudflare Workers. This burned us on May 6 — `get-task.ts` was updated weeks earlier, 170 tests passing, but the live worker returned stale 9-field data because nobody ran `wrangler deploy`. **Every MCP server fix MUST end with a deploy + verify against the live endpoint.**

**PITFALL — Vercel build pass ≠ runtime works:** `tsc` + `vite build` can succeed while the app crashes on load due to missing context providers. Gate 2 (smoke test) catches this.

---

### Gate 2: Runtime Smoke Test

**Rule:** Run `Smoke.test.tsx` (or equivalent) which renders the full app tree. This catches:

- Missing context providers (`useUndoToasts must be used inside <UndoToastProvider>`)
- Broken prop chains (component rendered but no-op callbacks)
- Import errors that don't surface in per-file typecheck

**Command:**
```bash
cd apps/web && /opt/data/home/.local/node_modules/.bin/pnpm vitest run src/Smoke.test.tsx
```

**If the smoke test doesn't exist yet:** Create it. See [`references/smoke-test-pattern.md`](references/smoke-test-pattern.md) for the complete pattern: `vi.hoisted()` for Supabase auth mocks, full App tree rendering, and anti-patterns to avoid.

**What we learned:** `tsc -b` + `vite build` both passed when `UndoToastProvider` was missing from `App.tsx`. Only runtime could catch it. The smoke test is cheap (runs in < 2 seconds) and catches the class of bugs that type systems can't.

---

### Gate 3: CRUD Save-Path Test Mandate

**Rule:** For ANY feature involving Create, Update, or Delete, the test suite MUST include:

| Test Case | What It Verifies |
|-----------|-----------------|
| **Render** | Component renders with pre-loaded data |
| **Edit** | User interaction updates form/UI state |
| **Save** | Clicking Save calls the correct API with expected payloads |
| **Success** | UI reflects post-save state (toast, list refresh, etc.) |

**Anti-pattern:** Tests that only verify rendering but never click Save. This burned us on the notes feature — concatenated notes displayed correctly, but Save crashed because the concatenated string (with `---` separators) was written back as one note.

**Detection:** If a test uses `screen.getByText` / `screen.getByTestId` to verify display but never calls `fireEvent.click(submitButton)` or `fireEvent.submit(form)`, it's incomplete.

**Mock verification:** After the save action, assert the mock mutation was called:
```typescript
expect(mockCreateTask).toHaveBeenCalledWith({
  title: 'New Task',
  notes: ['note 1', 'note 2'],  // verify payload shape
});
```

---

### Gate 4: Migration Pre-Flight Sweep

**Rule:** Before applying ANY migration that adds `NOT NULL`, a FK constraint, or an enum change:

1. **Sweep ALL INSERT into the affected table:**
   ```bash
   grep -rn "INSERT INTO <table>" --include="*.sql" --include="*.ts" apps/ supabase/
   ```
2. **Check database triggers** — `tg_provision_*` triggers are in early migration files and EASILY forgotten:
   ```sql
   SELECT proname, prosrc FROM pg_proc WHERE proname LIKE 'tg_%' OR proname LIKE 'trg_%';
   ```
3. **Check edge functions** — `supabase/functions/*/index.ts`
4. **Check MCP server tools** — `apps/mcp-server/src/mcp/tools/*.ts`
5. **After applying:** Sign up with a fresh test account → check auth logs for 500s
6. **Check PostgREST RLS:** Visit affected pages as unauthenticated user — RLS blocks embedded joins silently (returns `null`, not 403)

**REPEATED FAILURE:** Migration 0012 (`owner_user_id NOT NULL`) broke all new signups because `tg_provision_personal_workspace()` was never updated. Migration 0017 (`project_name NOT NULL`) broke re-invites with the same pattern. Both times the agent applied React-layer fixes before finding the DB root cause.

---

### Gate 5: Edge-Case Test Table

**Rule:** For every feature being fixed/built, the test plan must include:

| Feature | Happy Path | Empty State | Error State | Key Edge Case |
|---------|-----------|-------------|-------------|--------------|
| *example* | Normal creation ✓ | No data returned | Network error | Soft-deleted rows breaking sequence |

**Edge cases that have burned us:**
- **Soft-deleted rows with gaps** → `COUNT(*)` returns wrong task number → duplicate key
- **Hung async operations** → safety timeout cleared by same code it protects → permanent spinner
- **Unauthenticated users** → RLS silently blocks PostgREST joins → "Unknown Project"
- **Rapid auth events** → race condition between `getSession()` and `onAuthStateChange()`
- **`createTask()` silently drops fields** → form mapping gap → recurrence/reminder/assignee lost on create

**How to use:** During the bug-fix plan phase, fill out one row per affected feature. If you can't think of edge cases, ask: "What happens if data is missing? What happens if it's duplicated? What happens if the user isn't authenticated?"

---

### Gate 6: Debug Layer Order

**Rule:** When investigating production bugs, check layers in THIS order. Never start at the React UI:

1. **Supabase auth logs** (`mcp_supabase_get_logs(service='auth')`) — 500s on callback are DB trigger failures
2. **Deployed version** — is the fix actually live? Call the endpoint, check deployment state
3. **Database state** — run diagnostic SQL, check nulls, check constraints, verify RLS
4. **Network responses** — check actual HTTP status codes and bodies
5. **React layer LAST** — only after confirming all layers above are clean

**Why this order:** We applied 5 React fixes (CORS, redirects, token preservation, full-page reload, session storage) before finding a DB trigger failure. The trigger returned 500 → 302 redirect → user landed logged out. Auth logs showed the error immediately. We just never checked them first.

**Symptom that means "check lower layers":** "App appears to work but user ends up in wrong state." The 302 redirect masking a 500. The PostgREST join returning null instead of throwing. The component rendering but no-op callback doing nothing.

---

### Gate 7: Provider Wiring Audit

**Rule:** When adding ANY new React context provider or a component that uses a context hook:

1. **Verify the provider is wrapped in `App.tsx`** — file can exist with correct exports but never be imported
2. **Make the hook defensive:** Return no-op fallback instead of throwing when provider is missing:
   ```typescript
   export function useMyContext() {
     const ctx = useContext(MyContext);
     if (!ctx) {
       if (import.meta.env.DEV) console.warn('useMyContext: provider missing');
       return { /* safe defaults */ };
     }
     return ctx;
   }
   ```
3. **Add shared mock to `test-helpers/`** — prevents 53 Dashboard test files from breaking
4. **Run the smoke test** after wiring — catches the provider gap immediately

**What we learned:** `UndoToastProvider` was correctly defined and exported. `InviteDropdown` correctly called `useUndoToasts()`. But `App.tsx` never imported or rendered `<UndoToastProvider>`. `tsc` + `vite build` both passed. App crashed on every page load. 53 Dashboard tests broke. Fixed in one line by making the hook defensive, but the real fix should have been caught before Rav saw it.

---

## Pre-Fix vs Post-Fix Gates

| When | Gates to Run |
|------|-------------|
| **Before writing code** | Gate 5 (edge-case table), Gate 6 (debug layer order — if investigating) |
| **After writing code, before pushing** | Gate 3 (CRUD save-path test), Gate 7 (provider wiring) |
| **After deploying** | Gate 1 (deploy verification), Gate 2 (smoke test), Gate 4 (migration sweep — if DDL changed) |
| **Before telling user "fixed"** | ALL GATES must pass |

---

## Red Flags — Don't Report "Fixed" If:

- Vercel deployment state is not `READY`
- Cloudflare Worker was not redeployed after `apps/mcp-server/src/` changes
- Tests only verify rendering but never exercise save/submit path
- Migration was applied but no fresh test account signup was performed
- Auth logs show any 500 error after migration
- A new provider/context hook was added but `App.tsx` wasn't checked
- The smoke test fails or doesn't exist

---

## Quick Reference

```
GATE 1: Deploy + verify every changed target
GATE 2: Run smoke test (full app tree render)
GATE 3: CRUD tests include save path (not just render)
GATE 4: Migration sweep (all INSERT paths + fresh signup)
GATE 5: Edge-case table filled per feature
GATE 6: Debug from bottom up (auth logs → deploy → DB → network → React)
GATE 7: Provider wiring audit (App.tsx import + defensive hook + shared mock)
```
