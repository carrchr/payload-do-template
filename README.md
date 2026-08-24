# Project Overview (and Payload Website Template)

This is a "Proof of Concept" for deploying to Digital Ocean using the official [Payload Website Template](https://github.com/payloadcms/payload/blob/3.x/templates/website).

## Local Development

### Use Docker

Run the Payload/Next.js app and Postgres together via Docker instead of installing either locally:

1. Install Docker or compatible container application (eg [Orbstack](https://orbstack.dev) or [Colima](https://colima.run/))
1. `cp .env.example .env` — `docker-compose.yml` picks it up automatically
1. `docker compose up`
1. Open `http://localhost:3000`

Dependencies will install into a named Docker volume, not your local `node_modules`.  
**NOTE!!:** prepend node/pnpm commands with `docker compose exec payload`, e.g. `docker compose exec payload pnpm add some-package`.

To get full TypeScript support in your editor/IDE, you'll want to install node dependencies locally, too.
* Install Node.js `v22.23.1` (or 22.something) - it's recommended to use a version manager (`nodenv` or `nvm`, `asdf`, etc.) 
* `npm i -g pnpm` and `pnpm install` 
* unfortunately the `node_modules` don't stay "in sync" automatically with those shared via Docker

### Setup locally WITHOUT Docker

* Install Postgres 17 - [Download Postgresql](https://www.enterprisedb.com/downloads/postgres-postgresql-downloads) or `brew install postgresql@17` and `brew services start postgresql@17`
* Create a user: `createuser --pwprompt -S -D -R postgres` then enter password from `.env`
* Create the database: `createdb pt-payload-poc -O postgres`
* Install Node.js and `pnpm` as above
* Run `pnpm dev` to start Payload/Next.js on `localhost:3000`

### Working with Postgres

Postgres and other SQL-based databases follow a strict schema for managing your data. In comparison to our MongoDB adapter, this means that there's a few extra steps to working with Postgres.

Note that often times when making big schema changes you can run the risk of losing data if you're not manually migrating it.

#### Local Development

Ideally we recommend running a local copy of your database so that schema updates are as fast as possible. By default the Postgres adapter has `push: true` for development environments. This will let you add, modify and remove fields and collections without needing to run any data migrations.

If your database is pointed to production you will want to set `push: false` otherwise you will risk losing data or having your migrations out of sync.

- **Local dev**: `push: true` (default) auto-syncs schema on save — no migrations needed.
- **Production**: `push: false` — use migrations instead, or risk losing data on schema changes.

**Troubleshooting:** if your local server seems broke, first try `rm -rf` `node_modules` and/or `.next`

#### Migrations

[Migrations](https://payloadcms.com/docs/database/migrations) are essentially SQL code versions that keeps track of your schema. When deploy with Postgres you will need to make sure you create and then run your migrations.

Locally create a migration

```bash
pnpm payload migrate:create   # after changing collection/field shape, locally
pnpm payload migrate          # apply pending migrations — run before `pnpm start` in production
```

This creates the migration files you will need to push alongside with your new configuration. 

### Seed the template content

Click "seed database" in the admin panel to populate a few pages, posts, and a demo user:

- Email: `demo-author@payloadcms.com`
- Password: `password`

> This is destructive — it drops your current database before repopulating it. Only run it on a fresh project or data you can afford to lose.

## Production

1. `pnpm build` — production build (`.next` output)
1. `pnpm start` — serve the build
1. See [Deployment](#deployment) when you're ready to go live

## Deployment

### Self-hosting

Deploy Payload like any other Node.js/Next.js app — a VPS, [DigitalOcean App Platform](#deploying-to-digitalocean-app-platform) (see below), Coolify, etc. See the official [deployment docs](https://payloadcms.com/docs/production/deployment) for details beyond DO.

### Object storage (DO Spaces)

Media uploads go to a [DigitalOcean Spaces](https://www.digitalocean.com/products/spaces) bucket in production, via `@payloadcms/storage-s3` in `src/payload.config.ts` (only active when `NODE_ENV=production`). Locally, `Media`'s `staticDir` serves uploads from disk instead, so dev doesn't need Spaces credentials.

1. Install `doctl`: <https://docs.digitalocean.com/reference/doctl/how-to/install/>
1. Create a bucket: <https://cloud.digitalocean.com/spaces/new> (enable CDN for production)
1. Create access keys: `doctl spaces keys create <key-name> --grants 'bucket=<bucket-name>;permission=readwrite'`
1. Set `SPACES_KEY_ID`, `SPACES_SECRET_KEY`, `SPACES_BUCKET_NAME`, and `SPACES_REGION` (e.g. `nyc3`) — as app secrets in production (see [First deploy](#first-deploy-bootstrap)), and in `.env` locally only if you want to test against real Spaces.

> Changed the storage plugin config? Regenerate the admin import map (`pnpm generate:importmap`) and commit it. Payload's admin panel resolves plugin client components from this committed file, so one regenerated while a plugin was inactive will 404 once that plugin is active.

### Deploying to DigitalOcean App Platform

The app spec lives at `.do/app.yaml`: a `web` service and a `migrate` `PRE_DEPLOY` job (which runs `pnpm payload migrate` before each deploy), both built by App Platform's Node.js **buildpack** (no Dockerfile involved), plus an attached dev-tier Postgres database (`production: false` — cheap, single-node; `doctl apps propose` estimates ~$10/month total for this spec). Deploys are sourced from GitHub (`deploy_on_push: true`), so the DigitalOcean GitHub App needs access to this repo first (DO console → Settings → Apps → GitHub, or `https://cloud.digitalocean.com/apps/github/install`).

#### A platform limitation worth knowing

App Platform builds — buildpack or Dockerfile — never have database access. This is documented DO behavior, not a bug: see their [troubleshooting guide](https://docs.digitalocean.com/support/why-does-my-app-fail-to-build-while-trying-to-connect-to-a-digitalocean-managed-database/), which recommends against connecting to a database during build at all. The stock Payload website template queries Postgres at build time in several places — `generateStaticParams` in `[slug]/page.tsx`, `posts/[slug]/page.tsx`, and `posts/page/[pageNumber]/page.tsx` to pre-render every page/post path, plus `force-static`/default-static rendering of the root `/` page, `/posts`, and the two sitemap route handlers — all of which fail outright on App Platform. Those were all changed to skip the build-time query (`generateStaticParams` returns `[]`; the others use `export const dynamic = 'force-dynamic'`) so pages render on first request instead. Next's `dynamicParams` (default `true`) handles the `generateStaticParams` routes, and the existing `afterChange` revalidation hooks (`revalidatePage.ts`) keep everything fresh exactly as before. The only build-time env var this project still needs is `NEXT_PUBLIC_SERVER_URL` (inlined into the client bundle), which uses the bindable `${APP_URL}`.

DO's managed/dev Postgres also always requires TLS, using a certificate signed by DO's own CA rather than a publicly trusted one. node-postgres rejects that by default (`self-signed certificate in certificate chain` / `SELF_SIGNED_CERT_IN_CHAIN`) unless told to trust it explicitly, so `.do/app.yaml` also binds `DATABASE_CA_CERT: ${pt-payload-poc.CA_CERT}`, which `payload.config.ts`'s `postgresAdapter` passes as `pool.ssl.ca` when present (it's absent for local dev, which doesn't use TLS at all).

#### First deploy (bootstrap)

1. Fill in the `REPLACE_ME` secrets in a local, uncommitted copy — never commit real values into `.do/app.yaml`: `PAYLOAD_SECRET`, `CRON_SECRET`, `PREVIEW_SECRET`, `SPACES_KEY_ID`, `SPACES_SECRET_KEY`, `SPACES_BUCKET_NAME`, `SPACES_REGION`.
2. Create the app: `doctl apps create --spec .do/app.yaml`.

That's it — `NEXT_PUBLIC_SERVER_URL` resolves once the app has a URL, including on this first deploy, and `DATABASE_URL` is runtime-only. After that, `deploy_on_push` handles it — every push to `main` rebuilds and redeploys, running the `migrate` job first, same as [Migrations](#migrations) locally.

### Generate & update secrets

Use `openssl rand -base64 32` — use to generate `PAYLOAD_SECRET`, `CRON_SECRET`, and `PREVIEW_SECRET`.

### Testing a production build with Docker

`Dockerfile`, `Dockerfile.migrate`, and `docker-compose.prod.yml` are **not** used for the actual DigitalOcean deploy above (that's a buildpack build) — they're kept only as a local way to sanity-check a production build/start cycle. `docker-compose.prod.yml` builds and runs `Dockerfile` against its own isolated Postgres container and volume, separate from the one used by `docker-compose.yml` for local development, so it won't interfere with your dev database.

The production build queries Postgres at build time (to pre-render paginated post pages), and `NODE_ENV=production` disables Payload's `push` schema sync, so the database needs schema in it via migrations before you build:

1. Start just the Postgres service: `docker compose -f docker-compose.prod.yml up -d postgres`
1. Run your migrations against it (see [Migrations](#migrations)): `pnpm payload migrate`
1. Build and start the full stack: `docker compose -f docker-compose.prod.yml up --build`

The app will be available at `http://localhost:3000`, same as the dev stack. To tear everything down, including the isolated database volume: `docker compose -f docker-compose.prod.yml down -v`.
