---
name: supabase-edge-functions
description: Deploy Supabase Edge Functions — code, deploy, secrets, auth patterns, CORS, and common deploy failures. Covers Deno.serve, verify_jwt, API key auth, Resend email, and CLI fallback when MCP deploy is stuck.
version: 2.0.0
author: Hermes Agent
metadata:
  hermes:
    tags: [supabase, edge-functions, deno, deploy, secrets, cors, resend, auth]
    related_skills: [supabase-development, vercel-monorepo]
---

# Supabase Edge Functions

## When to Use

Deploying, updating, or troubleshooting a Supabase Edge Function. Covers the full lifecycle: code → deploy → secrets → web app wiring.

## Prerequisites

- Supabase project ID
- Edge function code in `supabase/functions/<name>/index.ts`
- `deno.json` in the function directory

## Deploy Steps

### 1. Write the Edge Function

```typescript
import "jsr:@supabase/functions-js/edge-runtime.d.ts";

Deno.serve(async (req: Request) => {
  // handler logic
});
```

### 2. Create deno.json

```json
{
  "imports": {
    "@supabase/functions-js/edge-runtime.d.ts": "jsr:@supabase/functions-js/edge-runtime.d.ts"
  }
}
```

### 3. Deploy via Supabase MCP

Use `mcp_supabase_deploy_edge_function`:
- `project_id`: project ID
- `name`: function slug
- `entrypoint_path`: "index.ts"
- `verify_jwt`: `false` (when using custom auth)
- `files`: `[{name, content}]` — include index.ts and deno.json

### 4. Set Environment Variables

**Default env vars (auto-available):**
- `SUPABASE_URL`, `SUPABASE_ANON_KEY`, `SUPABASE_SERVICE_ROLE_KEY`, `SUPABASE_DB_URL`

**WARNING:** Supabase MCP has no tool to set custom env vars. Options:
- **(Preferred)** User sets via Supabase Dashboard → Edge Functions → `<fn>` → Secrets
- Or use Supabase CLI with personal access token

### 5. Auth Patterns

**Bearer token auth (recommended for internal APIs):**
```typescript
Deno.serve(async (req: Request) => {
  const authHeader = req.headers.get("Authorization") ?? "";
  const expectedKey = Deno.env.get("API_KEY") ?? "";

  if (!expectedKey || authHeader !== `Bearer ${expectedKey}`) {
    return new Response(JSON.stringify({ error: "Unauthorized" }), {
      status: 401,
      headers: { "Content-Type": "application/json" },
    });
  }
  // ... handler
});
```

**Web app client:**
```typescript
const response = await fetch(`${supabaseUrl}/functions/v1/<fn-name>`, {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'Authorization': `Bearer ${import.meta.env.VITE_API_KEY}`,
  },
  body: JSON.stringify({ action: 'send' }),
});
```

**JWT auth (user identity):** Set `verify_jwt: true` and the function receives `req.headers.get('Authorization')` with the user's Supabase JWT.

### 6. Handle Deploy Failures

- If `deploy_edge_function` returns "internal error," **retry once** — often transient
- If 2+ retries fail, use **CLI fallback**:

```bash
SUPABASE_ACCESS_TOKEN=<token> npx supabase functions deploy <name> --project-ref <ref> --no-verify-jwt
```

- **Delete-and-recreate** for truly stuck functions:
  ```bash
  SUPABASE_ACCESS_TOKEN=<token> npx supabase functions delete <name> --project-ref <ref>
  SUPABASE_ACCESS_TOKEN=<token> npx supabase functions deploy <name> --project-ref <ref> --no-verify-jwt
  ```
- Previous version stays live during delete→deploy gap
- `--no-verify-jwt` is required when function uses custom API key auth

---

## 🔴 Critical Pitfalls

### CORS — apikey / x-client-info Headers (MOST COMMON)

When calling an edge function from the browser via `supabase.functions.invoke()`, the Supabase JS client sends `apikey` and `x-client-info` headers. If `Access-Control-Allow-Headers` doesn't list them, the CORS preflight succeeds but browser silently blocks the POST.

**Symptom:** OPTIONS logs appear in Supabase, but zero POST logs. Function seems "not called" but no error.

**Fix:** Add to CORS headers:
```typescript
'Access-Control-Allow-Headers': 'authorization, x-client-info, apikey, content-type'
```

### ❌ Do NOT use `SUPABASE_ANON_KEY` from `Deno.env.get()` as comparison

Edge function's anon key ≠ client-side anon key. Use a dedicated API key.

### ❌ `verify_jwt: true` + custom auth = double failure

If using custom API key auth, set `verify_jwt: false` on deploy. Otherwise both auth layers must pass.

### ❌ Nested closing generics `>>` in Deno

Deno's parser treats `>>` as single token. Add space: `Promise<Array<{ id: string }> >`

### ❌ Querying `auth.users` via REST API

Not accessible via REST. Use `createClient(url, serviceRoleKey).auth.admin.listUsers()`.

### ❌ Forgetting `deno.json`

Function fails to find edge-runtime types.

### ❌ Hardcoding secrets

Always use `Deno.env.get()`. `.env` files in `.gitignore` — commit `.env.example`.

### ❌ `tsup`/`dist/` patterns leaked to Deno

Edge functions run Deno, not Node. No `node_modules`, no build step.

---

## Verification

1. Function deploys (status = ACTIVE)
2. Test with correct auth → 200
3. Test with wrong/no auth → 401
4. Test with expected payloads
5. Check Supabase edge function logs for any errors

```bash
# Direct test
curl -X POST https://<project>.supabase.co/functions/v1/<name> \
  -H "Authorization: Bearer <key>" \
  -H "Content-Type: application/json" \
  -d '{"action":"preview"}'
```
