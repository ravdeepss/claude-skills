---
name: app-spec
description: Generate a new `app-spec.json` from scratch, or update an existing one, for any software project. Use when the user says "create an app-spec", "generate app-spec.json", "document this codebase as a spec", "write a machine-readable spec for this app", "refresh the app-spec", "update the spec — the code changed", "make an app-spec for ike-tasks / ike-saas / {project}", or passes a repo path and asks for a structured feature/architecture snapshot. Produces a single JSON file at the project root that downstream skills (notably `create-plan` and `plan-runner`) consume as the feature catalogue and architecture-of-record. Works for any stack — OSS client-server, SaaS multi-tenant, CLI, library — by adapting which sections are populated.
---

# App Spec

You are a staff-engineer-level codebase analyst. Your job: **produce or refresh an `app-spec.json` at the project root** — a single, machine-readable document that captures the app's architecture, technology, data model, API / data layer, frontend surface, and feature list.

Downstream skills depend on this file:

- `create-plan` reads the `features` array to build its coverage matrix.
- `plan-runner` (test mode) reads the `testEnvironment` block to know the mock strategy.
- Future code-gen / migration / review skills can use it as grounded context.

Every ambiguity you leave in the spec becomes a wrong assumption made by a downstream worker. Write for a worker agent that cannot re-read the repo.

## Modes

This skill has two modes. Pick one at the start based on what the repo contains.

### Mode A — GENERATE (new spec)

No `app-spec.json` exists (or the user says "start fresh"). You'll interview the user, survey the codebase, then write the file.

### Mode B — UPDATE (refresh existing spec)

An `app-spec.json` already exists. The user wants it brought back in sync with the code. You'll diff the spec against reality and emit a minimal patch, preserving sections the user curated by hand.

Announce the mode at the start: *"Running in GENERATE / UPDATE mode."*

## Reference examples (read before drafting)

Two example specs live in this skill folder. Read whichever is closer to the project's shape before drafting:

- `EXAMPLE_OSS_APP_SPEC.json` — a client-server monorepo (React + Express + SQLite, local-process deployment). Closer fit for: libraries, CLIs, single-process web apps, OSS tools, self-hosted apps.
- `EXAMPLE_SAAS_APP_SPEC.json` — a multi-tenant SaaS (React SPA + Supabase Postgres + Realtime + Edge Functions, workspace scoping via RLS). Closer fit for: multi-tenant apps, anything auth+RLS-backed, anything with realtime or MCP, Vercel/Cloudflare deployments.

These aren't templates to clone blindly — the shape is the same but the content must come from the repo you're analyzing. Use them to calibrate depth and which fields matter.

## Mode A — GENERATE workflow

### 1. Intake (short)

Ask only what you can't discover from the repo. Use `AskUserQuestion`. Typical gaps:

- **App id / display name / one-liner description** — the repo name isn't always the product name.
- **Target audience** — end user / internal / developer.
- **License & visibility** — private, OSS, which license.
- **Out-of-scope subsystems** — what the spec should list under `outOfScopeForRegressionV1` or an equivalent carve-out (e.g. "ignore the MCP server", "billing isn't built yet").
- **Parent / sibling spec relation** — if this app was forked from or parallels another (e.g. ike-saas ↔ ike-tasks). Capture that under a `parentSpec` object.
- **Feature tagging convention** — prefix style (`@pomodoro-timer` vs `pomodoro_timer` vs `FEAT-007`). If none exists, default to `kebab-case` and note it in the spec.

Don't ask more than 4 questions at a time. Batch them.

### 2. Codebase survey

Use Glob, Grep, and Read to populate the spec from ground truth. Work top-down:

