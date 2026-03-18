# Database Setup
Relevant source files
- [db/migrations/0001_initial_schema.sql](https://github.com/egac-web/egac/blob/0ba542fc/db/migrations/0001_initial_schema.sql)
- [db/seed.sql](https://github.com/egac-web/egac/blob/0ba542fc/db/seed.sql)
- [docs/snagging.md](https://github.com/egac-web/egac/blob/0ba542fc/docs/snagging.md)
- [package.json](https://github.com/egac-web/egac/blob/0ba542fc/package.json)
- [src/lib/repositories/age-groups.ts](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/repositories/age-groups.ts)
- [src/lib/repositories/bookings.ts](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/repositories/bookings.ts)
- [tests/unit/db.test.ts](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/db.test.ts)
- [tsconfig.json](https://github.com/egac-web/egac/blob/0ba542fc/tsconfig.json)
- [vitest.config.ts](https://github.com/egac-web/egac/blob/0ba542fc/vitest.config.ts)
- [wrangler.toml](https://github.com/egac-web/egac/blob/0ba542fc/wrangler.toml)

This document explains how to initialize and manage the D1 SQLite database for the EGAC platform. It covers database bindings, schema migrations, seeding initial data, and environment isolation patterns.

For information about the overall data model and entity relationships, see [Data Model](/egac-web/egac/1.2-data-model). For details on how repositories interact with the database, see [Repository Pattern](/egac-web/egac/7.1-repository-pattern).

---

## D1 Database Binding

The EGAC platform uses **Cloudflare D1**, a serverless SQLite database. The database binding is configured in `wrangler.toml` and exposed to the application runtime through the `Env` type.

### Binding Configuration

```
Application Code

Runtime Environment

wrangler.toml

Cloudflare binds

Access via

Used by

[[d1_databases]]
binding = 'DB'
database_name = 'egac'
database_id = '...'

Env interface
DB: D1Database

getDB(env)
returns D1Database

Repositories
enquiries.ts
bookings.ts
age-groups.ts
```

**Sources:**[wrangler.toml29-32](https://github.com/egac-web/egac/blob/0ba542fc/wrangler.toml#L29-L32)[src/lib/db/client.ts](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/db/client.ts)

The database binding is defined in three places:
LocationPurposeDatabase ID`wrangler.toml`Production binding`6a053384-20ee-4fee-bf73-aa29c1779fe3`Local developmentCreated via `wrangler d1 create`Auto-generatedStaging environmentSeparate D1 instanceConfigured per environment
The `getDB(env)` function retrieves the D1Database instance and throws a clear error if the binding is missing:

```
// From src/lib/db/client.ts
export function getDB(env: Env): D1Database {
  if (!env.DB) {
    throw new Error('D1 binding "DB" not found in environment');
  }
  return env.DB;
}
```

**Sources:**[tests/unit/db.test.ts111-122](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/db.test.ts#L111-L122)[src/lib/db/client.ts](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/db/client.ts)

---

## Schema Structure

The database schema is defined in a single migration file that creates eight tables with indexed foreign key relationships.

### Schema Overview Diagram

```
generates

creates

joins

classified_by

results_in

assigned_to

enrolled_in

belongs_to

receives

enquiries

TEXT

id

PK

TEXT

email

UK

TEXT

name

TEXT

dob

TEXT

age_group_id

FK

TEXT

invite_id

FK

INTEGER

processed

TEXT

events

TEXT

environment

invites

TEXT

id

PK

TEXT

token

UK

TEXT

enquiry_id

FK

TEXT

status

INTEGER

send_attempts

TEXT

environment

bookings

TEXT

id

PK

TEXT

enquiry_id

FK

TEXT

invite_id

FK

TEXT

age_group_id

FK

TEXT

session_date

TEXT

status

TEXT

environment

academy_waitlist

TEXT

id

PK

TEXT

enquiry_id

FK

TEXT

season_id

FK

TEXT

token

UK

INTEGER

position

TEXT

status

INTEGER

is_returning

TEXT

environment

age_groups

TEXT

id

PK

TEXT

code

UK

TEXT

booking_type

INTEGER

age_min_aug31

INTEGER

age_max_aug31

TEXT

session_days

INTEGER

capacity_per_session

academy_seasons

TEXT

id

PK

TEXT

code

UK

TEXT

age_group_id

FK

TEXT

start_date

TEXT

end_date

INTEGER

capacity

TEXT

status

membership_otps

TEXT

id

PK

TEXT

enquiry_id

FK

TEXT

token

UK

TEXT

environment

email_templates

TEXT

id

PK

TEXT

key

UK

TEXT

subject

TEXT

html

TEXT

variables
```

**Sources:**[db/migrations/0001_initial_schema.sql1-239](https://github.com/egac-web/egac/blob/0ba542fc/db/migrations/0001_initial_schema.sql#L1-L239)

### Table Descriptions
TablePurposeKey FieldsNotes`enquiries`Source of truth for all user submissions`id`, `email`, `age_group_id`, `events`Contains event sourcing JSON array`invites`Taster session booking invitations`id`, `token`, `status`, `send_attempts`14-day expiry enforced via cron`bookings`Confirmed taster sessions`id`, `session_date`, `status`, `age_group_id`Status: confirmed/attended/no_show/cancelled`age_groups`Age group configuration`id`, `code`, `booking_type`, `age_min_aug31`Single source of truth for routing logic`academy_waitlist`Academy season waiting list entries`id`, `position`, `is_returning`, `status`One row per enquiry per season`academy_seasons`Academy season definitions`id`, `start_date`, `end_date`, `capacity`Typically April–August annually`email_templates`Configurable email templates`key`, `html`, `variables`Supports `{{variable}}` placeholders`membership_otps`Membership form access tokens`token`, `enquiry_id`Generated after attendance marking
**Sources:**[db/migrations/0001_initial_schema.sql19-239](https://github.com/egac-web/egac/blob/0ba542fc/db/migrations/0001_initial_schema.sql#L19-L239)

### Environment Isolation

All data-bearing tables include an `environment` field with a default value of `'production'`. This enables multi-tenancy across development, staging, and production environments within a single D1 database instance.

```
-- Example from enquiries table
CREATE TABLE IF NOT EXISTS enquiries (
  id            TEXT PRIMARY KEY,
  -- ... other fields ...
  environment   TEXT DEFAULT 'production'
);
 
CREATE INDEX IF NOT EXISTS enquiries_env_idx ON enquiries(environment);
CREATE INDEX IF NOT EXISTS enquiries_email_env_idx ON enquiries(email, environment);
```

The `getEnv(env)` function returns the current environment string, which all repository queries automatically filter by:

```
// From src/lib/db/client.ts
export function getEnv(env: Env): string {
  return env.APP_ENV ?? 'production';
}
```

**Sources:**[db/migrations/0001_initial_schema.sql36-43](https://github.com/egac-web/egac/blob/0ba542fc/db/migrations/0001_initial_schema.sql#L36-L43)[src/lib/db/client.ts](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/db/client.ts)[tests/unit/db.test.ts124-135](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/db.test.ts#L124-L135)

---

## Running Migrations

The platform uses a single migration file that creates the complete schema. Migrations are executed using the `wrangler` CLI with the D1 `execute` command.

### Migration Commands

```
Target Databases

Migration Scripts (package.json)

Creates tables

Deploys schema

Deploys schema

npm run migrate:local
wrangler d1 execute egac
--local
--file=db/migrations/0001_initial_schema.sql

npm run migrate:staging
wrangler d1 execute egac-staging
--env staging
--file=db/migrations/0001_initial_schema.sql

Manual execution via wrangler
wrangler d1 execute egac
--file=db/migrations/0001_initial_schema.sql

Local SQLite
.wrangler/state/v3/d1

Cloudflare D1
egac-staging

Cloudflare D1
egac (production)
```

**Sources:**[package.json18-20](https://github.com/egac-web/egac/blob/0ba542fc/package.json#L18-L20)

### Step-by-Step Migration Process

#### 1. Local Development Migration

```
# Run the initial schema migration against local D1
npm run migrate:local
```

This executes the migration against a local SQLite database stored in `.wrangler/state/v3/d1/miniflare-D1DatabaseObject/`. The `--local` flag tells Wrangler to use the local development environment instead of deploying to Cloudflare.

**Sources:**[package.json18](https://github.com/egac-web/egac/blob/0ba542fc/package.json#L18-L18)

#### 2. Staging Environment Migration

```
# Deploy schema to staging D1 instance
npm run migrate:staging
```

This requires:

- A staging D1 database created via `wrangler d1 create egac-staging`
- A `[[d1_databases]]` binding in a `[env.staging]` section of `wrangler.toml`
- The `CLOUDFLARE_API_TOKEN` environment variable set for authentication

**Sources:**[package.json20](https://github.com/egac-web/egac/blob/0ba542fc/package.json#L20-L20)

#### 3. Production Migration

Production migrations must be executed manually to prevent accidental schema changes:

```
# Authenticate with Cloudflare
export CLOUDFLARE_API_TOKEN="your-token"
 
# Execute migration against production D1
wrangler d1 execute egac --file=db/migrations/0001_initial_schema.sql
```

**Important:** All schema DDL statements use `CREATE TABLE IF NOT EXISTS` and `CREATE INDEX IF NOT EXISTS`, making migrations idempotent. Re-running the migration is safe and will not destroy existing data.

**Sources:**[db/migrations/0001_initial_schema.sql22](https://github.com/egac-web/egac/blob/0ba542fc/db/migrations/0001_initial_schema.sql#L22-L22)[wrangler.toml29-32](https://github.com/egac-web/egac/blob/0ba542fc/wrangler.toml#L29-L32)

---

## Seeding Initial Data

After running migrations, the database must be seeded with initial configuration data for age groups and email templates. The seed file is idempotent and uses `INSERT OR IGNORE` to prevent duplicate entries.

### Seed Data Overview

```
Database Tables

db/seed.sql

INSERT OR IGNORE

INSERT OR IGNORE

Age Groups
ag_u11: waitlist
ag_u13: taster
ag_u15: taster
ag_u17: taster (inactive)
ag_u20: taster (inactive)

Email Templates
booking_invite
booking_confirmation
academy_waitlist
academy_invite
membership_invite
booking_reminder

age_groups table
5 default groups

email_templates table
6 default templates
```

**Sources:**[db/seed.sql1-224](https://github.com/egac-web/egac/blob/0ba542fc/db/seed.sql#L1-L224)

### Running the Seed Script

```
# Seed local development database
npm run migrate:seed
```

This executes `wrangler d1 execute egac --local --file=db/seed.sql`, which inserts:

- **5 age groups** with different configurations
- **6 email templates** with HTML and text content

**Sources:**[package.json19](https://github.com/egac-web/egac/blob/0ba542fc/package.json#L19-L19)

### Default Age Groups

The seed file creates these age groups based on UK Athletics age-on-31-August standards:
IDCodeLabelBooking TypeAge RangeSession DaysCapacityActive`ag_u11`u11U11 Academywaitlist9–10Saturday, Tuesday40 seasonNo`ag_u13`u13U13taster11–12Tuesday2 per sessionYes`ag_u15`u15U15 and oldertaster13–120Tuesday2 per sessionYes`ag_u17`u17U17taster15–16Tuesday2 per sessionNo`ag_u20`u20U20taster17–19Tuesday2 per sessionNo
**Sources:**[db/seed.sql12-53](https://github.com/egac-web/egac/blob/0ba542fc/db/seed.sql#L12-L53)

### Default Email Templates

Six email templates are seeded with pre-formatted HTML and text content:
KeyPurposeVariables`booking_invite`Sent when taster invite is created`contact_name`, `booking_url`, `available_dates_list``booking_confirmation`Sent after booking is created`contact_name`, `session_date`, `session_time`, `age_group_label`, `venue`, `add_to_calendar_url``academy_waitlist`Sent when added to academy waitlist`parent_name`, `child_name``academy_invite`Sent when academy place becomes available`parent_name`, `child_name`, `season_label`, `season_start`, `season_end`, `accept_url`, `decline_url``membership_invite`Sent after attending taster session`contact_name`, `membership_url``booking_reminder`Sent before upcoming taster session`contact_name`, `session_date`, `session_time`, `age_group_label`, `venue`
All templates support variable substitution using `{{variable_name}}` syntax.

**Sources:**[db/seed.sql59-223](https://github.com/egac-web/egac/blob/0ba542fc/db/seed.sql#L59-L223)

---

## Database Client Access Pattern

The application accesses the D1 database through a centralized client module that provides utility functions and ensures environment isolation.

### Client Module Architecture

```
Repositories

src/lib/db/client.ts

getDB(env): D1Database
Returns DB binding or throws

getEnv(env): string
Returns APP_ENV or 'production'

generateId(prefix): string
Creates prefixed random IDs

generateToken(bytes): string
Creates secure hex tokens

now(): string
Returns ISO 8601 timestamp

one<T>(stmt): Promise<T|null>
Execute and return single row

many<T>(stmt): Promise<T[]>
Execute and return all rows

run(stmt): Promise<void>
Execute with no return

enquiries.ts
insertEnquiry
getEnquiryByEmail

invites.ts
createInviteForEnquiry
markInviteSent

bookings.ts
createBooking
listUpcomingBookings

age-groups.ts
listAgeGroups
getAgeGroupById
```

**Sources:**[src/lib/db/client.ts](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/db/client.ts)[src/lib/repositories/enquiries.ts](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/repositories/enquiries.ts)[src/lib/repositories/invites.ts](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/repositories/invites.ts)[src/lib/repositories/bookings.ts](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/repositories/bookings.ts)[src/lib/repositories/age-groups.ts](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/repositories/age-groups.ts)

### Query Execution Helpers

The client provides three query execution helpers that wrap D1's prepared statement API:
FunctionReturn TypeUse CaseExample`one<T>()``Promise<T | null>`Single row queries (SELECT by ID)`getBookingById(id, env)``many<T>()``Promise<T[]>`Multi-row queries (SELECT lists)`listUpcomingBookings(weeks, env)``run()``Promise<void>`INSERT/UPDATE/DELETE`updateBookingStatus(id, status, env)`
All three helpers automatically handle D1's result structure and extract the relevant data.

**Sources:**[src/lib/db/client.ts](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/db/client.ts)[src/lib/repositories/bookings.ts9-36](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/repositories/bookings.ts#L9-L36)

### ID and Token Generation

The client provides cryptographically secure ID and token generation:

```
// Generate prefixed random IDs
const enquiryId = generateId('enq');  // Returns: "enq_a1b2c3d4e5f6..."
const bookingId = generateId('bkg');  // Returns: "bkg_x9y8z7w6v5u4..."
 
// Generate secure random tokens (default 24 bytes = 48 hex chars)
const inviteToken = generateToken();     // Returns: 48-char hex string
const shortToken = generateToken(12);    // Returns: 24-char hex string
```

IDs follow the pattern `{prefix}_{20_hex_chars}` and are generated using `crypto.randomUUID()` with timestamp mixing to ensure uniqueness.

**Sources:**[src/lib/db/client.ts](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/db/client.ts)[tests/unit/db.test.ts65-97](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/db.test.ts#L65-L97)

---

## Environment-Aware Queries

All repository functions that query or modify data must filter by the current environment. This pattern is enforced throughout the codebase using the `getEnv(env)` function.

### Example: Environment-Filtered Query

```
// From src/lib/repositories/bookings.ts
export async function listBookingsForDate(date: string, env: Env): Promise<Booking[]> {
  const db = getDB(env);
  const environment = getEnv(env);  // Returns 'development', 'staging', or 'production'
  
  return many<Booking>(
    db.prepare(
      'SELECT * FROM bookings WHERE session_date = ? AND environment = ? ORDER BY created_at ASC'
    ).bind(date, environment)
  );
}
```

This ensures that:

- Local development (`APP_ENV=development`) queries only development data
- Staging deployments (`APP_ENV=staging`) query only staging data
- Production (`APP_ENV=production`) queries only production data

**Sources:**[src/lib/repositories/bookings.ts80-90](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/repositories/bookings.ts#L80-L90)[tests/unit/db.test.ts306-327](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/db.test.ts#L306-L327)

### Multi-Environment Data Isolation

```
Application Instances

Single D1 Database

Queries only

Queries only

Queries only

Rows with
environment='production'

Rows with
environment='staging'

Rows with
environment='development'

Production App
APP_ENV='production'

Staging App
APP_ENV='staging'

Local Dev
APP_ENV='development'
```

**Sources:**[db/migrations/0001_initial_schema.sql36](https://github.com/egac-web/egac/blob/0ba542fc/db/migrations/0001_initial_schema.sql#L36-L36)[src/lib/db/client.ts](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/db/client.ts)[tests/unit/db.test.ts306-327](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/db.test.ts#L306-L327)

---

## Verification and Testing

After completing database setup, verify the installation using these checks:

### Local Database Verification

```
# 1. Check that tables were created
wrangler d1 execute egac --local --command "SELECT name FROM sqlite_master WHERE type='table';"
 
# Expected output: enquiries, invites, bookings, age_groups, academy_waitlist, 
#                  academy_seasons, email_templates, membership_otps
 
# 2. Verify age groups were seeded
wrangler d1 execute egac --local --command "SELECT code, label, active FROM age_groups;"
 
# Expected: 5 rows (u11, u13, u15, u17, u20)
 
# 3. Verify email templates were seeded
wrangler d1 execute egac --local --command "SELECT key FROM email_templates;"
 
# Expected: 6 rows (booking_invite, booking_confirmation, etc.)
```

### Integration Test Verification

The test suite in `tests/unit/db.test.ts` verifies:

- `generateId()` produces unique prefixed IDs
- `generateToken()` produces cryptographically secure tokens
- `getDB()` throws clear errors when binding is missing
- `getEnv()` returns the correct environment string with fallback to 'production'
- Config defaults and validation work correctly

Run the tests with:

```
npm test
```

**Sources:**[tests/unit/db.test.ts1-328](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/db.test.ts#L1-L328)[vitest.config.ts1-25](https://github.com/egac-web/egac/blob/0ba542fc/vitest.config.ts#L1-L25)

---

## Summary

The database setup process involves:

1. **Binding Configuration:** D1 database is bound via `wrangler.toml` with binding name `DB`
2. **Schema Migration:** Run `npm run migrate:local` to create all tables with environment isolation
3. **Data Seeding:** Run `npm run migrate:seed` to populate age groups and email templates
4. **Client Access:** Use `getDB(env)` to access the database, `getEnv(env)` for environment filtering
5. **Environment Isolation:** All queries automatically filter by `APP_ENV` to isolate dev/staging/production data

For next steps on how repositories use this database client, see [Repository Pattern](/egac-web/egac/7.1-repository-pattern). For details on the entities and their relationships, see [Data Model](/egac-web/egac/1.2-data-model).

**Sources:**[wrangler.toml29-32](https://github.com/egac-web/egac/blob/0ba542fc/wrangler.toml#L29-L32)[package.json18-20](https://github.com/egac-web/egac/blob/0ba542fc/package.json#L18-L20)[db/migrations/0001_initial_schema.sql1-239](https://github.com/egac-web/egac/blob/0ba542fc/db/migrations/0001_initial_schema.sql#L1-L239)[db/seed.sql1-224](https://github.com/egac-web/egac/blob/0ba542fc/db/seed.sql#L1-L224)[src/lib/db/client.ts](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/db/client.ts)[src/lib/repositories/bookings.ts](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/repositories/bookings.ts)[tests/unit/db.test.ts](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/db.test.ts)