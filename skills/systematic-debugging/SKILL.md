---
name: systematic-debugging
description: Use when encountering any bug, test failure, or unexpected behavior. 4-phase root cause investigation — NO fixes without understanding the problem first.
version: 1.1.0
author: Hermes Agent (adapted from obra/superpowers)
license: MIT
metadata:
  hermes:
    tags: [debugging, troubleshooting, problem-solving, root-cause, investigation]
    related_skills: [test-driven-development, writing-plans, subagent-driven-development]
---

# Systematic Debugging

## Overview

Random fixes waste time and create new bugs. Quick patches mask underlying issues.

**Core principle:** ALWAYS find root cause before attempting fixes. Symptom fixes are failure.

**Violating the letter of this process is violating the spirit of debugging.**

## The Iron Law

```
NO FIXES WITHOUT ROOT CAUSE INVESTIGATION FIRST
```

If you haven't completed Phase 1, you cannot propose fixes.

## When to Use

Use for ANY technical issue:
- Test failures
- Bugs in production
- Unexpected behavior
- Performance problems
- Build failures
- Integration issues

**Use this ESPECIALLY when:**
- Under time pressure (emergencies make guessing tempting)
- "Just one quick fix" seems obvious
- You've already tried multiple fixes
- Previous fix didn't work
- You don't fully understand the issue

**Don't skip when:**
- Issue seems simple (simple bugs have root causes too)
- You're in a hurry (rushing guarantees rework)
- Someone wants it fixed NOW (systematic is faster than thrashing)

## The Four Phases

You MUST complete each phase before proceeding to the next.

### Layer Order Rule — Debug from Bottom Up

**CRITICAL:** When debugging production issues in a multi-layer system (DB → API → Worker → React UI), check layers in THIS order. Do NOT start at the UI and work down:

1. **Supabase auth logs** (`mcp_supabase_get_logs(service='auth')`) — catches DB trigger failures, 500s on signup/callback
2. **Deployed version** — is the fix actually deployed? (call live MCP endpoint, check Vercel deployment state, check edge function version)
3. **Database state** — run diagnostic SQL, check for nulls in NOT NULL columns, check RLS policies, verify migrations were applied
4. **Network tab / API responses** — check actual HTTP responses (not what code expects)
5. **React layer LAST** — only after confirming all layers above are clean

**Why this order:** We burned hours applying 5 React-layer fixes (CORS, redirects, token preservation, etc.) for a bug whose root cause was a DB trigger failure (migration 0012 NOT NULL). The trigger was returning 500 on auth callback, but the 302 redirect masked it as "works in browser but user lands logged out." Auth logs showed the error immediately — we just never checked them first.

**Symptom pattern that indicates a lower-layer problem:** "App appears to work but user ends up in wrong state" (logged out, missing data, "Unknown Project"). The redirect/fallback behavior in the UI masks the real error. Always check server-side logs before debugging client code.

---

## Phase 1: Root Cause Investigation

**BEFORE attempting ANY fix:**

### 1. Read Error Messages Carefully

- Don't skip past errors or warnings
- They often contain the exact solution
- Read stack traces completely
- Note line numbers, file paths, error codes

**Action:** Use `read_file` on the relevant source files. Use `search_files` to find the error string in the codebase.

### 2. Reproduce Consistently

- Can you trigger it reliably?
- What are the exact steps?
- Does it happen every time?
- If not reproducible → gather more data, don't guess

**Action:** Use the `terminal` tool to run the failing test or trigger the bug:

```bash
# Run specific failing test
pytest tests/test_module.py::test_name -v

# Run with verbose output
pytest tests/test_module.py -v --tb=long
```

### 3. Check Recent Changes

- What changed that could cause this?
- Git diff, recent commits
- New dependencies, config changes

**Action:**

```bash
# Recent commits
git log --oneline -10

# Uncommitted changes
git diff

# Changes in specific file
git log -p --follow src/problematic_file.py | head -100
```

### 4. Gather Evidence in Multi-Component Systems

**WHEN system has multiple components (API → service → database, CI → build → deploy):**

**BEFORE proposing fixes, add diagnostic instrumentation:**

For EACH component boundary:
- Log what data enters the component
- Log what data exits the component
- Verify environment/config propagation
- Check state at each layer

Run once to gather evidence showing WHERE it breaks.
THEN analyze evidence to identify the failing component.
THEN investigate that specific component.

### 5. Trace Data Flow

**WHEN error is deep in the call stack:**

- Where does the bad value originate?
- What called this function with the bad value?
- Keep tracing upstream until you find the source
- Fix at the source, not at the symptom

