---
name: mcp-server-testing
description: Test patterns and anti-patterns for MCP servers using vitest + mocked Supabase/KV. Use when writing tests, fixing subagent-produced tests, or dispatching test-writing tasks to subagents.
version: 2.0.0
author: Hermes Agent
metadata:
  hermes:
    tags: [testing, mcp-server, vitest, mocks, subagent-pitfalls, supabase, kv]
    related_skills: [subagent-driven-development, mcp-oauth-debugging]
---

# MCP Server Testing Patterns

## When to Use

Reference this skill whenever:
- Writing tests for an MCP server (any framework: Hono, Express, raw HTTP)
- Fixing tests produced by subagents (they consistently produce the same 3 anti-patterns)
- Dispatching test-writing tasks to subagents — include the anti-pattern warnings

## ⛔ Anti-Pattern #1: `clearTools` + `vi.resetModules` (MOST COMMON)

### What subagents do (BROKEN):

```typescript
describe('some_tool', () => {
  beforeEach(() => {
    clearTools();
    vi.resetModules();
  });

  it('tool is registered', async () => {
    await import('../src/mcp/tools/some-tool'); // re-register
    const tool = getToolByName('some_tool');
    expect(tool).toBeDefined();
  });
});
```

### Why it breaks:

1. `vi.resetModules()` invalidates vitest's module cache
2. Static `import '../src/mcp/tools/some-tool'` at file top becomes a no-op
3. `await import(...)` inside tests works for first test but `clearTools()` wipes for subsequent tests
4. Result: `Cannot read properties of undefined (reading 'handler')` on tests 2+

### ✅ Correct pattern:

```typescript
import { describe, it, expect, vi } from 'vitest';
import { getToolByName } from '../src/mcp/tools/registry';
import '../src/mcp/tools/some-tool'; // side-effect registration — ONE TIME

describe('some_tool', () => {
  it('tool is registered', () => {
    const tool = getToolByName('some_tool');
    expect(tool).toBeDefined();
  });

  it('handler works', async () => {
    const tool = getToolByName('some_tool')!;
    const ctx = createMockContext();
    const result = await tool.handler({ ... }, ctx);
    expect(result).toHaveProperty('id');
  });
});
```

**Key rules:**
- NO `clearTools()` in beforeEach
- NO `vi.resetModules()` anywhere
- NO `await import(...)` inside individual tests
- Static import at file top — vitest runs each test file in isolation

---

## ⛔ Anti-Pattern #2: Object Literal Self-Reference (TDZ Error)

### What subagents do (BROKEN):

```typescript
const chain = {
  eq: vi.fn().mockReturnValue(chain),  // ❌ ReferenceError: Cannot access 'chain' before initialization
  select: vi.fn().mockReturnValue(chain),
  single: vi.fn().mockResolvedValue({ data, error }),
};
```

### ✅ Correct pattern:

```typescript
const chain: Record<string, unknown> = {};
chain.eq = vi.fn().mockReturnValue(chain);
chain.select = vi.fn().mockReturnValue(chain);
chain.single = vi.fn().mockResolvedValue({ data, error });
chain.maybeSingle = vi.fn().mockResolvedValue({ data, error });
chain.insert = vi.fn().mockReturnValue(chain);
chain.limit = vi.fn().mockReturnValue(chain);
```

Build incrementally to avoid Temporal Dead Zone. Use `Record<string, unknown>` type.

---

## ⛔ Anti-Pattern #3: Table/Collection Name Mismatch

Subagents use different names in tests vs implementation:
- Test uses `pomodoro_timers` → implementation uses `pomodoro_sessions`
- Test uses `usage_stats` → implementation uses `usage_counters`

**Fix:** After subagent produces tests, grep the implementation for actual names:

```bash
grep -rn "from('" src/mcp/tools/some-tool.ts
# Then update all test mocks to match
```

---

## ✅ Standard Mock Supabase Chain

```typescript
function createMockChain(resolveData: unknown, resolveError: unknown = null) {
  const chain: Record<string, unknown> = {};
  chain.eq = vi.fn().mockReturnValue(chain);
  chain.is = vi.fn().mockReturnValue(chain);
  chain.select = vi.fn().mockReturnValue(chain);
  chain.insert = vi.fn().mockReturnValue(chain);
  chain.limit = vi.fn().mockReturnValue(chain);
  chain.single = vi.fn().mockResolvedValue({ data: resolveData, error: resolveError });
  chain.maybeSingle = vi.fn().mockResolvedValue({ data: resolveData, error: resolveError });
  return chain;
}
```

### Mocking multiple `from()` calls on the same table:

```typescript
let fromCallIndex = 0;
const fromFn = vi.fn((_table: string) => {
  if (_table === 'tasks') {
    return createMockChain({ id: 'task-1', title: 'Test' });
  }
  if (_table === 'sessions') {
    const idx = fromCallIndex++;
    if (idx === 0) return createMockChain(null); // lookup: none exists
    return createMockChain({ id: 'sess-1', status: 'active' }); // insert
  }
  return createMockChain(null);
});
```

### Mocking KV:

```typescript
const mockKv = {
  get: vi.fn().mockImplementation((key: string) => {
    if (key.startsWith('plan:')) return Promise.resolve('pro');
    if (key.startsWith('burst:')) return Promise.resolve('3');
    return Promise.resolve(null);
  }),
  put: vi.fn().mockResolvedValue(undefined),
  delete: vi.fn().mockResolvedValue(undefined),
} as unknown as KVNamespace;
```

### Mocking fetch for Supabase network calls:

```typescript
const originalFetch = globalThis.fetch;
beforeEach(() => {
  vi.spyOn(globalThis, 'fetch').mockImplementation((url) => {
    if (String(url).includes('supabase')) {
      return Promise.reject(new TypeError('fetch failed (mocked)'));
    }
    return originalFetch(url);
  });
});
afterEach(() => vi.restoreAllMocks());
```

---

## ✅ RequestContext Mock

```typescript
function createMockContext(): RequestContext {
  return {
    userId: 'user-123',
    workspaceId: 'ws-1',
    supabase: mockSupabase as unknown as SupabaseClient<Database>,
    clientName: 'test-client',
    env: undefined as unknown as RequestContext['env'],
  };
}
```

---

## ✅ Integration Test Patterns

- Use `Hono<any>` type for test apps (avoids strict env type mismatch)
- Mock `globalThis.fetch` for REST API calls
- Use `vi.spyOn(sideEffectModule, 'logSomething').mockResolvedValue(undefined)` to suppress side effects
- In-memory KV via `Map<string, string>` with getter/putter functions
- Tests must complete in < 5 seconds each (no real network calls)

---

## Post-Subagent Cleanup Checklist

After EVERY subagent that writes tests:

```bash
# 1. Search for anti-patterns
grep -rn 'clearTools\|resetModules\|await import.*tools/' tests/

# 2. Search for TDZ risk
grep -rn 'mockReturnValue(chain)' tests/

# 3. Verify table names match
grep "from('" src/mcp/tools/new-tool.ts
grep "from('" tests/new-tool.test.ts

# 4. Run typecheck + tests
pnpm typecheck && pnpm test
```

---

## Summary: Tell Subagents

```
DO NOT:
- Use clearTools() or vi.resetModules() in beforeEach
- Call await import(...) inside individual tests (static imports are sufficient)
- Use object literal self-reference for mock chains (TDZ error)
- Guess table names — verify them from implementation

DO:
- Use static imports for tool registration
- Build mock chains incrementally: const chain = {}; chain.eq = ...
- Include env: undefined as unknown as RequestContext['env'] in mock context
- Use createMockChain() helper
```
