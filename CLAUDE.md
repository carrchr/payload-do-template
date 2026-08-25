# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Payload CMS Skill

This project uses the Payload CMS skill at `.claude/skills/payload/`. Start with `.claude/skills/payload/SKILL.md` for a quick reference, then see `.claude/skills/payload/reference/` for detailed docs (fields, hooks, access control, queries, adapters, plugins).

## Project Overview

"Proof of Concept" Marketing Website (using PayloadCMS template), built on the official Payload Website Template. Uses PostgreSQL 17 (via `@payloadcms/db-postgres`), Next.js 16 App Router, Tailwind CSS 4, and shadcn/ui components. Will deploy to Digital Ocean App Platform.

`docker-compose.yml` spins up a local Postgres 17 container plus the app (pnpm-based); `.env.example` documents the required env vars including `DATABASE_URL` in Postgres form.

## Commands

```bash
pnpm dev                    # start dev server at localhost:3000
pnpm build                  # production build (also runs postbuild -> next-sitemap)
pnpm start                  # run production build

pnpm lint                   # eslint
pnpm lint:fix

pnpm test                   # runs test:int then test:e2e
pnpm test:int               # vitest, integration tests in tests/int/**/*.int.spec.ts
pnpm test:e2e               # playwright, tests in tests/e2e/**

pnpm payload migrate:create # create a Postgres migration after schema changes
pnpm payload migrate        # run pending migrations
pnpm generate:types         # regenerate src/payload-types.ts from the Payload config
pnpm generate:importmap     # regenerate admin panel import map (needed after adding admin components)
```

Run a single vitest test: `pnpm vitest run tests/int/api.int.spec.ts -t "test name"`
Run a single playwright test: `pnpm playwright test tests/e2e/frontend.e2e.spec.ts -g "test name"`

Postgres in dev uses `push: true` (schema auto-pushed, no migrations needed locally). Once a database is pointed at production, set `push: false` and use migrations — see README "Working with Postgres" section for details.

After changing any collection/global/field shape, run `pnpm generate:types` so `src/payload-types.ts` stays in sync — code importing stale types will silently drift from the actual schema.

## Architecture

**Single Next.js app, two route groups sharing one Payload instance:**
- `src/app/(payload)/` — Payload admin panel (`/admin`) and REST/GraphQL API (`/api`), generated/managed by Payload.
- `src/app/(frontend)/` — the public marketing site (pages, posts, search, sitemaps), hand-written React reading from Payload's Local API.

`src/payload.config.ts` is the root of the Payload app: registers collections, globals, plugins, the Postgres adapter, and the jobs-queue access rule (logged-in user OR `CRON_SECRET` bearer token, for scheduled publishing).

**Collections** (`src/collections/`): `Pages`, `Posts`, `Media`, `Categories`, `Users`. `Pages` and `Posts` are the two content types rendered by the frontend; both are draft-enabled (`versions.drafts`) and layout-builder enabled.

**Layout builder blocks** (`src/blocks/`): `Hero`, `Content`, `Media`, `CallToAction`, `ArchiveBlock`, `Form`, `Code`, `RelatedPosts`. `src/blocks/RenderBlocks.tsx` maps a page/post's block array to React components. `src/heros/` (`HighImpact`/`MediumImpact`/`LowImpact`/`PostHero`) is a separate, parallel system for the hero region at the top of a page, selected via `RenderHero.tsx`.

**Access control** (`src/access/`): three composable primitives — `anyone` (public), `authenticated` (any logged-in user), `authenticatedOrPublished` (public can read published docs; logged-in users see everything). Collections import and combine these rather than writing inline access logic.

**Draft preview + on-demand revalidation**: Pages/Posts use Payload's draft/versions system for previewing unpublished content. `afterChange` hooks (see `src/collections/Pages/hooks/revalidatePage.ts`, `src/hooks/revalidateRedirects.ts`) call Next.js revalidation when a document's `_status` becomes `published`, so the statically-rendered frontend stays in sync. Because of this, most frontend fetches use `no-store`/`force-dynamic` rather than relying on Next.js's own cache (see README "Cache" section) — self-hosted (not Payload Cloud) so this is the operative caching strategy.

**Plugins** (`src/plugins/index.ts`) wire up: redirects (`pages`/`posts`), nested-docs (`categories`), SEO (title/URL generators), form-builder (contact/lead forms, payment fields disabled), and search (indexes `posts`, with custom fields via `src/search/fieldOverrides.ts` and `src/search/beforeSync.ts`).

**`@/*` path alias** maps to `src/*` (see `tsconfig.json`) — used throughout instead of relative imports.

**Header/Footer** (`src/Header/`, `src/Footer/`) are Payload globals, not collections — each has a `config.ts` (schema), a server `Component.tsx`, and for Header a `Component.client.tsx` for theme-aware client behavior.

## Deployment

`.do/app.yaml` is the DigitalOcean App Platform spec (`web` service + `migrate` pre-deploy job + dev-tier Postgres), deployed via App Platform's Node.js buildpack (no Dockerfile) — see README's "Deploying to DigitalOcean App Platform" for the full bootstrap walkthrough. Two DO-specific gotchas this project works around: (1) App Platform's **build** step never has database access (any scope/buildpack-vs-Dockerfile) but the **run** step does, so the `web` service splits its build across Next's experimental two-phase build mode — `build_command: pnpm build:compile` (no DB) and `run_command: pnpm build:generate && pnpm start` (DB-dependent static generation, run once at container start before serving traffic) — restoring real `generateStaticParams`/static rendering on the frontend routes instead of the on-demand-only fallback the stock template would otherwise need; see README's "A platform limitation worth knowing"; (2) DO's managed/dev Postgres requires TLS with a DO-signed CA, which `payload.config.ts`'s `postgresAdapter` trusts via the bindable `DATABASE_CA_CERT` (`${pt-payload-poc.CA_CERT}`) env var, absent/unused in local dev. `Dockerfile`/`Dockerfile.migrate` still exist for local production-build testing only (see README "Testing a production build with Docker") and are unrelated to the actual deploy path.
