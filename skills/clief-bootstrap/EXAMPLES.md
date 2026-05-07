# CLIEF Bootstrap -- Full Examples

Three complete examples showing how the three-layer architecture adapts to different kinds of work. Use these as reference when bootstrapping a project. The layers stay the same. The labels change.

---

## Example 1: Content Creator

### Folder structure

```
my-content-project/
  CLAUDE.md
  script-lab/
    CONTEXT.md
    ideas/
    drafts/
    final/
  production/
    CONTEXT.md
    briefs/
    specs/
    builds/
    output/
  distribution/
    CONTEXT.md
    platforms/
    scheduling/
    analytics/
```

### CLAUDE.md

```markdown
# My Content Project

I create educational videos and written content for an audience learning AI tools.

## Workspaces
- /script-lab -- Idea development, writing, drafts
- /production -- Building and producing content
- /distribution -- Publishing, scheduling, repurposing

## Routing

| Task | Go to | Read |
|------|-------|------|
| Write or brainstorm | /script-lab | CONTEXT.md |
| Build or produce | /production | CONTEXT.md |
| Publish or repurpose | /distribution | CONTEXT.md |

## Naming conventions
- Drafts: topic-name_draft.md
- Final scripts: topic-name_final.md
- Published: YYYY-MM-platform-topic.md

## Rules
- Read this file first on every new task
- Each workspace has its own CONTEXT.md -- read it before working in that workspace
- Ask before creating files outside of designated workspace folders
- When unsure, ask
```

### script-lab/CONTEXT.md

```markdown
# Script Lab

## What this workspace is for
This is where thinking happens. Ideas go in, drafts come out. All writing starts here before moving to production.

## Process
1. Capture ideas in /ideas as short notes
2. Develop promising ideas into drafts in /drafts
3. Polish drafts and move finals to /final
4. Hand off to /production when the script is locked

## What good looks like
- Scripts that sound like a real person talking, not a textbook
- Clear structure: hook, explanation, example, payoff
- Every script has a "What You Will Get From This" at the top

## What to avoid
- Jargon without explanation
- Starting with "In today's world" or any variation
- Scripts longer than 2000 words without a clear reason

## Files and organization
- /ideas -- one file per idea, named topic-name_idea.md
- /drafts -- working drafts, named topic-name_draft.md
- /final -- locked scripts ready for production, named topic-name_final.md
```

---

## Example 2: Freelancer / Consultant

### Folder structure

```
my-consulting-practice/
  CLAUDE.md
  client-alpha/
    CONTEXT.md
    intake/
    deliverables/
    communications/
  client-beta/
    CONTEXT.md
    intake/
    deliverables/
    communications/
  templates/
    CONTEXT.md
    proposals/
    reports/
    frameworks/
  business-dev/
    CONTEXT.md
    pipeline/
    outreach/
    case-studies/
```

### CLAUDE.md

```markdown
# My Consulting Practice

I am a strategy consultant working with mid-market SaaS companies on go-to-market execution.

## Active Clients
- /client-alpha -- Q2 GTM strategy engagement
- /client-beta -- Product positioning refresh

## Internal
- /templates -- Reusable proposals, reports, frameworks
- /business-dev -- Pipeline, outreach, case studies

## Routing

| Task | Go to | Read |
|------|-------|------|
| Client work for Alpha | /client-alpha | CONTEXT.md |
| Client work for Beta | /client-beta | CONTEXT.md |
| Build a new proposal | /templates | CONTEXT.md, then client folder |
| Outreach or pipeline | /business-dev | CONTEXT.md |

## Rules
- Never reference one client's information in another client's workspace
- Proposals always start from /templates and get customized in the client folder
- Deliverables go in /client-[name]/deliverables, drafts stay in working folders
- Read this file first on every new task
```

### client-alpha/CONTEXT.md

```markdown
# Client Alpha -- Q2 GTM Strategy

Last updated: 2026-05-01

## What this workspace is for
All work related to the Alpha engagement. They are a 50-person B2B SaaS company launching a new product line in Q3. We are building their go-to-market strategy.

## Process
1. Review intake materials in /intake
2. Build deliverables in /deliverables
3. Draft client communications in /communications
4. All deliverables go through one internal review before sending

## What good looks like
- Strategic recommendations backed by data, not opinions
- Deliverables formatted to their brand guidelines (see /intake/brand-guide.pdf)
- Communications that are direct and concise -- this client values brevity

## What to avoid
- Referencing any other client's data or strategy
- Generic recommendations that could apply to any company
- Jargon the client has not used themselves

## Files and organization
- /intake -- SOW, discovery notes, brand guide, competitor analysis
- /deliverables -- finished work products
- /communications -- email drafts, meeting agendas, status updates
```

---

## Example 3: Developer

### Folder structure

```
my-app/
  CLAUDE.md
  planning/
    CONTEXT.md
    specs/
    architecture/
    decisions/
  src/
    CONTEXT.md
    components/
    services/
    utils/
    tests/
  docs/
    CONTEXT.md
    api/
    guides/
    changelog/
  ops/
    CONTEXT.md
    deploy/
    monitoring/
    scripts/
```

### CLAUDE.md

```markdown
# My App

TaskFlow -- a project management tool for small engineering teams.

## Tech Stack
- Frontend: React + TypeScript
- Backend: Node.js + Express
- Database: PostgreSQL via Supabase
- Deploy: Vercel (frontend) + Supabase (backend)

## Workspaces
- /planning -- Specs, architecture, decisions
- /src -- Application code
- /docs -- Documentation
- /ops -- Deployment and operations

## Routing

| Task | Go to | Read | Skills |
|------|-------|------|--------|
| Spec a feature | /planning | CONTEXT.md | -- |
| Write code | /src | CONTEXT.md | testing |
| Write docs | /docs | CONTEXT.md | doc-authoring |
| Deploy or debug infra | /ops | CONTEXT.md | -- |

## Naming conventions
- Specs: feature-name_spec.md
- Components: PascalCase (e.g., TaskCard.tsx)
- Tests: feature-name.test.ts
- Decision records: YYYY-MM-DD-decision-title.md

## Rules
- Read this file first on every new task
- Each workspace has its own CONTEXT.md -- read it before working in that workspace
- All new features start with a spec in /planning before any code is written
- Tests are required for all new code in /src
```

### src/CONTEXT.md

```markdown
# Source Code

Last updated: 2026-05-01

## What this workspace is for
The application codebase. All React components, backend services, utilities, and tests live here.

## Process
1. Read the relevant spec from /planning before starting
2. Write tests first (TDD preferred)
3. Implement the feature
4. Run the test suite before marking as done
5. Update /docs if the change affects the API or user-facing behavior

## What good looks like
- TypeScript with strict mode, no any types
- Components are small (under 150 lines) and composable
- Services handle one domain each
- Every exported function has a JSDoc comment
- Tests cover happy path and at least one edge case

## What to avoid
- Direct database calls from components -- always go through a service
- Inline styles -- use Tailwind utility classes
- God components that do everything
- Skipping tests for "simple" changes

## Files and organization
- /components -- React components, one per file, PascalCase
- /services -- Backend services, one per domain (auth, tasks, teams)
- /utils -- Shared utilities (formatting, validation, helpers)
- /tests -- Test files mirroring the structure above
```
