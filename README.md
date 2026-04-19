# claude-skills

A small repository of reusable Claude skill definitions for planning and executing agent-driven work.

## What this project contains

- `skills/create-plan/SKILL.md`
  - A skill for writing structured project plans as `<NAME>_PLAN.md` files.
  - Designed for an Opus-class agent to produce explicit, worker-ready task plans.
- `skills/plan-runner/SKILL.md`
  - A skill for executing plan tasks one at a time from a structured plan file.
  - Designed to dispatch each task to a worker model or export a prompt for an external model.
- `skills/create-plan/PLAN_TEMPLATE.md`
  - A template showing the exact plan format expected by `plan-runner`.

## Purpose

This repository is meant to store Claude skill recipes rather than application code.
Use it to:

1. Define a reusable planning skill that breaks a project into a machine-executable task plan.
2. Define a companion execution skill that reads the plan and runs the next task.

## How it works

- `create-plan` is the authoring skill.
  - It interviews the user, writes a clear plan, and saves it as `<NAME>_PLAN.md` at the project root.
  - The plan format includes architecture decisions, worker instructions, context files, phased tasks, and a dependency graph.
- `plan-runner` is the dispatcher skill.
  - It finds the plan file, selects the next incomplete task, and builds a prompt for a worker model.
  - It can run the task through a Claude sub-agent or generate a self-contained prompt file for external models.

## Recommended workflow

1. Start by authoring a plan with `create-plan`.
2. Review or edit the resulting `<NAME>_PLAN.md` if needed.
3. Execute the plan using `plan-runner` with commands like "run next task" or "prep task 1.1".

## Repository layout

- `LICENSE` — project license
- `README.md` — this file
- `skills/` — directory containing each skill as its own folder
  - `create-plan/`
    - `SKILL.md`
    - `PLAN_TEMPLATE.md`
  - `plan-runner/`
    - `SKILL.md`

## Extending this workspace

To add a new skill:

1. Create a new folder under `skills/`.
2. Add a `SKILL.md` file with the skill manifest and instructions.
3. If the skill relies on a structured plan or output format, include any templates or examples.

## Notes

This repo currently contains only skill definitions and documentation. The actual skill execution happens in the Claude agent environment that loads these skill manifests.