| Section | Evidence to collect |
|---|---|
| `app.repository` / `architecture.directories` | Root folder tree (`ls`), monorepo config (`pnpm-workspace.yaml`, `nx.json`, `turbo.json`, `lerna.json`). |
| `architecture.pattern` | Pick one of: `client-server-monorepo`, `spa-backend-monorepo`, `supabase-backed-spa-monorepo`, `cli-library`, `micro-services`, `fullstack-framework` (Next/Remix/Rails). Name it, then describe in one sentence. |
| `architecture.deployment` | CI files (`.github/workflows/`, `vercel.json`, `netlify.toml`, `wrangler.toml`, `fly.toml`), start scripts in `package.json` / `Makefile`. |
| `technology.runtime` / language / bundler | Top-level `package.json` / `pyproject.toml` / `Cargo.toml`, `engines`, `packageManager`, `tsconfig.json`. |
| `technology.client` | Framework (React, Vue, Svelte, etc.), router, state lib, styling, entry file. Confirm from import statements, not from outdated READMEs. |
| `technology.backend` / `technology.database` | Server framework, DB driver, auth lib, deployment target. If it's Supabase / Firebase / Planetscale, capture the service name. |
| `database.tables` | Migration files (`migrations/`, `supabase/migrations/`, `prisma/schema.prisma`, `db/schema.rb`). Cite the canonical source file in `schemaSourceOfTruth`. Enumerate tables with columns, PK/FK, enums, indexes. |
| `api` or `dataLayer` | For OSS Express-style: enumerate `app.METHOD(path)` routes. For SaaS / Supabase: enumerate the data-layer module files and the tables each one touches. For trpc / graphql: routers / resolvers. |
| `frontend.routes` / `components` / `hooks` / `keyboardShortcuts` / `localStorageKeys` | Read the router config, component folder, hooks folder. For shortcuts, grep for `keydown` / `useHotkey` / shortcut libs. For storage keys, grep `localStorage.setItem` / `sessionStorage`. |
| `features` | See **Feature extraction** below. |
| `testEnvironment` (optional) | If the project has a mock layer, document: file path, env flag, fixture location, auth mock behavior, realtime mock behavior. |
| `outOfScopeForRegressionV1` | User told you in intake; also mark anything the user flagged as "broken / WIP / not-yet-built". |

**Cite sources.** For each non-obvious claim, note the file you read (e.g. `apps/web/src/router.tsx:42`). You don't have to put citations in the final JSON, but keep them in your working notes so you can defend a value if questioned.

### 3. Feature extraction (the critical step)

The `features` array is what downstream skills rely on most. Each feature needs:

```jsonc
{
  "id": "pomodoro-timer",              // kebab-case, stable, used as @tag in test plans
  "name": "Pomodoro Timer",            // human-readable
  "description": "...",                // one or two sentences
  "entryPoints": ["components/PomodoroTimer.tsx", "hooks/usePomodoro.ts"],
  "dataLayer": ["data/pomodoroSessions.ts"],   // or api endpoints for OSS apps
  "schemaTables": ["pomodoro_sessions"],
  "routes": ["/"],                     // if a dedicated route exists
  "dependencies": ["auth", "workspaces"],      // other feature ids it requires
  "userStories": [
    "As a user, I can start a 25-min pomodoro from the dashboard.",
    "A running pomodoro survives a page reload."
  ],
  "acceptanceCriteria": [
    "Timer counts down in 1-second increments and shows mm:ss.",
    "Completing a pomodoro logs a row to pomodoro_sessions with start_at/end_at.",
    "A second pomodoro cannot start while one is running."
  ],
  "tags": ["productivity", "session-backed"],
  "status": "implemented" | "partial" | "planned"
}
```

How to find features:

1. **Start from the UI.** Grep the router for routes. Each route → one or more features.
2. **Walk the components folder.** Major named components (`KanbanBoard`, `PomodoroTimer`, `TagManager`) map 1:1 to features.
3. **Walk the data layer.** Each CRUD-grouping file (`data/tasks.ts`, `data/projects.ts`) often corresponds to a feature or a group of them.
4. **Read the README / docs.** They'll mention features the code hides (e.g. keyboard shortcuts, CLI subcommands).
5. **Capture cross-cutting features** — auth, workspaces, realtime sync, theming, notifications — even though they don't have a single entry point.

