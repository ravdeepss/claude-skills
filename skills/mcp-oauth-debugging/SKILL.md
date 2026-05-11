---
name: mcp-oauth-debugging
description: Debug MCP server OAuth authorization flows — Hono + Cloudflare Workers + KV. Covers 11 common failure modes, CORS, client_secret_basic, token exchange, and discovery endpoints. Framework-agnostic, applicable to any MCP server implementing OAuth.
version: 2.0.0
author: Hermes Agent
metadata:
  hermes:
    tags: [oauth, mcp-server, debugging, cloudflare-worker, hono, kv, cors]
    related_skills: [mcp-server-testing, cloudflare-worker-deploy, native-mcp]
---

# MCP Server OAuth Debugging

## When to Use

Use when debugging OAuth authorization failures for any MCP server (Claude Desktop/Cowork, Cursor, VS Code, etc.). Covers the full lifecycle: authorize → consent → approve → token exchange → MCP connection.

## OAuth Flow Reference

```
1. Client → GET /oauth/authorize?redirect_uri=...&state=...
2. Server → stores request in KV (5-min TTL) → redirects to consent page
3. User → logs in → sees consent screen → clicks "Authorize"
4. Frontend → GET /oauth/request/:id → POST /oauth/request/:id/approve (Bearer token)
5. Server → verifies user membership → creates auth code → returns {redirect_uri, code, state}
6. Frontend → redirects to redirect_uri?code=X&state=Y (use window.location.href, NOT React Router)
7. Client → POST /oauth/token with client_secret_basic auth → exchanges code for access token
8. Client → connects to MCP endpoint with access token
```

## ⚠️ 11 Critical Pitfalls

### 1. RLS + Anon Key = Silent Failure

The approve handler queries a membership table to verify the user belongs to a workspace/org. If this table has Row Level Security (RLS) enabled, the anon key will NOT return results — even if the user IS a member.

**Symptom:** "Approval failed" with no specific error.

**Fix:** Use `SUPABASE_SERVICE_ROLE_KEY` instead of `SUPABASE_ANON_KEY`. The service role key bypasses RLS.

```typescript
// ❌ BROKEN — RLS blocks the query
const supabase = createClient(url, env.SUPABASE_ANON_KEY);

// ✅ CORRECT — bypasses RLS
const supabase = createClient(url, env.SUPABASE_SERVICE_ROLE_KEY);
```

### 2. Env Var Not Set on Worker

Changing code to use a new env var won't work if the secret isn't configured on the worker.

**Verification:**
```bash
npx wrangler secret list --name <worker-name>
```

**Set secret:**
```bash
echo "your-secret" | npx wrangler secret put SECRET_NAME --name <worker-name>
```

### 3. WEB_APP_URL Mismatch

The `WEB_APP_URL` env var must match the actual frontend domain. If pointing to a preview URL but users visit production, redirects and CORS break.

**Check:** `wrangler.toml` AND live worker settings (manual API changes can cause drift).

### 4. Generic Error Messages Hide Real Problems

Always surface actual server responses — never hide errors behind "please try again":

```typescript
// ❌ Generic — useless for debugging
catch {
  setError('Failed to authorize. Please try again.');
}

// ✅ Detailed — tells you exactly what failed
catch (err) {
  const message = err instanceof Error ? err.message : String(err);
  console.error('OAuth approval failed:', message);
  setError(`Authorization Error - ${message}`);
}
```

### 5. Response Format Contract

The approve endpoint must return `{ redirect_uri, code, state }` as separate fields. The frontend constructs the full redirect URL. Never return a pre-built URL string.

### 6. React Router `navigate()` Cannot Cross-Origin Redirect

React Router uses `history.replaceState()` which browsers block for cross-origin URLs.

**Fix:** Use `window.location.href` for cross-origin redirects:

```typescript
// ❌ BROKEN — replaceState can't change origin
navigate(redirectUrl.toString(), { replace: true });

// ✅ CORRECT — full page navigation to different origin
window.location.href = redirectUrl.toString();
```

### 7. `client_secret_basic` Not Implemented

Server advertises `token_endpoint_auth_methods_supported: ['client_secret_basic']`. Clients send `client_id:client_secret` via HTTP Basic auth — NOT in the POST body.