**Action:** Use `search_files` to trace references:

```python
# Find where the function is called
search_files("function_name(", path="src/", file_glob="*.py")

# Find where the variable is set
search_files("variable_name\\s*=", path="src/", file_glob="*.py")
```

### 6. Check for Missing CSS Classes (Component UI Breakdown)

When components render with "buttons that look like labels", "inputs with no borders", or generally unstyled text — especially after porting from another codebase — the root cause is often **CSS classes referenced in JSX but not defined in the stylesheet**.

**Diagnosis:**
1. `grep -rn "className.*btn-primary\|className.*form-input\|className.*btn-icon"` on the component file to find what CSS classes it uses
2. `grep -rn "\.btn-primary\|\.form-input" apps/web/src/index.css` — if no matches, the classes are missing
3. Check if there's a reference/legacy CSS file: `find . -name "*.css" | xargs grep -l "\.btn-primary"` — this finds where the styles were defined before
4. Port the missing class definitions into the current stylesheet, **mapping CSS custom properties** to the current theme system (old project may use `var(--accent-primary)` while current uses `var(--color-brand-accent-primary)`)

**Common in:** Monorepos, codebase migrations, multi-agent codebases where one agent writes components and another writes styles.

### 7. Check for Unwired Components (Multi-Agent Codebases)

In codebases built by multiple agents or contributors, a component file may exist with proper types/props but the parent component never imports or renders it (e.g., a modal toggled via state but never rendered in JSX). This is especially common when one agent created the component and another built the page.

**Diagnosis:**
1. `search_files` for the component name across the codebase
2. Check if it's imported by the parent that toggles it
3. If not imported → wire it up (import + render with appropriate props)
4. Check mutation/data hook signatures — the parent may need to pass context (e.g., `workspaceId`) that the component's callback doesn't include
5. Watch for `exactOptionalPropertyTypes: true` in tsconfig — can't pass `{ key: undefined }` where `key?: string` is declared. Use `...(value ? { key: value } : {})` spread pattern instead.

### Phase 1 Completion Checklist

- [ ] Error messages fully read and understood
- [ ] Issue reproduced consistently
- [ ] Recent changes identified and reviewed
- [ ] Evidence gathered (logs, state, data flow)
- [ ] Problem isolated to specific component/code
- [ ] Root cause hypothesis formed

**STOP:** Do not proceed to Phase 2 until you understand WHY it's happening.

---

## Phase 2: Pattern Analysis

**Find the pattern before fixing:**

### 1. Find Working Examples

- Locate similar working code in the same codebase
- What works that's similar to what's broken?

**Action:** Use `search_files` to find comparable patterns:

```python
search_files("similar_pattern", path="src/", file_glob="*.py")
```

### 2. Compare Against References

- If implementing a pattern, read the reference implementation COMPLETELY
- Don't skim — read every line
- Understand the pattern fully before applying

### 3. Identify Differences

- What's different between working and broken?
- List every difference, however small
- Don't assume "that can't matter"

### 4. Understand Dependencies

- What other components does this need?
- What settings, config, environment?
- What assumptions does it make?

---

## Phase 3: Hypothesis and Testing

**Scientific method:**

### 1. Form a Single Hypothesis

- State clearly: "I think X is the root cause because Y"
- Write it down
- Be specific, not vague

### 2. Test Minimally

- Make the SMALLEST possible change to test the hypothesis
- One variable at a time
- Don't fix multiple things at once

### 3. Verify Before Continuing

- Did it work? → Phase 4
- Didn't work? → Form NEW hypothesis
- DON'T add more fixes on top

### 4. When You Don't Know

- Say "I don't understand X"
- Don't pretend to know
- Ask the user for help
- Research more

---

## Phase 4: Implementation

**Fix the root cause, not the symptom:**

### 1. Create Failing Test Case

- Simplest possible reproduction
- Automated test if possible
- MUST have before fixing
- Use the `test-driven-development` skill

### 2. Implement Single Fix

- Address the root cause identified
- ONE change at a time
- No "while I'm here" improvements
- No bundled refactoring

### 3. Verify Fix

```bash
# Run the specific regression test
pytest tests/test_module.py::test_regression -v

# Run full suite — no regressions
pytest tests/ -q
```

### 4. If Fix Doesn't Work — The Rule of Three

- **STOP.**
- Count: How many fixes have you tried?
- If < 3: Return to Phase 1, re-analyze with new information
- **If ≥ 3: STOP and question the architecture (step 5 below)**
- DON'T attempt Fix #4 without architectural discussion