Aim for **15–30 features** for a medium app. Fewer for a library, more for a full product. If you have a "misc" feature, split it — "misc" is a tag you missed.

### 4. Draft the JSON

Top-level shape (omit sections that don't apply):

```jsonc
{
  "$schema": "https://webapp-registry.internal/schema/v1.0.0/app-spec.json",
  "specVersion": "1.0.0",
  "generatedAt": "<ISO timestamp>",
  "generator": "app-spec-skill@<skill-version-or-date>",
  "parentSpec": { /* optional; see intake */ },

  "app": {
    "id": "...", "name": "...", "version": "...", "description": "...",
    "repository": "<path-or-url>", "license": "...", "authors": [...], "tags": [...]
  },

  "architecture": {
    "pattern": "...", "description": "...",
    "deployment": { /* strategy + per-component host */ },
    "directories": { /* map of logical name -> path */ }
  },

  "technology": {
    "runtime": { "name": "...", "requiredVersion": "...", "packageManager": "..." },
    "client":  { /* framework / language / bundler / deps / state / routing / styling */ },
    "backend": { /* kind / components / authenticationModel */ },
    "database":{ /* engine / extensions / features */ }
  },

  "database": {
    "engine": "...",
    "schemaSourceOfTruth": "<file path>",
    "enums": { /* name -> [values] */ },
    "standardColumns": [ /* id, created_at, updated_at, etc. */ ],
    "softDeletePattern": { /* optional */ },
    "tables": [ /* array of table objects with columns + constraints + indexes */ ]
  },

  // ONE OF the following two, depending on architecture:
  "api":        { /* REST / RPC endpoints */ },
  "dataLayer":  { /* client-side data modules for SaaS / Supabase-style apps */ },

  "frontend": {
    "entryPoint": "...", "rootComponent": "...",
    "routes": [ /* path, component, access (public/auth) */ ],
    "viewModes": [ /* for apps with view switchers: All Tasks / Matrix / Kanban */ ],
    "components": [ /* major named components with file paths */ ],
    "hooks": [ /* custom hooks */ ],
    "keyboardShortcuts": [ /* keys + action */ ],
    "localStorageKeys": [ /* key + purpose + format */ ]
  },

  "styling": { /* optional: colors, theme strategy, icon library */ },

  "features": [ /* see Feature extraction above */ ],

  "testEnvironment": {      /* optional — populate if a mock/test layer exists */
    "strategy": "in-app-mock | route-interception | real-db-ephemeral | hybrid",
    "mockLayer": { "file": "...", "envFlag": "...", "fixturesPath": "...", "globals": "..." },
    "authMock": { "credentials": {...}, "behavior": "..." },
    "realtimeMock": "...",
    "timeMock": "..."
  },

  "outOfScopeForRegressionV1": [ /* subsystems explicitly excluded from first-round testing */ ]
}
```

JSON rules:

- Use double quotes, not single.
- Use ISO 8601 for all timestamps.
- Use `kebab-case` for ids; `snake_case` only where it mirrors DB column names; `PascalCase` for component names.
- Keep descriptions to one or two sentences. If you need more, link to a doc file.
- Don't invent fields — stick to the shape above.
- If a field is unknown, omit it. Don't write `"TBD"`.

### 5. Self-review before saving

Walk through:

1. Does every feature have `id`, `description`, at least one of `entryPoints` / `dataLayer`, and `acceptanceCriteria`?
2. Are feature ids unique? Are they all `kebab-case`?
3. Does every table in `database.tables` cite its migration / schema source?
4. Does `schemaSourceOfTruth` point at a file that actually exists?
5. Are all router paths in `frontend.routes` present in the code?
6. Does `outOfScopeForRegressionV1` match what the user told you in intake?
7. Does `testEnvironment.mockLayer.file` exist, or is it listed as "to be created by {task}"?
8. Is `generatedAt` current?

### 6. Save & summarize

Write to `<project-root>/app-spec.json`. Show the user:

- Path (`computer://` link if in Cowork).
- One-paragraph summary: number of tables, number of features, architectural pattern, any sections omitted.
- A suggestion: *"Feed this to `create-test-plan` to generate a regression plan for all {N} features."*

## Mode B — UPDATE workflow

### 1. Load the existing spec

Read `app-spec.json` in full. Capture its `specVersion`, `generatedAt`, `features[*].id`, `database.tables[*].name`, and `frontend.routes[*].path`. These are your baseline.

### 2. Reconcile each section

For each section, compare spec → code. Flag three kinds of drift:

| Drift | Example | Action |
|---|---|---|
| **Added** | New component `BulkEditBar.tsx` with no feature in the spec. | Add a feature entry, or extend an existing one. |
| **Removed** | `legacy-importer` feature still in spec, but the file was deleted. | Remove the feature entry. If the user might still want it, move to `archivedFeatures` with a `removedAt`. |
| **Changed** | `tasks` table now has a `snoozed_until` column. | Patch the table block. |

Be **conservative** about removing. If a feature looks unused but you can't prove it was deleted, ask.

### 3. Preserve human-curated content

Sections most likely to be hand-edited:

- `features[*].description` / `userStories` / `acceptanceCriteria` — keep exact wording unless the user asks for a rewrite.
- `parentSpec`, `outOfScopeForRegressionV1`, `testEnvironment` — user decisions, don't clobber.
- Custom top-level fields (`roadmap`, `glossary`, anything not in the shape above) — preserve verbatim.

Workflow: produce a **diff summary** first (list of proposed changes with rationale), let the user approve, then write. Don't overwrite sections the user curated without explicit OK.

### 4. Bump metadata

- Update `generatedAt`.
- If structural sections changed, bump `specVersion` minor (e.g. `1.0.0 → 1.1.0`). If only content changed, keep `specVersion`.
- Optionally keep a `changeLog` array with `{ date, summary }` entries so downstream consumers can detect drift.

### 5. Save & summarize

Same as GENERATE step 6, but the summary is a **diff**: *"Added 3 features, updated 2 tables, removed 1 component. Preserved all hand-curated descriptions and the `parentSpec` block."*

## Edge cases

- **Monorepo with multiple apps** — write one spec per app (`apps/web/app-spec.json`, `apps/mobile/app-spec.json`) plus a top-level `workspace-spec.json` that points at each. Don't try to cram two apps into one file.
- **Pre-production project (code doesn't run yet)** — still generate the spec, but mark every unimplemented feature `status: "planned"`. This is a legitimate snapshot of intent.
- **No clear schema source** — if migrations are scattered or there's no ORM, create a Phase 0 task for `create-plan` to write a `SCHEMA.md` first, and stub `database.tables` with `"status": "needs-schema-doc"`.
- **Huge app (>50 features)** — still one spec. Split `features` into logical groups via a `group` field (`group: "productivity"`) rather than splitting the file.
- **Spec diverges from code-of-record** — tell the user clearly. Offer UPDATE mode. Don't silently reconcile.
- **User asks to tag the spec to a git commit** — add `"commitSha": "<sha>"` and `"branch": "<name>"` under top level. Useful for reproducing a past spec.

## Relationship to other skills

- `app-spec` (this skill) → ground-truth document.
- `create-plan` reads `features[]`, `testEnvironment`, and `outOfScopeForRegressionV1` to draft plans and structure phases.
- `plan-runner` (test mode) reads `testEnvironment` when classifying failures into the 5 buckets.
- `engineering:system-design` / `engineering:architecture` can _consume_ this file when writing ADRs, but don't use those skills to produce the spec itself — they don't enforce the schema shape.

Keep the spec small, accurate, and current. It's the one file every other skill in the stack trusts.
