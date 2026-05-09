# Skills

Last updated: 2026-05-07

## What this workspace is for
This is where all agent skill definitions live. Each skill is a self-contained folder with a SKILL.md manifest and any supporting templates, examples, or data files. The work here is designing, writing, and maintaining reusable AI agent instructions.

## Process
1. Identify the skill's purpose — what triggers it, what it produces
2. Create a folder under `skills/{skill-name}/`
3. Write `SKILL.md` with the YAML frontmatter (name, description with trigger phrases) followed by the full instruction set
4. Add supporting files — templates, examples, schemas — as needed
5. Test the skill by invoking it in a real agent session

## What good looks like
- Each SKILL.md frontmatter has a `description` field with trigger phrases an agent can match
- Instructions are model-agnostic — plain markdown, no HTML/XML, no Claude-specific jargon
- Templates and examples are included when the skill produces structured output
- Skills reference each other explicitly (e.g., create-plan references plan-runner and app-spec)
- A new agent entering this folder can understand what each skill does from the folder name and SKILL.md alone

## What to avoid
- Skills that depend on a specific model's features — keep them cross-model compatible
- Giant monolithic SKILL.md files — if over 300 lines, consider splitting into separate skills
- Ambiguous trigger phrases in the description field
- Forgetting to update EXAMPLES.md or templates when the skill's output format changes

## Files and organization
- `skills/{skill-name}/SKILL.md` — The skill manifest and instructions (required)
- `skills/{skill-name}/EXAMPLES.md` — Worked examples showing the skill in action (optional)
- `skills/{skill-name}/PLAN_TEMPLATE.md` — Templates for skills that produce structured files (optional)
- `skills/{skill-name}/EXAMPLE_*.json` — Example outputs for reference (optional)

Current skills:
- `create-plan` — Author structured project plans for worker agents
- `plan-runner` — Execute plan tasks one at a time via dispatch
- `clief-bootstrap` — Bootstrap CLIEF three-layer folder architecture
- `app-spec` — Generate application specification from a codebase
- `test-runner` — Execute test suites and report results
- `model-strategy` — Generate model routing strategy and config for agentic companies (any domain)