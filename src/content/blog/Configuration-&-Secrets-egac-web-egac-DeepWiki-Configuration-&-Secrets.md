---
title: "Configuration and secrets"
description: "This page documents the configuration and secrets management system"
pubDate: 2026-03-18
---
# Configuration & Secrets
Relevant source files
- [db/migrations/0001_initial_schema.sql](https://github.com/egac-web/egac/blob/0ba542fc/db/migrations/0001_initial_schema.sql)
- [db/seed.sql](https://github.com/egac-web/egac/blob/0ba542fc/db/seed.sql)
- [src/env.d.ts](https://github.com/egac-web/egac/blob/0ba542fc/src/env.d.ts)
- [src/pages/admin/academy.astro](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/academy.astro)
- [src/pages/admin/settings.astro](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/settings.astro)
- [tests/unit/db.test.ts](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/db.test.ts)
- [vitest.config.ts](https://github.com/egac-web/egac/blob/0ba542fc/vitest.config.ts)
- [wrangler.toml](https://github.com/egac-web/egac/blob/0ba542fc/wrangler.toml)

This page documents the configuration and secrets management system for the EGAC platform. It covers environment variables, secret storage, runtime bindings, and the system configuration stored in Cloudflare KV.

For information about database schema setup and migrations, see [Database Setup](/egac-web/egac/2.2-database-setup). For type definitions related to the runtime environment, see [Environment Type Definitions](/egac-web/egac/10.1-environment-type-definitions).

---

## Configuration Architecture

The EGAC platform uses a three-tier configuration system:

1. **Build-time configuration**: Static variables defined in `wrangler.toml`
2. **Runtime secrets**: Sensitive credentials managed via Wrangler CLI
3. **Dynamic system configuration**: User-editable settings stored in KV namespace

All configuration is accessible through the `Env` type and injected into request handlers via Cloudflare's runtime bindings.

**Sources:**[wrangler.toml1-49](https://github.com/egac-web/egac/blob/0ba542fc/wrangler.toml#L1-L49)[src/env.d.ts5-11](https://github.com/egac-web/egac/blob/0ba542fc/src/env.d.ts#L5-L11)

---

## Environment Variables and Bindings

### Wrangler Configuration File

The `wrangler.toml` file defines the project metadata, bindings, and non-secret environment variables:
ConfigurationValuePurpose`name``"egac"`Cloudflare project identifier`compatibility_date``"2024-09-23"`Cloudflare Workers compatibility date`pages_build_output_dir``"dist"`Astro build output directory`APP_ENV``"production"`Environment identifier for data isolation`EMAIL_FROM``"no-reply@updates.eastgrinsteadac.co.uk"`Default sender address for emails
**Sources:**[wrangler.toml1-25](https://github.com/egac-web/egac/blob/0ba542fc/wrangler.toml#L1-L25)

### Resource Bindings

The platform binds to two Cloudflare resources:

```
Runtime Bindings

[[d1_databases]]

[[kv_namespaces]]

wrangler.toml
binding definitions

D1Database
binding: DB
database_id: 6a053384...

KVNamespace
binding: KV
id: 28ecc4df...

Env type
src/lib/db/schema.js
```

**D1 Database Binding**

- **Binding name**: `DB`
- **Database name**: `egac`
- **Database ID**: `6a053384-20ee-4fee-bf73-aa29c1779fe3`
- **Purpose**: Relational data storage (enquiries, bookings, age groups, etc.)

**KV Namespace Binding**

- **Binding name**: `KV`
- **Namespace ID**: `28ecc4df31b44e5e92cc4730c1870829`
- **Purpose**: System configuration storage with fast global access

**Sources:**[wrangler.toml26-40](https://github.com/egac-web/egac/blob/0ba542fc/wrangler.toml#L26-L40)

---

## Secrets Management

Three secrets must be configured via the Wrangler CLI. These are **never** committed to version control or defined in `wrangler.toml`.

### Required Secrets
SecretPurposeSet via`RESEND_API_KEY`Authentication for Resend email API`wrangler secret put RESEND_API_KEY``ADMIN_TOKEN`Authentication for admin interface and admin API endpoints`wrangler secret put ADMIN_TOKEN``CRON_SECRET`Authentication for cron job triggers`wrangler secret put CRON_SECRET`
### Setting Secrets

Secrets are set per project using the Wrangler CLI:

```
wrangler secret put RESEND_API_KEY
# Prompt: Enter a secret value:
# <paste your Resend API key>
 
wrangler secret put ADMIN_TOKEN
# <enter a secure random token>
 
wrangler secret put CRON_SECRET
# <enter a secure random token for cron authentication>
```

Secrets are stored securely in Cloudflare's infrastructure and injected at runtime as environment variables accessible via `Astro.locals.runtime.env` in Pages handlers.

**Sources:**[wrangler.toml17-21](https://github.com/egac-web/egac/blob/0ba542fc/wrangler.toml#L17-L21)[src/pages/admin/settings.astro13-15](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/settings.astro#L13-L15)

---

## Runtime Environment Type

The `Env` interface defines all available runtime bindings and secrets:

```
type parameter

Env interface (schema.js)

DB: D1Database

KV: KVNamespace

RESEND_API_KEY: string

ADMIN_TOKEN: string

ADMIN_EMAIL?: string

EMAIL_FROM: string

CRON_SECRET: string

APP_ENV: string

CloudflareRuntime
@astrojs/cloudflare

App.Locals
(Astro request context)

Env

Request handler
Astro.locals.runtime.env
```

The type definition in [src/env.d.ts5-11](https://github.com/egac-web/egac/blob/0ba542fc/src/env.d.ts#L5-L11) extends `App.Locals` with the Cloudflare runtime:

```
type CloudflareRuntime = import('@astrojs/cloudflare').Runtime<import('./lib/db/schema.js').Env>;
 
declare global {
  namespace App {
    interface Locals extends CloudflareRuntime {}
  }
}
```

This allows all Astro pages and API routes to access `Astro.locals.runtime.env.DB`, `Astro.locals.runtime.env.KV`, and all secrets.

**Sources:**[src/env.d.ts1-14](https://github.com/egac-web/egac/blob/0ba542fc/src/env.d.ts#L1-L14)[src/pages/admin/settings.astro13](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/settings.astro#L13-L13)

---

## System Configuration (KV-based)

System configuration is stored in the KV namespace under the key `config:system`. This configuration is user-editable through the admin interface and can be updated at runtime without redeploying the application.

### Configuration Schema

The `SystemConfig` interface defines all configurable system settings:
FieldTypeDefaultPurpose`weeks_ahead_booking``string``"8"`How many weeks ahead taster sessions can be booked`capacity_per_slot``string``"2"`Default capacity per taster session`academy_max_age``string``"10"`Maximum age for academy waitlist (on Aug 31)`site_name``string``"East Grinstead Athletics Club"`Club name used in emails and UI`site_url``string``"https://eastgrinsteadac.co.uk"`Base URL for generating links`email_from``string``"no-reply@updates.eastgrinsteadac.co.uk"`Default sender address`venue_name``string``"East Court"`Venue name for session confirmations`venue_address``string``"College Lane, East Grinstead, RH19 3LT"`Full venue address`admin_email``string``"enquiries@eastgrinsteadac.co.uk"`Admin notification recipient
**Sources:**[tests/unit/db.test.ts141-170](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/db.test.ts#L141-L170)

### Configuration Access Pattern

```
"CONFIG_DEFAULTS"
"KV namespace"
"getSystemConfig(env)"
"Request handler"
"CONFIG_DEFAULTS"
"KV namespace"
"getSystemConfig(env)"
"Request handler"
alt
[Config exists in KV]
[No config in KV]
Request config
KV.get('config:system')
JSON string
JSON.parse()
Merge with defaults
null
Use CONFIG_DEFAULTS
SystemConfig object
```

### Core Configuration Functions

The configuration API is defined in `src/lib/config.js`:

**`getSystemConfig(env: Env): Promise<SystemConfig>`**

- Retrieves configuration from KV namespace
- Merges stored values with `CONFIG_DEFAULTS`
- Falls back gracefully to defaults if KV read fails
- Returns a complete `SystemConfig` object

**`setSystemConfig(updates: Partial<SystemConfig>, env: Env): Promise<SystemConfig>`**

- Validates updates using `validateConfigUpdate()`
- Merges updates with current configuration
- Writes updated config to KV as JSON string
- Returns the complete updated configuration

**`validateConfigUpdate(updates: Partial<SystemConfig>): string[]`**

- Validates numeric ranges (capacity 1-10, weeks 1-16)
- Validates email format for `email_from` and `admin_email`
- Validates URL format for `site_url`
- Returns array of error messages (empty array = valid)

**`clearConfigCache(env: Env): Promise<void>`**

- Clears any cached configuration (currently no-op, reserved for future caching layer)

**Sources:**[src/pages/admin/settings.astro4-9](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/settings.astro#L4-L9)[tests/unit/db.test.ts19-26](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/db.test.ts#L19-L26)[tests/unit/db.test.ts237-274](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/db.test.ts#L237-L274)

### Typed Configuration Parsers

Helper functions convert string configuration to typed values:

```
getAcademyMaxAge(config: SystemConfig): number
// Parses academy_max_age string to integer, defaults to 10
 
getWeeksAhead(config: SystemConfig): number  
// Parses weeks_ahead_booking string to integer, defaults to 8
 
getCapacityPerSlot(config: SystemConfig): number
// Parses capacity_per_slot string to integer, defaults to 2
```

These parsers handle invalid string values gracefully by falling back to sensible defaults.

**Sources:**[tests/unit/db.test.ts280-300](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/db.test.ts#L280-L300)

---

## Configuration Validation

The `validateConfigUpdate` function enforces business rules on configuration changes:

### Validation Rules
FieldValidation`capacity_per_slot`Must be integer between 1 and 10`weeks_ahead_booking`Must be integer between 1 and 16`academy_max_age`Must be integer between 5 and 15`email_from`Must match email regex pattern`admin_email`Must match email regex pattern`site_url`Must start with `http://` or `https://``site_name`Must not be empty`venue_name`Must not be empty`venue_address`Must not be empty
If validation fails, the function returns an array of error messages. The admin settings page displays these errors to the user and prevents the update from being saved.

**Sources:**[tests/unit/db.test.ts237-274](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/db.test.ts#L237-L274)[src/pages/admin/settings.astro100-106](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/settings.astro#L100-L106)

---

## Environment Isolation

The `APP_ENV` variable enables multi-environment data isolation within a single D1 database. All database queries automatically filter by the current environment.

### Supported Environments

- `development` - Local development
- `staging` - Pre-production testing (if configured)
- `production` - Live production data

### Environment Resolution

The `getEnv(env: Env)` function returns the current environment:

```
export function getEnv(env: Env): string {
  return env.APP_ENV ?? 'production';
}
```

When `APP_ENV` is undefined, it defaults to `'production'` for safety.

### Environment-aware Queries

The `envQuery` helper automatically injects environment filtering into all database queries:

```
// All queries built with envQuery include:
// WHERE environment = ?
// with the current environment value as a parameter
```

This ensures development data never mixes with production data, even when using the same D1 database instance.

**Sources:**[tests/unit/db.test.ts123-136](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/db.test.ts#L123-L136)[tests/unit/db.test.ts305-327](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/db.test.ts#L305-L327)[wrangler.toml23](https://github.com/egac-web/egac/blob/0ba542fc/wrangler.toml#L23-L23)

---

## Admin Configuration Interface

The system settings page (`/admin/settings`) provides a UI for managing configuration values. Access requires admin authentication via cookie token.

### Configuration Sections

The settings interface is organized into four sections:

**1. Booking Defaults**

- `weeks_ahead_booking` - Number of weeks ahead for session availability
- `capacity_per_slot` - Maximum athletes per taster session
- `academy_max_age` - Maximum age for academy eligibility

**2. Site and Venue**

- `site_name` - Club name
- `site_url` - Base URL for generated links
- `venue_name` - Training venue name
- `venue_address` - Full venue address

**3. Email Settings**

- `email_from` - Sender address for all emails
- `admin_email` - Recipient for admin notifications

**4. Age Groups**

- Inline editing of age group parameters
- `label`, `booking_type`, `age_min_aug31`, `age_max_aug31`
- `session_days` (JSON array), `session_time`, `capacity_per_session`
- `sort_order` and `active` flag

**Sources:**[src/pages/admin/settings.astro158-228](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/settings.astro#L158-L228)[src/pages/admin/settings.astro230-291](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/settings.astro#L230-L291)

### Configuration Update Flow

```
"KV namespace"
"setSystemConfig"
"validateConfigUpdate"
"POST handler"
"Settings form"
"Admin user"
"KV namespace"
"setSystemConfig"
"validateConfigUpdate"
"POST handler"
"Settings form"
"Admin user"
alt
[Validation passes]
[Validation fails]
Submit changes
POST /admin/settings
validateConfigUpdate(updates)
[] (empty errors)
setSystemConfig(updates, env)
KV.put('config:system', JSON)
Success
Updated config
Show success message
["error1", "error2"]
Show error messages
```

The form submission at [src/pages/admin/settings.astro20-108](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/settings.astro#L20-L108) handles both system config updates (`action=config-save`) and age group updates (`action=age-group-save`).

**Sources:**[src/pages/admin/settings.astro20-108](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/settings.astro#L20-L108)[src/pages/admin/settings.astro124-141](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/settings.astro#L124-L141)

---

## Configuration in Email Templates

System configuration values are injected into email templates as variables:
Template VariableConfig Source`{{site_name}}``config.site_name``{{site_url}}``config.site_url``{{venue}}``config.venue_name``{{venue_address}}``config.venue_address`
This allows administrators to update branding and venue information without modifying email template HTML.

**Sources:**[db/seed.sql64-223](https://github.com/egac-web/egac/blob/0ba542fc/db/seed.sql#L64-L223)

---

## Development vs Production Configuration

### Local Development Setup

For local development with Wrangler, create a `.dev.vars` file (gitignored) with secrets:

```
RESEND_API_KEY=re_xxxxx
ADMIN_TOKEN=dev_admin_token
CRON_SECRET=dev_cron_secret

```

Set `APP_ENV=development` in wrangler.toml for the development environment or use environment-specific wrangler configs.

### Production Deployment

Production secrets are set via Wrangler CLI targeting the production project:

```
wrangler secret put RESEND_API_KEY --env production
wrangler secret put ADMIN_TOKEN --env production
wrangler secret put CRON_SECRET --env production
```

The `APP_ENV` variable in [wrangler.toml23](https://github.com/egac-web/egac/blob/0ba542fc/wrangler.toml#L23-L23) is set to `"production"` for the main deployment.

**Sources:**[wrangler.toml1-49](https://github.com/egac-web/egac/blob/0ba542fc/wrangler.toml#L1-L49)

---

## Summary Table: Configuration Sources
Configuration TypeStorageAccess MethodMutabilityProject metadata`wrangler.toml`Build-timeStaticBindings (D1, KV)`wrangler.toml`Runtime via `env`StaticNon-secret vars`wrangler.toml``[vars]`Runtime via `env`StaticSecretsWrangler secretsRuntime via `env`Static (via CLI)System configKV namespace`getSystemConfig(env)`Dynamic (via admin UI)Age groupsD1 databaseRepository functionsDynamic (via admin UI)Email templatesD1 database`getEmailTemplateByKey()`Dynamic (seeded, future UI TBD)
**Sources:**[wrangler.toml1-49](https://github.com/egac-web/egac/blob/0ba542fc/wrangler.toml#L1-L49)[src/pages/admin/settings.astro110-111](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/settings.astro#L110-L111)[db/seed.sql1-224](https://github.com/egac-web/egac/blob/0ba542fc/db/seed.sql#L1-L224)
