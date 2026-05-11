---
name: vercel-monorepo
description: Deploy pnpm workspace monorepos with internal packages to Vercel. Fixes the "module resolution fails because dist/ is gitignored" problem using source exports, path aliases, and Vite resolve aliases. Includes SPA rewrites and build verification.
version: 2.0.0
author: Hermes Agent
metadata:
  hermes:
    tags: [vercel, pnpm, monorepo, vite, typescript, deploy, spa]
    related_skills: [vercel-deployment, supabase-development]
---

# Vercel Deployment for pnpm Monorepos

## When to Use

Deploying a pnpm workspace monorepo to Vercel where internal packages (e.g., `@org/shared`) are consumed by a Vite app in `apps/web/`.

## Problem Pattern

Vercel clones fresh from git. If your internal packages export from `dist/` (built by tsup/tsc), and `dist/` is in `.gitignore`, Vercel gets **zero built files**. Result: TypeScript can't resolve the module → **all types become `error`** → cascading TS errors on every import.

This looks like dozens of individual type bugs but is actually **one root cause**: missing module resolution.

## Fix 1: Export Source Directly

For internal workspace packages consumed by Vite apps, skip the build step. Vite handles TypeScript natively.

**Before (broken on Vercel):**
```json
{
  "main": "dist/index.js",
  "types": "dist/index.d.ts",
  "exports": {
    ".": {
      "types": "./dist/index.d.ts",
      "import": "./dist/index.mjs",
      "require": "./dist/index.js"
    }
  },
  "scripts": {
    "build": "tsup src/index.ts --format esm,cjs --dts"
  }
}
```

**After (works everywhere):**
```json
{
  "main": "src/index.ts",
  "types": "src/index.ts",
  "exports": {
    ".": "./src/index.ts"
  },
  "scripts": {
    "build": "echo 'No build needed'"
  }
}
```

## Fix 2: Path Aliases (Required)

Even with source exports, Vercel may build from `apps/web/` without `pnpm install` at monorepo root. The workspace symlink may not exist. **Add path aliases** so TS + Vite resolve directly:

### `apps/web/tsconfig.json`:
```json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@org/shared": ["../../packages/shared/src"],
      "@org/shared/*": ["../../packages/shared/src/*"]
    }
  }
}
```

### `apps/web/vite.config.ts`:
```ts
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'
import path from 'path'

export default defineConfig({
  plugins: [react()],
  resolve: {
    alias: {
      '@org/shared': path.resolve(__dirname, '../../packages/shared/src'),
    },
  },
})
```

**Verify without symlink:** Delete `node_modules/@org` and run `tsc -b` — should pass via path alias.

## Vercel Config

Place `vercel.json` in BOTH locations for flexibility:

### `apps/web/vercel.json` (Root Directory = `apps/web`):
```json
{
  "framework": "vite",
  "buildCommand": "cd ../.. && npx pnpm install && cd apps/web && npx tsc -b && npx vite build",
  "outputDirectory": "dist",
  "installCommand": "cd ../.. && npx pnpm install --frozen-lockfile",
  "rewrites": [
    { "source": "/(.*)", "destination": "/index.html" }
  ]
}
```

### `vercel.json` at repo root (Root Directory = `/`):
```json
{
  "framework": "vite",
  "buildCommand": "cd apps/web && npx tsc -b && npx vite build",
  "outputDirectory": "apps/web/dist",
  "installCommand": "npx pnpm install --frozen-lockfile",
  "rewrites": [
    { "source": "/(.*)", "destination": "/index.html" }
  ]
}
```

## Verification Checklist

1. `pnpm install` — relinks workspace
2. `rm -rf apps/web/node_modules/@org` — remove symlink (simulate Vercel)
3. `cd apps/web && npx tsc -b` — typecheck clean WITHOUT symlink
4. `cd apps/web && npx vite build` — build succeeds WITHOUT symlink
5. `pnpm install` — restore symlink for local dev
6. `git push` — Vercel auto-deploys

## Pitfalls

- **`pnpm-lock.yaml` MUST be committed** (not gitignored)
- **Source exports alone are not enough** — add tsconfig paths + vite aliases. Without symlink, neither tsc nor vite can find files
- **Don't assume Root Directory setting** — put vercel.json in both root and apps/web/
- **Test without symlink BEFORE pushing** — `rm -rf apps/web/node_modules/@org && tsc -b` is the definitive test
- **ALWAYS check Vercel build status after pushing** — use `mcp_vercel_getDeployments`. State: INITIALIZING → BUILDING → READY (or ERROR). Don't report success until READY
- **`.npmrc` with `prefix` setting breaks Vercel** — Vercel rejects `config prefix cannot be changed from project config`. Put `.npmrc` in `.gitignore`, create `.npmrc.example` as reference
- **Pre-existing TS errors WILL fail Vercel** if `tsc -b` is in buildCommand — fix ALL errors or remove `tsc -b` from buildCommand if intentional
- **`moduleResolution: "Bundler"`** is required for source-direct exports
- **Hermes-specific:** `vite build` may be falsely detected as long-running server. Workaround: `node -e "const {build} = require('vite'); build({logLevel:'info'}).then(() => console.log('BUILD OK')).catch(e => {console.error(e); process.exit(1)})"`
- **SPA rewrites required** — `{ "source": "/(.*)", "destination": "/index.html" }` for client-side routing
- **Deployment Protection:** Private repos return 401 with `_vercel_sso_nonce` cookie. Disable in Dashboard → Settings → Deployment Protection
