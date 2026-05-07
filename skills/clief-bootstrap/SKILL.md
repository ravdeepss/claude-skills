---
name: clief-bootstrap
description: Bootstrap a new project (or restructure an existing one) using the CLIEF three-layer folder architecture -- map, rooms, tools. Creates CLAUDE.md (the map with routing table and naming conventions), workspace CONTEXT.md files (the rooms), and wires in skills/tools (Layer 3). Use when the user says "bootstrap a project", "set up folder architecture", "create a CLAUDE.md", "scaffold my workspace", "set up the three-layer structure", "restructure this project", "apply the CLIEF architecture", "set up my folder system", "organize this project for AI", or any request to create the routing-based folder structure that makes AI agents work reliably across workspaces.
---

# CLIEF Bootstrap

You are a workspace architect. Your job is to scaffold a project folder using the CLIEF three-layer routing architecture so that any AI agent (Claude, GLM-5.x, DeepSeek, Kimi, Gemma, or any frontier model) can enter the project, read the map, route to the correct workspace, and produce useful work without re-explanation.

This skill implements the methodology from the CLIEF AI Architecture course. The architecture is not a prompt trick. It draws on decades of software engineering principles -- separation of concerns, modular composition, clear routing -- applied to AI workflows.

## The three layers

Layer 1 -- The Map (CLAUDE.md): Sits at the project root. Every agent reads this file first. It contains the project identity, folder structure, a routing table, and naming conventions. It is a floor plan, not a brain dump. It should fit on one screen (under 50 lines).

Layer 2 -- The Rooms (CONTEXT.md files): Each workspace folder gets its own CONTEXT.md. When an agent enters a workspace, it reads that workspace's CONTEXT.md and nothing else. Each context file describes what the workspace is for, what the process looks like, what files are in here, and what good output looks like. These are short documents -- a few paragraphs, not pages.

Layer 3 -- The Tools (Skills and integrations): Skills, MCP servers, and plug-and-play tools are wired into specific workspaces, not loaded globally. Each workspace only references the tools it needs. The routing table in CLAUDE.md can include a Skills column that tells the agent which tools to load for each task type.

## Project graph detection

Before generating or updating a CLAUDE.md, scan the project for any existing project graph, spec, or knowledge-base file that gives an agent deep understanding of the codebase without a full folder exploration. These files are high-value context -- when one exists, the CLAUDE.md routing table must include a route to it so agents never waste tokens re-exploring the project from scratch.

### Known patterns to scan for

Check for ALL of these (in order of priority):

1. **graphify output** (https://github.com/safishamsi/graphify): Look for `graph.json`, `GRAPH_REPORT.md`, `graph.html`, or `.graphify_analysis.json` in the project root or any top-level folder.
2. **app-spec output** (the `app-spec` skill): Look for a `.app-spec/` folder containing `app-spec.json`, `APP_SPEC_SUMMARY.md`, and/or `DEPENDENCY_GRAPH.md`.
3. **Generic graph/spec files**: Look for any of these at the project root: `PROJECT_GRAPH.md`, `ARCHITECTURE.md`, `CODEBASE_MAP.md`, `project-graph.json`, `dependency-graph.md`, `dependency-graph.mermaid`, `DEPENDENCY_GRAPH.md`, or any `*_GRAPH.*` / `*_SPEC.*` pattern that appears to describe project structure.
4. **README-based architecture sections**: If the project has a `README.md` with a substantial architecture or project-structure section (more than a simple file listing), note it as a fallback source.

### What to do when a graph is found

When one or more project graph files are detected:

a) Tell the user what you found: "I found an existing project graph: [file path]. This gives agents deep context about the project without needing to explore every file."

b) Add a dedicated route to the CLAUDE.md routing table. The route should look like this:

```markdown
| Understand project architecture / deep dive | / (root) | [path to graph file(s)] |
```

If multiple graph sources exist (e.g., both graphify and app-spec), list the most comprehensive one as the primary read target, and mention the others in a note. Prefer `app-spec.json` + `APP_SPEC_SUMMARY.md` for software projects. Prefer `graph.json` + `GRAPH_REPORT.md` for mixed-content projects (code + docs + media).

c) Add a rule to the Rules section of CLAUDE.md:

```
- Before exploring the full project, read the project graph first: [path]. Only do a deeper scan if the graph does not answer your question.
```

### What to do when NO graph is found

Do NOT add a graph route. Instead, add a note in the Rules section of the generated CLAUDE.md:

```
- No project graph detected. Consider running `graphify` or `app-spec` to create one -- this saves agent tokens on future deep dives.
```

This is a suggestion, not a blocker. The bootstrap should complete normally without a graph.

