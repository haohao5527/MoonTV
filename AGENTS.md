# AGENTS.md

## Project Overview

MoonTV is a video aggregation player built with **Next.js 14 (App Router)**, **TypeScript**, and **Tailwind CSS**. It aggregates content from multiple third-party video sources and supports online playback via ArtPlayer + HLS.js.

## Quick Commands

```bash
pnpm dev              # Start dev server (runs gen:runtime + gen:manifest first)
pnpm build            # Production build (runs gen:runtime + gen:manifest first)
pnpm lint             # ESLint check
pnpm lint:fix         # ESLint fix + Prettier format
pnpm lint:strict      # ESLint with zero warnings tolerance
pnpm typecheck        # TypeScript type check (tsc --noEmit)
pnpm test             # Run Jest tests
pnpm format           # Prettier format all files
```

**Always run `pnpm gen:runtime && pnpm gen:manifest` before build/dev if the auto-run doesn't trigger.** These scripts are auto-called by `dev` and `build` scripts but are critical — they generate `src/lib/runtime.ts` from `config.json` and `public/manifest.json` from `SITE_NAME` env var.

## Codegen: Do Not Skip

Two scripts run before build/dev and must not be forgotten:

1. `pnpm gen:runtime` — reads `config.json` and generates `src/lib/runtime.ts`. This file is **auto-generated, never edit it manually**.
2. `pnpm gen:manifest` — reads `SITE_NAME` env and generates `public/manifest.json`.

The pre-commit hook also runs `pnpm gen:version` which updates `src/lib/version.ts` and `VERSION.txt` with a timestamp.

## Package Manager

Use **pnpm** (v10.12.4 pinned in `package.json`). Do not use npm or yarn.

## Node Version

**v20.10.0** (see `.nvmrc`).

## Path Aliases

- `@/*` → `src/*`
- `~/*` → `public/*`

These are configured in both `tsconfig.json` and `jest.config.js`.

## Architecture

```
src/
  app/              # Next.js App Router pages and API routes
    api/            # Server-side API endpoints (search, favorites, playrecords, admin, etc.)
    play/           # Video playback page
    search/         # Search results page
    admin/          # Admin panel (requires non-localStorage storage)
    login/          # Login page
  components/       # React components
  lib/              # Core logic
    config.ts       # Config loading (merges file config + admin DB config)
    db.ts           # Storage abstraction layer (singleton DbManager)
    auth.ts         # Cookie-based auth
    downstream.ts   # Video source API adapters
  middleware.ts     # Auth middleware (cookie signature verification)
  styles/           # Global CSS
config.json         # Source site definitions (api_site entries, cache_time, custom_category)
scripts/            # Build-time code generation scripts
proxy.worker.js     # Cloudflare Worker for CORS proxy (standalone, not part of Next.js app)
```

## Storage Backends

Controlled by `NEXT_PUBLIC_STORAGE_TYPE` env var. Four modes:

| Mode | Use case | Multi-user sync |
|------|----------|-----------------|
| `localstorage` | Default, client-side only | No |
| `redis` | Docker self-hosted | Yes |
| `upstash` | Vercel/serverless | Yes |
| `d1` | Cloudflare Pages | Yes |

Storage implementations: `src/lib/redis.db.ts`, `src/lib/upstash.db.ts`, `src/lib/d1.db.ts`.

## Config System

`config.json` at root defines video sources (`api_site`). This is merged at runtime with admin config from the database (when using non-localStorage storage). The `getConfig()` function in `src/lib/config.ts` handles this merge.

In Docker, `config.json` is read from disk at startup (`DOCKER_ENV=true`). Otherwise, the build-time generated `src/lib/runtime.ts` is used.

## ESLint Rules

- `simple-import-sort` is enabled — imports must follow the configured sort order
- `unused-imports` plugin — unused imports are warnings (not errors)
- Strict lint (`pnpm lint:strict`) runs with `--max-warnings=0`
- Pre-commit hook runs `lint-staged` which applies `eslint --max-warnings=0` + `prettier -w`

## Commit Convention

Enforced by `commitlint` via `.husky/commit-msg`. Must use [Conventional Commits](https://www.conventionalcommits.org/):

```
feat: add new feature
fix: bug fix
docs: documentation
chore: maintenance
style: formatting
refactor: code restructuring
ci: CI changes
test: tests
perf: performance
revert: revert
vercel: vercel-specific
```

## Docker

- Multi-stage build: deps → builder → runner
- Dockerfile forces `runtime = 'nodejs'` on all route files (replaces edge runtime)
- Uses `start.js` as entrypoint which starts the standalone server and runs a cron job
- Build output: `output: 'standalone'` in `next.config.js`

## Testing

- Framework: Jest with `jest-environment-jsdom`
- Setup file: `jest.setup.js`
- SVG mocks at `src/__mocks__/svg.tsx`
- Path aliases mirror tsconfig

## Key Gotchas

- `src/lib/runtime.ts` is auto-generated — do not edit manually, changes will be overwritten
- `reactStrictMode` is **disabled** in `next.config.js`
- PWA is disabled in development mode (`next-pwa` with `disable: process.env.NODE_ENV === 'development'`)
- `config.json` is read once at startup in Docker mode — config changes require a restart
- The `proxy.worker.js` is a standalone Cloudflare Worker, not part of the Next.js app
- Pre-commit hook auto-generates version files — expect `src/lib/version.ts` and `VERSION.txt` to be staged
