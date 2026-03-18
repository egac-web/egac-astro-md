---
title: "Installation and Setup"
description: "This page documents step-by-step instructions for installing dependencies and initialising the platform"
pubDate: 2026-03-18
---
# Installation & Setup
Relevant source files
- [astro.config.mjs](https://github.com/egac-web/egac/blob/0ba542fc/astro.config.mjs)
- [package-lock.json](https://github.com/egac-web/egac/blob/0ba542fc/package-lock.json)
- [package.json](https://github.com/egac-web/egac/blob/0ba542fc/package.json)
- [src/layouts/Base.astro](https://github.com/egac-web/egac/blob/0ba542fc/src/layouts/Base.astro)
- [src/styles/tailwind.css](https://github.com/egac-web/egac/blob/0ba542fc/src/styles/tailwind.css)
- [tsconfig.json](https://github.com/egac-web/egac/blob/0ba542fc/tsconfig.json)

This document provides step-by-step instructions for installing dependencies, configuring the local development environment, and initializing the database for the EGAC platform. It covers the initial setup required to run the application locally on your development machine.

For information about database schema and migrations, see [Database Setup](/egac-web/egac/2.2-database-setup). For details on environment variables and secrets management, see [Configuration & Secrets](/egac-web/egac/2.3-configuration-and-secrets).

---

## Purpose and Scope

This page guides developers through the complete local setup process, from cloning the repository to running the development server. By the end of this guide, you will have:

- All npm dependencies installed
- The Astro development server configured
- The local D1 database initialized with schema and seed data
- The Cloudflare platform proxy enabled for local runtime emulation

---

## Prerequisites

The following software must be installed on your development machine before proceeding:
RequirementMinimum VersionPurpose**Node.js**`18.0.0`JavaScript runtime for Astro and tooling**npm**`8.0.0+`Package manager (included with Node.js)**Git**Any recent versionVersion control for cloning repository
**Optional but Recommended:**

- **Wrangler CLI** (installed automatically as dev dependency) — Cloudflare's command-line tool for D1 database management and local emulation

**Sources:**[package.json45-47](https://github.com/egac-web/egac/blob/0ba542fc/package.json#L45-L47)

---

## Installation Steps

### Step 1: Clone the Repository

```
git clone https://github.com/egac-web/egac.git
cd egac
```

The repository contains the complete Astro Pages application, including:

- `/src` — Application source code
- `/db` — Database migrations and seed files
- Configuration files (astro.config.mjs, tsconfig.json, package.json)

### Step 2: Install Dependencies

Install all required npm packages using the lockfile to ensure reproducible builds:

```
npm install
```

This command installs dependencies defined in [package.json22-43](https://github.com/egac-web/egac/blob/0ba542fc/package.json#L22-L43) including:

**Production Dependencies:**

- `@astrojs/cloudflare` — Cloudflare Pages adapter for Astro
- `astro` — Static site framework with SSR capabilities
- `date-fns` — Date utility library

**Development Dependencies:**

- `@astrojs/tailwind` — Tailwind CSS integration
- `@cloudflare/workers-types` — TypeScript types for Cloudflare Workers runtime
- `wrangler` — Cloudflare CLI for D1 database and local development
- `vitest` — Unit testing framework
- `typescript`, `eslint`, `prettier` — Code quality tooling

The installed `node_modules/` directory will be approximately 500MB in size.

**Sources:**[package.json22-43](https://github.com/egac-web/egac/blob/0ba542fc/package.json#L22-L43)

---

### Step 3: Configure Environment

No environment configuration is required for basic local development. The Astro configuration at [astro.config.mjs1-14](https://github.com/egac-web/egac/blob/0ba542fc/astro.config.mjs#L1-L14) enables the Cloudflare platform proxy automatically:

```
adapter: cloudflare({
  platformProxy: {
    enabled: true,
  },
})
```

This setting allows the development server to emulate Cloudflare's runtime environment, providing access to:

- **D1 database bindings** — Local SQLite instance
- **KV namespace bindings** — In-memory key-value store
- **Runtime context** — `Astro.locals.runtime.env`

For production environment variables and secrets (Resend API keys, admin tokens), see [Configuration & Secrets](/egac-web/egac/2.3-configuration-and-secrets).

**Sources:**[astro.config.mjs6-13](https://github.com/egac-web/egac/blob/0ba542fc/astro.config.mjs#L6-L13)

---

### Step 4: Initialize Database

The application requires a local D1 database with the schema and initial data. Execute the following commands in sequence:

#### 4a. Create Migration

Run the initial schema migration to create all database tables:

```
npm run migrate:local
```

This executes `wrangler d1 execute egac --local --file=db/migrations/0001_initial_schema.sql`, which creates the following tables:

- `enquiries` — User registration submissions
- `invites` — Booking invitation tokens
- `bookings` — Taster session reservations
- `age_groups` — Age classification rules
- `academy_waitlist` — Academy enrollment queue
- `academy_seasons` — Academy term definitions
- `email_templates` — Transactional email content
- `membership_otps` — One-time membership invite codes

#### 4b. Seed Initial Data

Populate the database with default age groups and email templates:

```
npm run migrate:seed
```

This command loads seed data from `db/seed.sql`, which typically includes:

- Default age group configurations (U11, U13, U15, U17, U20, Seniors)
- Email template content (booking_invite, academy_waitlist, etc.)
- System configuration defaults

The local D1 database is stored in `.wrangler/state/v3/d1/` as a SQLite file.

**Sources:**[package.json18-19](https://github.com/egac-web/egac/blob/0ba542fc/package.json#L18-L19)

---

### Step 5: Start Development Server

Launch the Astro development server with hot module replacement:

```
npm run dev
```

Expected output:

```
🚀 astro v4.x.x started in XXXms

  ┃ Local    http://localhost:4321/
  ┃ Network  use --host to expose

  ┃ D1        (wrangler d1)
  ┃ KV        (wrangler kv)

```

The development server runs at `http://localhost:4321` and automatically:

- Reloads on file changes (`.astro`, `.ts`, `.css` files)
- Proxies requests to D1 and KV bindings via Wrangler
- Provides detailed error messages with stack traces

Press `Ctrl+C` to stop the server.

**Sources:**[package.json8](https://github.com/egac-web/egac/blob/0ba542fc/package.json#L8-L8)

---

## Development Environment Architecture

```
Developer Machine

Local Storage

Wrangler Platform Proxy

Astro Dev Server

SQLite Database
.wrangler/state/v3/d1/

Browser
http://localhost:4321

astro dev
(Vite SSR)

@astrojs/cloudflare
adapter

D1 Binding Proxy
runtime.env.DB

KV Binding Proxy
runtime.env.KV

In-Memory KV
(ephemeral)
```

**Diagram: Local Development Runtime Stack**

The `platformProxy.enabled` setting in [astro.config.mjs10-12](https://github.com/egac-web/egac/blob/0ba542fc/astro.config.mjs#L10-L12) bridges Astro's development server to Cloudflare's runtime APIs. When a request reaches an API route or page component, the adapter injects `runtime.env` into `Astro.locals`, providing direct access to D1 and KV bindings.

**Sources:**[astro.config.mjs6-14](https://github.com/egac-web/egac/blob/0ba542fc/astro.config.mjs#L6-L14)

---

## Setup Command Flow

```
Developer starts setup

git clone repository

npm install
(package.json dependencies)

npm run migrate:local
(wrangler d1 execute)

npm run migrate:seed
(wrangler d1 execute)

npm run dev
(astro dev)

node_modules/
~500MB installed

SQLite DB created
.wrangler/state/

8 tables populated
age_groups, templates

Server running
localhost:4321
```

**Diagram: Installation Command Sequence**

Each `npm run` command is defined in the `scripts` section of [package.json7-20](https://github.com/egac-web/egac/blob/0ba542fc/package.json#L7-L20) The `migrate:*` scripts use the Wrangler CLI to interact with the D1 database, while `dev` starts the Astro development server with automatic Cloudflare runtime emulation.

**Sources:**[package.json7-20](https://github.com/egac-web/egac/blob/0ba542fc/package.json#L7-L20)

---

## TypeScript Configuration

The project uses strict TypeScript settings defined in [tsconfig.json1-19](https://github.com/egac-web/egac/blob/0ba542fc/tsconfig.json#L1-L19) Key compiler options:
OptionValueImpact`target``ES2022`Modern JavaScript features (async/await, optional chaining)`moduleResolution``bundler`Astro/Vite-compatible module resolution`strict``true`Enables all strict type-checking options`types``@cloudflare/workers-types`, `vitest/globals`, `node`Provides type definitions for Cloudflare APIs and testing`noUnusedLocals``true`Catches unused variables (enforced by linter)
No additional TypeScript setup is required. The Astro development server automatically compiles `.ts` and `.astro` files using the configuration.

**Sources:**[tsconfig.json2-16](https://github.com/egac-web/egac/blob/0ba542fc/tsconfig.json#L2-L16)

---

## File Structure After Setup

After completing all installation steps, your project directory will contain:

```
egac/
├── node_modules/          # Installed npm packages (~500MB)
├── .wrangler/
│   └── state/
│       └── v3/
│           └── d1/        # Local SQLite database files
├── src/
│   ├── pages/             # Astro pages and API routes
│   ├── layouts/           # Base.astro, AdminBase.astro
│   ├── lib/               # Business logic, repositories, email
│   └── styles/            # tailwind.css
├── db/
│   ├── migrations/        # 0001_initial_schema.sql
│   └── seed.sql           # Default data
├── astro.config.mjs       # Astro + Cloudflare adapter config
├── package.json           # Dependencies and scripts
├── tsconfig.json          # TypeScript configuration
└── wrangler.toml          # Cloudflare D1/KV bindings (if exists)

```

The `.wrangler/` directory is automatically created by Wrangler and should be added to `.gitignore`. It contains local development artifacts that should not be committed to version control.

**Sources:**[astro.config.mjs1-14](https://github.com/egac-web/egac/blob/0ba542fc/astro.config.mjs#L1-L14)[tsconfig.json1-19](https://github.com/egac-web/egac/blob/0ba542fc/tsconfig.json#L1-L19)[package.json1-48](https://github.com/egac-web/egac/blob/0ba542fc/package.json#L1-L48)

---

## Verification Steps

After starting the development server with `npm run dev`, verify the setup by accessing these URLs:

### 1. Public Homepage

Navigate to `http://localhost:4321/` in your browser. You should see:

- EGAC header with navigation (defined in [src/layouts/Base.astro25-33](https://github.com/egac-web/egac/blob/0ba542fc/src/layouts/Base.astro#L25-L33))
- Public homepage content
- No console errors in browser DevTools

### 2. API Health Check

Make a test request to verify API routes are functioning:

```
curl http://localhost:4321/api/health
```

Expected response:

```
{
  "status": "ok",
  "timestamp": "2025-01-15T10:30:00.000Z"
}
```

### 3. Database Connectivity

Verify D1 database access by checking the Wrangler logs in the terminal running `npm run dev`. Look for:

```
[wrangler:inf] ⎔ Starting local server...
[wrangler:inf] ⎔ Ready on http://localhost:4321

```

No D1 connection errors should appear.

### 4. Tailwind CSS Loading

Confirm CSS is loading by inspecting elements on the homepage. The body element should have classes from [src/layouts/Base.astro24](https://github.com/egac-web/egac/blob/0ba542fc/src/layouts/Base.astro#L24-L24):

- `min-h-screen`
- `bg-gray-50`
- `text-gray-900`

If styles are missing, verify [src/styles/tailwind.css1-3](https://github.com/egac-web/egac/blob/0ba542fc/src/styles/tailwind.css#L1-L3) exists and is imported in the layout.

**Sources:**[src/layouts/Base.astro1-43](https://github.com/egac-web/egac/blob/0ba542fc/src/layouts/Base.astro#L1-L43)[src/styles/tailwind.css1-3](https://github.com/egac-web/egac/blob/0ba542fc/src/styles/tailwind.css#L1-L3)

---

## Common Issues and Solutions

### Issue: "Cannot find module '@astrojs/cloudflare'"

**Cause:** Dependencies not installed or incomplete installation.

**Solution:**

```
rm -rf node_modules package-lock.json
npm install
```

### Issue: "D1 database not found"

**Cause:** Migration scripts not executed.

**Solution:**

```
npm run migrate:local
npm run migrate:seed
```

Verify `.wrangler/state/v3/d1/` directory exists.

### Issue: "Port 4321 already in use"

**Cause:** Another process is using the default Astro port.

**Solution:**

```
# Kill existing process
lsof -ti:4321 | xargs kill -9
 
# Or start on different port
npm run dev -- --port 3000
```

### Issue: Node version mismatch

**Cause:** Node.js version below 18.0.0.

**Solution:**
Check current version:

```
node --version
```

If below v18.0.0, install Node 18+ using [nvm](https://github.com/egac-web/egac/blob/0ba542fc/nvm) or download from [nodejs.org](https://nodejs.org/).

**Sources:**[package.json45-47](https://github.com/egac-web/egac/blob/0ba542fc/package.json#L45-L47)

---

## Available npm Scripts

The following commands are available after installation (defined in [package.json7-20](https://github.com/egac-web/egac/blob/0ba542fc/package.json#L7-L20)):
CommandPurpose`npm run dev`Start development server at `localhost:4321``npm run build`Build production bundle for deployment`npm run preview`Preview production build locally`npm test`Run unit tests with Vitest`npm run test:watch`Run tests in watch mode`npm run test:coverage`Generate test coverage report`npm run lint`Run ESLint on source files`npm run format`Format code with Prettier`npm run type-check`Run TypeScript compiler without emitting files`npm run migrate:local`Apply database migrations to local D1`npm run migrate:seed`Seed local database with initial data`npm run migrate:staging`Apply migrations to staging environment
**Sources:**[package.json7-20](https://github.com/egac-web/egac/blob/0ba542fc/package.json#L7-L20)

---

## Next Steps

With the development environment set up, proceed to:

- **[Database Setup](/egac-web/egac/2.2-database-setup)** — Learn about the database schema, migration system, and entity relationships
- **[Configuration & Secrets](/egac-web/egac/2.3-configuration-and-secrets)** — Configure environment variables for Resend API, admin tokens, and multi-environment settings
- **[Local Development](/egac-web/egac/11.2-local-development)** — Explore development workflows, debugging techniques, and testing patterns
- **[Public User Interface](/egac-web/egac/3-public-user-interface)** — Understand the enquiry form, booking flow, and academy waitlist features

To begin exploring the codebase, review:

- Public pages in `src/pages/` directory
- API routes in `src/pages/api/` directory
- Shared layouts in `src/layouts/` directory ([Base.astro1-43](https://github.com/egac-web/egac/blob/0ba542fc/Base.astro#L1-L43))

**Sources:**[src/layouts/Base.astro1-43](https://github.com/egac-web/egac/blob/0ba542fc/src/layouts/Base.astro#L1-L43)
