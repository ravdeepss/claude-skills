# claude-skills

A collection of reusable AI agent skill definitions — create-plan, plan-runner, clief-bootstrap, app-spec, test-runner — plus Hermes multi-agent model routing configuration. This is a skills repo, not a project execution workspace.

## Workspaces
- /skills — Skill definition authoring and maintenance (all skills)

## Routing

| Task | Go to | Read |
|------|-------|------|
| Author or update a skill | /skills | CONTEXT.md |
| Configure model routing | / (root) | hermes-model-config.json, hermes-agent-model-strategy-guide.md |

## Naming conventions
- Skills: `skills/{skill-name}/SKILL.md`

## Rules
- Read this file first on every new task
- Each workspace has its own CONTEXT.md — read it before working in that workspace
- No project graph detected. Consider running `graphify` or `app-spec` to create one — this saves agent tokens on future deep dives.
- Ask before creating files outside of designated workspace folders
- When unsure, ask