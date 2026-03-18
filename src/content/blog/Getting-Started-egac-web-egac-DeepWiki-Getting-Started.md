---
title: "Getting Started"
description: "This page documents how to set up the platform for local development"
pubDate: 2026-03-18
---
# Getting Started
Relevant source files
- [astro.config.mjs](https://github.com/egac-web/egac/blob/0ba542fc/astro.config.mjs)
- [db/migrations/0001_initial_schema.sql](https://github.com/egac-web/egac/blob/0ba542fc/db/migrations/0001_initial_schema.sql)
- [db/seed.sql](https://github.com/egac-web/egac/blob/0ba542fc/db/seed.sql)
- [package.json](https://github.com/egac-web/egac/blob/0ba542fc/package.json)
- [src/layouts/Base.astro](https://github.com/egac-web/egac/blob/0ba542fc/src/layouts/Base.astro)
- [src/styles/tailwind.css](https://github.com/egac-web/egac/blob/0ba542fc/src/styles/tailwind.css)
- [tests/unit/db.test.ts](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/db.test.ts)
- [tsconfig.json](https://github.com/egac-web/egac/blob/0ba542fc/tsconfig.json)
- [vitest.config.ts](https://github.com/egac-web/egac/blob/0ba542fc/vitest.config.ts)
- [wrangler.toml](https://github.com/egac-web/egac/blob/0ba542fc/wrangler.toml)

This page provides an overview of setting up the EGAC platform for local development. It covers the prerequisites, initial setup steps, and common development workflows. The platform is an Astro-based application deployed to Cloudflare Pages with D1 (SQLite) database and KV storage.

For detailed step-by-step instructions on specific setup tasks, see:

- [Installation & Setup](/egac-web/egac/2.1-installation-and-setup) for installing dependencies and configuring the environment
- [Database Setup](/egac-web/egac/2.2-database-setup) for initializing the D1 database and running migrations
- [Configuration & Secrets](/egac-web/egac/2.3-configuration-and-secrets) for environment variables and secrets management

---

## Prerequisites

The following must be installed before beginning setup:
RequirementMinimum VersionPurpose**Node.js**18.0.0JavaScript runtime (specified in [package.json45-47](https://github.com/egac-web/egac/blob/0ba542fc/package.json#L45-L47))**npm**Latest LTSPackage manager**Wrangler CLI**3.67.0+Cloudflare development tool (in [package.json43](https://github.com/egac-web/egac/blob/0ba542fc/package.json#L43-L43))**Git**Any recentVersion control
The platform requires a Cloudflare account for production deployment, but local development uses Wrangler's local mode with Miniflare for simulating Cloudflare bindings.

**Sources:**[package.json45-47](https://github.com/egac-web/egac/blob/0ba542fc/package.json#L45-L47)

---

## Local Development Architecture

The development environment runs Astro's dev server with Cloudflare adapter integration, which provides access to D1, KV, and environment bindings through Wrangler's platform proxy.

### Development Environment Diagram

```
Local Machine

Source Code

Wrangler Local Mode

Astro Dev Server (npm run dev)

http://localhost:4321

runtime.env.DB

runtime.env.KV

Developer

astro dev
Port 4321

@astrojs/cloudflare adapter
platformProxy.enabled = true

Miniflare Runtime
Simulates Workers environment

Local D1 Database
.wrangler/state/v3/d1/

Local KV Store
.wrangler/state/v3/kv/

src/pages/**/*.astro
Public & Admin routes

src/pages/api/**/*.ts
API endpoints

src/lib/**/*.ts
Business logic & repos
```

The `platformProxy` configuration in [astro.config.mjs9-12](https://github.com/egac-web/egac/blob/0ba542fc/astro.config.mjs#L9-L12) enables the Cloudflare adapter to provide `Astro.locals.runtime.env` with access to `DB`, `KV`, and other bindings during local development.

**Sources:**[astro.config.mjs1-14](https://github.com/egac-web/egac/blob/0ba542fc/astro.config.mjs#L1-L14)[package.json8](https://github.com/egac-web/egac/blob/0ba542fc/package.json#L8-L8)

---

## Quick Start Overview

The following steps initialize a working local development environment. Each step references a detailed child page for complete instructions.

### 1. Clone and Install Dependencies

```
git clone https://github.com/egac-web/egac.git
cd egac
npm install
```

This installs all dependencies from [package.json22-43](https://github.com/egac-web/egac/blob/0ba542fc/package.json#L22-L43) including Astro, Cloudflare adapter, Wrangler, and testing tools.

### 2. Initialize Local Database

```
npm run migrate:local
npm run migrate:seed
```

These commands execute:

- `migrate:local` runs [db/migrations/0001_initial_schema.sql](https://github.com/egac-web/egac/blob/0ba542fc/db/migrations/0001_initial_schema.sql) against the local D1 database
- `migrate:seed` populates [db/seed.sql](https://github.com/egac-web/egac/blob/0ba542fc/db/seed.sql) with default age groups and email templates

See [Database Setup](/egac-web/egac/2.2-database-setup) for details on the schema and seeding process.

### 3. Configure Environment Variables

Create a `.dev.vars` file in the project root with required secrets:

```
RESEND_API_KEY=re_...
ADMIN_TOKEN=your_secure_token
CRON_SECRET=your_cron_secret
ADMIN_EMAIL=admin@example.com

```

Non-secret configuration is defined in [wrangler.toml22-24](https://github.com/egac-web/egac/blob/0ba542fc/wrangler.toml#L22-L24) See [Configuration & Secrets](/egac-web/egac/2.3-configuration-and-secrets) for the complete list of required variables.

### 4. Start Development Server

```
npm run dev
```

This starts the Astro dev server at `http://localhost:4321` with hot module replacement enabled.

**Sources:**[package.json7-20](https://github.com/egac-web/egac/blob/0ba542fc/package.json#L7-L20)[wrangler.toml1-49](https://github.com/egac-web/egac/blob/0ba542fc/wrangler.toml#L1-L49)

---

## Development Workflow

The platform provides npm scripts for common development tasks. The following diagram shows the typical development cycle and available commands.

### Development Commands Workflow

```
Build & Preview

Database Operations

Testing

Code Quality Checks

Development Cycle

Before commit

Before commit

Before commit

DB changes

Ready to deploy

Start Development

npm run dev
Astro dev server + HMR

Edit .astro, .ts files

Browser auto-refreshes
localhost:4321

npm run type-check
tsc --noEmit

npm run lint
eslint src tests

npm run format
prettier --write

npm run format:check
prettier --check

npm run test
vitest run

npm run test:watch
vitest (watch mode)

npm run test:coverage
vitest + coverage report

npm run migrate:local
wrangler d1 execute --local

npm run migrate:seed
Load seed.sql

npm run migrate:staging
wrangler d1 execute --env staging

npm run build
astro build → dist/

npm run preview
Preview production build
```

**Sources:**[package.json7-20](https://github.com/egac-web/egac/blob/0ba542fc/package.json#L7-L20)

---

## Project Structure

The codebase follows a layered architecture with clear separation between presentation, API, business logic, and data access.
DirectoryPurpose`src/pages/`Astro pages (public routes at `/`, admin at `/admin/*`)`src/pages/api/`API route handlers (`/api/enquiry`, `/api/admin/*`, `/api/cron/*`)`src/lib/logic/`Business logic modules (age resolution, enquiry routing, token generation)`src/lib/db/repositories/`Data access layer (enquiries, invites, bookings, academy)`src/lib/db/`Database client, schema types, query utilities`src/lib/email/`Email templates, rendering, and sending logic`src/layouts/`Astro layout components (`Base.astro`, `AdminBase.astro`)`src/styles/`Tailwind CSS imports`db/migrations/`SQL schema migrations`db/`Seed data SQL files`tests/unit/`Unit tests (business logic, utilities)`tests/integration/`Integration tests (API endpoints, repositories)
The entry point for all runtime configuration is [wrangler.toml](https://github.com/egac-web/egac/blob/0ba542fc/wrangler.toml) which defines bindings for `DB` (D1), `KV` (KV namespace), and environment variables.

**Sources:**[wrangler.toml1-49](https://github.com/egac-web/egac/blob/0ba542fc/wrangler.toml#L1-L49)[astro.config.mjs1-14](https://github.com/egac-web/egac/blob/0ba542fc/astro.config.mjs#L1-L14)

---

## Bindings and Runtime Environment

The Cloudflare runtime provides access to platform services through bindings. These are accessed in code via `Astro.locals.runtime.env` in Astro pages and API routes.

### Runtime Bindings Reference

```
Secrets (.dev.vars / wrangler secret)

Environment Variables (wrangler.toml vars)

Bindings (wrangler.toml)

Runtime Context

Astro.locals.runtime.env

DB: D1Database
binding = 'DB'
database_name = 'egac'

KV: KVNamespace
binding = 'KV'
id = '28ecc4df...'

APP_ENV = 'production'

EMAIL_FROM = 'no-reply@...'

RESEND_API_KEY

ADMIN_TOKEN

CRON_SECRET
```

All bindings are type-safe via the `Env` interface defined in the codebase. The `getDB()` helper validates the presence of the `DB` binding at runtime ([tests/unit/db.test.ts112-122](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/db.test.ts#L112-L122)).

**Sources:**[wrangler.toml22-39](https://github.com/egac-web/egac/blob/0ba542fc/wrangler.toml#L22-L39)[tests/unit/db.test.ts112-135](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/db.test.ts#L112-L135)

---

## Verification Steps

After completing setup, verify the environment is working correctly:

### 1. Database Connectivity

```
wrangler d1 execute egac --local --command "SELECT COUNT(*) FROM age_groups"
```

Expected output: 5 rows (U11, U13, U15, U17, U20 age groups from [db/seed.sql12-53](https://github.com/egac-web/egac/blob/0ba542fc/db/seed.sql#L12-L53))

### 2. Development Server

Start the dev server and navigate to `http://localhost:4321`. You should see the public enquiry form.

### 3. Admin Interface

Navigate to `http://localhost:4321/admin` and authenticate using the `ADMIN_TOKEN` from your `.dev.vars` file.

### 4. Run Tests

```
npm run test
```

All tests in `tests/unit/` and `tests/integration/` should pass. Test configuration is in [vitest.config.ts1-24](https://github.com/egac-web/egac/blob/0ba542fc/vitest.config.ts#L1-L24)

**Sources:**[package.json18-20](https://github.com/egac-web/egac/blob/0ba542fc/package.json#L18-L20)[db/seed.sql12-53](https://github.com/egac-web/egac/blob/0ba542fc/db/seed.sql#L12-L53)[vitest.config.ts1-24](https://github.com/egac-web/egac/blob/0ba542fc/vitest.config.ts#L1-L24)

---

## TypeScript Configuration

The project uses strict TypeScript with ESM module resolution. Key settings:
SettingValuePurpose`target`ES2022Modern JavaScript features`module`ESNextES modules with top-level await`moduleResolution`bundlerAstro/Vite bundler resolution`strict`trueAll strict type checking enabled`types``@cloudflare/workers-types`Cloudflare Workers API types
All source files must use `.ts` or `.astro` extensions. When importing TypeScript files, use `.js` extension in import paths (ESM convention): `import { x } from './module.js'`. The resolver maps this to `module.ts` ([vitest.config.ts18-23](https://github.com/egac-web/egac/blob/0ba542fc/vitest.config.ts#L18-L23)).

**Sources:**[tsconfig.json1-19](https://github.com/egac-web/egac/blob/0ba542fc/tsconfig.json#L1-L19)[vitest.config.ts18-23](https://github.com/egac-web/egac/blob/0ba542fc/vitest.config.ts#L18-L23)

---

## Next Steps

Once your local environment is running:

1. Review the [System Architecture](/egac-web/egac/1.1-system-architecture) to understand component relationships
2. Explore the [Data Model](/egac-web/egac/1.2-data-model) to understand database entities
3. Read [Public User Interface](/egac-web/egac/3-public-user-interface) to understand the enquiry and booking flow
4. Review [API Reference](/egac-web/egac/5-api-reference) before building features

For production deployment instructions, see [Deployment](/egac-web/egac/11.3-deployment).

**Sources:**[package.json1-48](https://github.com/egac-web/egac/blob/0ba542fc/package.json#L1-L48)[wrangler.toml1-49](https://github.com/egac-web/egac/blob/0ba542fc/wrangler.toml#L1-L49)