**Symptom:** Token exchange returns 400 `invalid_request: Missing required parameters` because body params are empty.

**Fix:** Parse `Authorization: Basic ...` header as fallback:

```typescript
function parseBasicAuth(c: Context): { client_id: string; client_secret: string } | null {
  const authHeader = c.req.header('Authorization');
  if (!authHeader?.startsWith('Basic ')) return null;
  const decoded = atob(authHeader.slice(6));
  const colonIndex = decoded.indexOf(':');
  if (colonIndex === -1) return null;
  return {
    client_id: decodeURIComponent(decoded.slice(0, colonIndex)),
    client_secret: decodeURIComponent(decoded.slice(colonIndex + 1)),
  };
}

// In handler:
let client_id = body.client_id;
let client_secret = body.client_secret;
if (!client_id || !client_secret) {
  const basic = parseBasicAuth(c);
  if (basic) {
    client_id = client_id || basic.client_id;
    client_secret = client_secret || basic.client_secret;
  }
}
```

### 8. CORS Blocks Client Token Request

The allowed origins must include the client's domain (e.g., `https://claude.ai` for Claude, `vscode://` for VS Code).

```typescript
app.use('/oauth/*', cors({
  origin: (origin) => {
    const allowed = [env.WEB_APP_URL, '<client-domain>'];
    return allowed.includes(origin) ? origin : '';
  },
  allowMethods: ['GET', 'POST', 'OPTIONS'],
  allowHeaders: ['Content-Type', 'Authorization'],
  credentials: true,
}));
```

### 9. KV Request Expiry

Auth requests stored in KV expire (typically 5 min). If user takes too long to log in, the request is gone → 404.

### 10. McpEndpointNotFound — Root URL Returns 401

Clients use `GET /` to discover the MCP server AFTER OAuth completes. If `/` returns 401, they show "no MCP server found."

**Root cause:** Auth middleware registered before root handler catches all requests.

**Fix:** Register `GET /` BEFORE auth middleware:

```typescript
// BEFORE app.use('*', authMiddleware)
app.get('/', (c) => c.json({
  name: '<server-name>',
  version: '0.1.0',
  mcpEndpoint: `${new URL(c.req.url).origin}/mcp`,
  oauthMetadata: `${new URL(c.req.url).origin}/.well-known/oauth-authorization-server`,
}));

app.get('/.well-known/mcp.json', (c) => c.json({
  endpoint: `${new URL(c.req.url).origin}/mcp`,
  protocolVersion: '2025-11-25',
  capabilities: { tools: { listChanged: true } },
}));
```

### 11. Streamable HTTP Requires GET + POST on `/mcp`

MCP spec requires both POST (JSON-RPC) and GET (SSE stream). Register both:

```typescript
app.post('/mcp', async (c) => handleMcpRequest(c.req.raw, ctx));
app.get('/mcp', async (c) => handleMcpRequest(c.req.raw, ctx));
```

## Debugging Checklist

1. Check worker logs: `npx wrangler tail --name <worker-name>`
2. Verify all env vars are set: `npx wrangler secret list --name <worker-name>`
3. Check RLS on membership tables: `SELECT relname, relrowsecurity FROM pg_class WHERE relname = '<table>';`
4. Verify consent page loads correct server URL
5. Test approve endpoint directly with curl
6. Check browser console for actual error messages
7. Verify CORS: test OPTIONS preflight from client origin
8. Verify `client_secret_basic` support in token handler
9. **Deploy the worker:** Git push does NOT deploy to Cloudflare. Always run `npx wrangler deploy`.

## Deployment

- Worker deploys via `npx wrangler deploy`
- Frontend deploys via platform-specific CD (Vercel, Netlify, etc.)
- All public/discovery routes must be registered BEFORE auth middleware in Hono

## Verification

After deploying:
```bash
# Verify root returns metadata (not 401)
curl https://<worker-url>/

# Verify MCP server card
curl https://<worker-url>/.well-known/mcp.json

# Verify CORS from client origin
curl -X OPTIONS https://<worker-url>/oauth/token \
  -H "Origin: <client-origin>" \
  -H "Access-Control-Request-Method: POST" \
  -v 2>&1 | grep -i 'access-control'
```