### 5. If 3+ Fixes Failed: Question Architecture

**Pattern indicating an architectural problem:**
- Each fix reveals new shared state/coupling in a different place
- Fixes require "massive refactoring" to implement
- Each fix creates new symptoms elsewhere

**STOP and question fundamentals:**
- Is this pattern fundamentally sound?
- Are we "sticking with it through sheer inertia"?
- Should we refactor the architecture vs. continue fixing symptoms?

**Discuss with the user before attempting more fixes.**

This is NOT a failed hypothesis — this is a wrong architecture.

---

## Red Flags — STOP and Follow Process

If you catch yourself thinking:
- "Quick fix for now, investigate later"
- "Just try changing X and see if it works"
- "Add multiple changes, run tests"
- "Skip the test, I'll manually verify"
- "It's probably X, let me fix that"
- "I don't fully understand but this might work"
- "Pattern says X but I'll adapt it differently"
- "Here are the main problems: [lists fixes without investigation]"
- Proposing solutions before tracing data flow
- **"One more fix attempt" (when already tried 2+)**
- **Each fix reveals a new problem in a different place**

**ALL of these mean: STOP. Return to Phase 1.**

**If 3+ fixes failed:** Question the architecture (Phase 4 step 5).

## Common Rationalizations

| Excuse | Reality |
|--------|---------|
| "Issue is simple, don't need process" | Simple issues have root causes too. Process is fast for simple bugs. |
| "Emergency, no time for process" | Systematic debugging is FASTER than guess-and-check thrashing. |
| "Just try this first, then investigate" | First fix sets the pattern. Do it right from the start. |
| "I'll write test after confirming fix works" | Untested fixes don't stick. Test first proves it. |
| "Multiple fixes at once saves time" | Can't isolate what worked. Causes new bugs. |
| "Reference too long, I'll adapt the pattern" | Partial understanding guarantees bugs. Read it completely. |
| "I see the problem, let me fix it" | Seeing symptoms ≠ understanding root cause. |
| "One more fix attempt" (after 2+ failures) | 3+ failures = architectural problem. Question the pattern, don't fix again. |

## Quick Reference

| Phase | Key Activities | Success Criteria |
|-------|---------------|------------------|
| **1. Root Cause** | Read errors, reproduce, check changes, gather evidence, trace data flow | Understand WHAT and WHY |
| **2. Pattern** | Find working examples, compare, identify differences | Know what's different |
| **3. Hypothesis** | Form theory, test minimally, one variable at a time | Confirmed or new hypothesis |
| **4. Implementation** | Create regression test, fix root cause, verify | Bug resolved, all tests pass |

## Hermes Agent Integration

### Investigation Tools

Use these Hermes tools during Phase 1:

- **`search_files`** — Find error strings, trace function calls, locate patterns
- **`read_file`** — Read source code with line numbers for precise analysis
- **`terminal`** — Run tests, check git history, reproduce bugs
- **`web_search`/`web_extract`** — Research error messages, library docs

### With delegate_task

For complex multi-component debugging, dispatch investigation subagents:

```python
delegate_task(
    goal="Investigate why [specific test/behavior] fails",
    context="""
    Follow systematic-debugging skill:
    1. Read the error message carefully
    2. Reproduce the issue
    3. Trace the data flow to find root cause
    4. Report findings — do NOT fix yet

    Error: [paste full error]
    File: [path to failing code]
    Test command: [exact command]
    """,
    toolsets=['terminal', 'file']
)
```

### With test-driven-development

When fixing bugs:
1. Write a test that reproduces the bug (RED)
2. Debug systematically to find root cause
3. Fix the root cause (GREEN)
4. The test proves the fix and prevents regression

## Known Anti-Patterns

### The Self-Defeating Safety Net

**Pattern:** A safety timeout meant to protect against a hanging operation is cleared by the same code path it's supposed to guard, *before* that code path completes.

```ts
// ❌ BUG: safety timer cleared before the code it protects finishes
const safetyTimer = setTimeout(() => forceLoadingOff(), 8000);
onAuthStateChange(async (event, session) => {
  clearTimeout(safetyTimer);  // ← kills the safety net immediately
  await handleSession(session); // ← if this hangs, nobody saves you
});
```

**How it manifests:** Everything works in the happy path. The timeout never fires during testing because the async operation completes quickly. But in production (slow network, cold start, rate limiting), the async operation hangs — and the safety net was already dismantled.

**Three-layer fix pattern:**