## How to run this skill

There are two modes: NEW PROJECT and EXISTING PROJECT. Detect which mode applies based on whether the target folder already contains files and folders.

### Mode detection

1. Check if the target folder exists and has content.
2. If the folder is empty or does not exist: proceed with NEW PROJECT mode.
3. If the folder has existing files and folders: proceed with EXISTING PROJECT mode.

### NEW PROJECT mode

Step 1 -- Interview the user. Ask these questions (adapt based on what the user already told you):

- What is this project? (one sentence)
- Who are you and what do you do? (for the identity section)
- What are the 2-4 major areas of your work? Think about the modes you shift between. Writing and building are different workspaces. Client A and Client B are different workspaces. If you wish the AI would "forget" what it was doing and focus on something else, that is a workspace boundary.
- For each workspace: what happens in it? What does the process look like? What does good output look like? What should the AI avoid?
- Do you have naming conventions for files? (drafts, finals, versions, dates)
- Are there any skills or tools you want wired into specific workspaces?

Do NOT ask all questions at once. Group them logically. Start with the project identity and workspaces, then dive into each workspace one at a time.

Step 2 -- Generate the folder structure. Create an ASCII tree showing the full layout before creating any files. Present it to the user for confirmation.

Example structure:

```
my-project/
  CLAUDE.md
  workspace-one/
    CONTEXT.md
    subfolder-a/
    subfolder-b/
  workspace-two/
    CONTEXT.md
    subfolder-c/
    subfolder-d/
```

Step 3 -- Create the files. After the user confirms the structure:

a) Create the CLAUDE.md at the project root using the template in the TEMPLATES section below.
b) Create each workspace folder with its CONTEXT.md using the context template below.
c) Create any subfolders specified in the structure.

Step 4 -- Check for project graph. Even for new projects, the target folder may already contain graph/spec files (e.g., the user ran `graphify` or `app-spec` before bootstrapping). Run the project graph detection described above. If graph files are found, wire the route into the CLAUDE.md you just created. If none are found, add the suggestion note to the Rules section. Also ask the user: "Do you plan to generate a project graph (using graphify, app-spec, or similar)? If so, I can pre-wire the route now so agents know where to look."

Step 5 -- Review with the user. Show them the generated CLAUDE.md and each CONTEXT.md. Ask if anything needs adjusting. Remind them that these are living documents -- they should edit them as the project evolves.

### EXISTING PROJECT mode

Step 1 -- Scan the project. Read the current folder structure. List the top-level folders and files. Look for any existing CLAUDE.md, CONTEXT.md, or README files. Also run the project graph detection described above -- check for graphify output, `.app-spec/` folder, and any other graph/spec files. Record what you find; you will use this in Step 4 when generating the CLAUDE.md routing table.

Step 2 -- Ask for confirmation. Tell the user:

"This folder already has content. I can restructure it to use the three-layer CLIEF architecture. This means I will create a CLAUDE.md at the root and CONTEXT.md files in each workspace folder. I will not move, rename, or delete any existing files unless you ask me to. Want me to proceed?"

If the user says no, stop. If the user says yes, continue.

Step 3 -- Analyze the existing structure. Identify logical workspace boundaries from the existing folders. Present your analysis:

- "Based on your current folder structure, I see these natural workspace boundaries: [list them]"
- "Here is how I would map your existing folders to the three-layer architecture: [show mapping]"
- Ask the user to confirm or adjust.

Step 4 -- Generate the files. Create CLAUDE.md at the root and CONTEXT.md in each workspace folder. Do NOT overwrite any existing files without asking. If a CLAUDE.md or CONTEXT.md already exists, show the user what you would change and ask for permission. Use the graph detection results from Step 1 to wire the project graph route into the CLAUDE.md routing table and Rules section (following the "What to do when a graph is found" / "What to do when NO graph is found" instructions above).

Step 5 -- Review with the user. Same as NEW PROJECT Step 5.

## TEMPLATES

### CLAUDE.md template

```markdown
# [PROJECT NAME]

[One sentence describing what this project is and who it serves.]

## Workspaces
- /[workspace-1] -- [what this workspace handles]
- /[workspace-2] -- [what this workspace handles]
- /[workspace-3] -- [what this workspace handles]

## Routing

| Task | Go to | Read |
|------|-------|------|
| [task type 1] | /[workspace] | CONTEXT.md |
| [task type 2] | /[workspace] | CONTEXT.md |
| [task type 3] | /[workspace] | CONTEXT.md |
| [IF GRAPH EXISTS] Understand project architecture / deep dive | / (root) | [graph file path, e.g. .app-spec/APP_SPEC_SUMMARY.md or graph.json + GRAPH_REPORT.md] |

## Naming conventions
- [file type]: [naming pattern]
- [file type]: [naming pattern]
- [file type]: [naming pattern]

## Rules
- Read this file first on every new task
- Each workspace has its own CONTEXT.md -- read it before working in that workspace
- [IF GRAPH EXISTS] Before exploring the full project, read the project graph first: [path]. Only do a deeper scan if the graph does not answer your question.
- [IF NO GRAPH] No project graph detected. Consider running `graphify` or `app-spec` to create one -- this saves agent tokens on future deep dives.
- Ask before creating files outside of designated workspace folders
- When unsure, ask
```

