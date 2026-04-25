---
name: test-runner
description: "[DEPRECATED — merged into plan-runner] This skill's functionality (test → investigate → fix → retest loop) is now part of plan-runner's test mode. Use plan-runner instead. Triggers: 'run regression tests', 'run the test plan', 'run feature X tests', 'test → fix until green', 'regression check for {feature}'."
---

# Test Runner — DEPRECATED

This skill has been merged into **`plan-runner`** as its **test mode**.

All test-runner functionality — the autonomous test → investigate → fix → retest loop, failure classification (A–E buckets), escalation, budget knobs, and final reporting — is now available via `plan-runner` when:

- A task is a verification gate.
- A task's goal/spec is test-focused.
- The user explicitly requests test execution ("run regression tests", "test → fix until green", etc.).

**Use `plan-runner` instead of this skill.**