1. **Safety timer is independent** — never cleared by the code it protects. Only cleared on component unmount.
2. **The async operation has its own timeout** — `AbortController` with a deadline so it can't hang forever.
3. **State transitions use `try/finally`** — the "done loading" flag is set in a `finally` block, guaranteeing it runs regardless of success/error/timeout.

```ts
// ✅ FIX: three independent layers
const safetyTimer = setTimeout(() => setIsLoading(false), 8000); // never cleared by auth

onAuthStateChange(async (event, session) => {
  // NOTE: do NOT clearTimeout(safetyTimer) here!
  await handleSession(session); // has its own AbortController timeout
  // handleSession uses try/finally to guarantee setIsLoading(false)
});
```

**Detection clue:** "Works on first load but hangs on refresh" — the safety timer fires on first load (no session yet, resolves quickly) but gets defeated on refresh when a session exists and the subsequent query is slow.

### The Invisible Overlay Stealing Clicks

**Pattern:** A component designed for standalone use (popover, dropdown, modal) has a full-viewport backdrop/overlay (`position: fixed; inset: 0; z-index: 99`) that intercepts all clicks. When that component is later embedded inside another container (drawer, panel, tab content), the backdrop sits on top of the interactive elements, making them unclickable.

**How it manifests:** User reports "clicking X does nothing" — buttons, checkboxes, links visually render and respond to hover but clicks never fire. No console errors. The elements appear fully functional.

**Diagnosis:**
1. Open browser DevTools → inspect the unclickable element
2. Click the "Select element" tool and click on the area — DevTools selects the backdrop div, NOT the target element
3. Check z-index: the backdrop has higher z-index than the content
4. `search_files` for `backdrop|overlay` in the component file — find the offending `<div className="...-backdrop">`

**Fix:** Remove the full-viewport backdrop when the component is embedded in a container that provides its own click-outside handling. If the backdrop is needed for standalone mode, gate it on a prop like `standalone` or `useBackdrop`.

```tsx
// ❌ BUG: backdrop covers everything when embedded
<div className="tt-backdrop" style={{ position: 'fixed', inset: 0, zIndex: 99 }}>
  <div className="tt-content">
    <button onClick={handleClick}>This click never fires</button>
  </div>
</div>

// ✅ FIX: remove backdrop when embedded in a parent container
// Parent (drawer/modal) already handles click-outside and closing
<div className="tt-content">
  <button onClick={handleClick}>Clicks work now</button>
</div>
```

**Detection clue:** "I can see the element and hover works, but clicking does nothing" — especially after a component was recently moved from standalone to embedded use.

### Component Unmount Kills Minimized State

**Pattern:** A component with two modes (fullscreen + minimized) is conditionally rendered based on a parent's "open" state. When the user minimizes, the parent sets `open = false`, which **unmounts** the entire component — destroying the minimized pill/widget along with it.

**How it manifests:** User minimizes a timer/notification/widget → it disappears completely instead of showing the minimized version. The minimized view exists in the code but is never visible.

**Diagnosis:**
1. Find the render condition: `{showComponent && <MyComponent onMinimize={...} />}`
2. `onMinimize` calls `setShowComponent(false)` → unmounts → minimized pill (inside the component) is destroyed
3. The component has a `viewMode === 'minimized'` branch that never renders because the component is already gone

**Fix:** Separate "open" from "minimized" with independent state. The component stays mounted when minimized; only fully closing sets both to false.

```tsx
// ❌ BUG: minimize unmounts the component entirely
const [showTimer, setShowTimer] = useState(false);
// render: {showTimer && <PomodoroTimer onMinimize={() => setShowTimer(false)} />}

// ✅ FIX: separate minimized state keeps component mounted
const [showTimer, setShowTimer] = useState(false);
const [timerMinimized, setTimerMinimized] = useState(false);
// render: {(showTimer || timerMinimized) && <PomodoroTimer
//   onMinimize={() => { setTimerMinimized(true); setShowTimer(false); }}
//   onExpand={() => setTimerMinimized(false)}
//   onClose={() => { setTimerMinimized(false); setShowTimer(false); }}
// />}
```

**Detection clue:** "Minimize button makes it vanish" — the component's minimized view code exists but is unreachable because the parent unmounts it first.

---

## Real-World Impact

From debugging sessions:
- Systematic approach: 15-30 minutes to fix
- Random fixes approach: 2-3 hours of thrashing
- First-time fix rate: 95% vs 40%
- New bugs introduced: Near zero vs common

**No shortcuts. No guessing. Systematic always wins.**