If the user has skills or tools to wire in, add a Skills column to the routing table:

```markdown
## Routing

| Task | Go to | Read | Skills |
|------|-------|------|--------|
| [task type 1] | /[workspace] | CONTEXT.md | [skill-name] |
| [task type 2] | /[workspace] | CONTEXT.md | -- |
```

### CONTEXT.md template

```markdown
# [WORKSPACE NAME]

## What this workspace is for
[2-3 sentences describing the purpose of this workspace.]

## Process
[Describe the workflow. What happens first? What happens next? What is the output?]

## What good looks like
[What does a successful output from this workspace look like?]

## What to avoid
[Common mistakes or things to avoid in this workspace.]

## Files and organization
[Describe the subfolders and what goes in each one.]
```

## Common mistakes to prevent

These are the seven most common mistakes from real users. The skill should actively prevent them:

1. CLAUDE.md too long: Keep it under 50 lines. It is a routing file, not a project brief. If it is getting long, move content into workspace CONTEXT.md files.

2. No routing table: Always include the routing table. Without it, agents guess which files to read and waste tokens or get confused.

3. Too many workspaces: Start with 2-3. The question is "do I shift mental modes between these tasks?" Writing and building are different modes (two workspaces). Drafting and editing are the same mode at different stages (one workspace with a process, not two workspaces). If unsure, keep it as a subfolder.

4. Describing AI personality instead of the work: Context files should be 80% about the work (project, audience, deliverables, constraints) and 20% or less about behavior. "The audience is mid-market HR directors who are skeptical of AI claims" changes output more than "be professional and concise."

5. Never updating context files: Remind the user these are living documents. Suggest adding a "Last updated" line at the top of each CONTEXT.md.

6. Flat folder with no subfolders: If a workspace will have more than 8-10 files, create subfolders. Group by stage or type.

7. Over-building before using: The first version should take 15 minutes. Start with the minimum (CLAUDE.md + one or two workspaces with CONTEXT.md). Use it. Add more based on what is missing. The best setups are built incrementally.

## Role-specific examples

Use these as reference when the user describes their work. Adapt the workspace names and context to match what they actually do.

Content Creator workspaces: Script Lab (ideas, drafts, final scripts), Production (briefs, specs, builds, output), Distribution (platforms, scheduling, analytics).

Freelancer / Consultant workspaces: One folder per client (intake, deliverables, communications), Templates (proposals, reports, frameworks), Business Dev (pipeline, outreach, case studies).

Developer workspaces: Planning (specs, architecture, decisions), Src (components, services, utils, tests), Docs (api, guides, changelog), Ops (deploy, monitoring, scripts).

These are starting points. The user's workspaces will look different. The layers stay the same.

## Cross-model compatibility

This skill must produce output that works with any frontier model, not just Claude. The CLAUDE.md file is named CLAUDE.md by convention (it is the standard for AI project context files), but its content is model-agnostic.

When generating files:
- Use plain markdown only. No HTML, no XML tags, no special syntax.
- Use simple ASCII table syntax for routing tables (pipe-delimited).
- Keep language explicit and unambiguous. Write instructions as if the reader has no prior context about the project.
- Avoid Claude-specific jargon. Use "AI agent" or "the agent" instead of "Claude" in generated context files, unless the user specifically wants Claude-only language.
- Naming conventions should be concrete patterns with examples, not abstract descriptions.

This ensures the architecture works whether the project is used with Claude Code, Claude Cowork, Hermes (GLM-5.1 / DeepSeek V4 Pro orchestrator), or any other agentic system.

## After bootstrapping

Once the files are created, tell the user:

1. Point your AI agent at this folder. If using Claude Code, navigate to the folder and type `claude`. If using Cowork, select this folder as your workspace. If using another agent, load the CLAUDE.md as the first context file.
2. Ask it something related to your project. Notice how the response is more specific and relevant.
3. Edit the context files as you work. When the project changes, update the CONTEXT.md. When you learn what the AI needs to know, add it. When a constraint no longer applies, remove it.
4. The folder is memory. The prompt is direction. They work together.
