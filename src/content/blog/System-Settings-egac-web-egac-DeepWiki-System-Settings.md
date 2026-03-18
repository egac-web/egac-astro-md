# System Settings
Relevant source files
- [db/migrations/0001_initial_schema.sql](https://github.com/egac-web/egac/blob/0ba542fc/db/migrations/0001_initial_schema.sql)
- [db/seed.sql](https://github.com/egac-web/egac/blob/0ba542fc/db/seed.sql)
- [docs/snagging.md](https://github.com/egac-web/egac/blob/0ba542fc/docs/snagging.md)
- [src/lib/repositories/age-groups.ts](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/repositories/age-groups.ts)
- [src/lib/repositories/bookings.ts](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/repositories/bookings.ts)
- [src/pages/admin/academy.astro](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/academy.astro)
- [src/pages/admin/settings.astro](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/settings.astro)
- [tests/unit/db.test.ts](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/db.test.ts)
- [vitest.config.ts](https://github.com/egac-web/egac/blob/0ba542fc/vitest.config.ts)
- [wrangler.toml](https://github.com/egac-web/egac/blob/0ba542fc/wrangler.toml)

## Purpose and Scope

The System Settings page provides administrative controls for configuring global platform behavior, including booking policies, venue information, email settings, and age group definitions. This page manages two distinct types of configuration: **system-wide settings** stored in Cloudflare KV (key-value storage) and **age group definitions** stored in the D1 database. For managing email templates and content, see [Email Templates](/egac-web/egac/8.2-email-templates). For academy season configuration, see [Academy Management](/egac-web/egac/4.6-academy-management).

**Sources:**[src/pages/admin/settings.astro1-294](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/settings.astro#L1-L294)

---

## Configuration Storage Architecture

System settings use a dual-storage model to optimize for different access patterns:

### Configuration Storage Model

```
Storage

Age Groups Layer

Configuration Layer

Admin Settings Page

action=config-save

action=age-group-save

Load data

Load data

Validate first

Write JSON

Invalidate

Read with fallback

Delete cache key

SQL UPDATE

SQL SELECT

settings.astro
Form submission handler

getSystemConfig()

setSystemConfig()

validateConfigUpdate()

clearConfigCache()

listAgeGroups()

updateAgeGroup()

KV Namespace
Key: 'config:system'
JSON value with defaults

D1 Database
Table: age_groups
Rows per age group
```

**Storage Decision Rationale:**
Configuration TypeStorageReasonSystem config (booking rules, site info, email settings)KVLow write frequency, read-heavy, needs fast global access, benefits from cachingAge groupsD1Relational data with foreign key references, queried alongside bookings/enquiries, requires transactional updates
**Sources:**[src/pages/admin/settings.astro4-9](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/settings.astro#L4-L9)[src/lib/repositories/age-groups.ts1-118](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/repositories/age-groups.ts#L1-L118)[wrangler.toml36-39](https://github.com/egac-web/egac/blob/0ba542fc/wrangler.toml#L36-L39)[db/migrations/0001_initial_schema.sql88-106](https://github.com/egac-web/egac/blob/0ba542fc/db/migrations/0001_initial_schema.sql#L88-L106)

---

## System Configuration Fields

### Configuration Data Structure

The `SystemConfig` type defines nine global settings stored as a single JSON object in KV under the key `config:system`:
FieldTypeDefaultValidationPurpose`weeks_ahead_booking`string`"8"`Integer 1-16Number of weeks into the future that taster session dates are generated`capacity_per_slot`string`"2"`Integer 1-10Default capacity per taster session (overridden by age group capacity)`academy_max_age`string`"10"`Integer 5-15Maximum age (on Aug 31) for academy waitlist routing`site_name`string`"East Grinstead Athletics Club"`Non-emptyClub name used in email templates`site_url`string`"https://eastgrinsteadac.co.uk"`Valid URLBase URL for generating booking links`email_from`string`"no-reply@updates.eastgrinsteadac.co.uk"`Valid emailSender address for all outbound emails`venue_name`string`"East Court"`Non-emptyVenue name displayed in confirmations`venue_address`string`"College Lane, East Grinstead RH19 3LT"`Non-emptyFull venue address for booking confirmations`admin_email`string`"admin@eastgrinsteadac.co.uk"`Valid emailRecipient for admin alerts and test messages
**Configuration Read Flow:**

```
"CONFIG_DEFAULTS"
"KV Namespace"
"getSystemConfig()"
"settings.astro"
"CONFIG_DEFAULTS"
"KV Namespace"
"getSystemConfig()"
"settings.astro"
alt
[KV returns value]
[KV returns null or error]
await getSystemConfig(env)
get('config:system')
JSON string
JSON.parse()
{...CONFIG_DEFAULTS, ...parsed}
Use full defaults
SystemConfig object
```

All configuration reads are **fault-tolerant**: if KV is unavailable or returns invalid JSON, the system falls back to `CONFIG_DEFAULTS` defined in `src/lib/config.js`.

**Sources:**[src/pages/admin/settings.astro88-98](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/settings.astro#L88-L98)[tests/unit/db.test.ts141-169](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/db.test.ts#L141-L169)[tests/unit/db.test.ts236-274](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/db.test.ts#L236-L274)

---

## Configuration Update Flow

### Save Operation

When the admin submits the system config form (`action=config-save`), the following sequence executes:

```
"KV Namespace"
"clearConfigCache()"
"setSystemConfig()"
"validateConfigUpdate()"
"settings.astro POST handler"
"Admin User"
"KV Namespace"
"clearConfigCache()"
"setSystemConfig()"
"validateConfigUpdate()"
"settings.astro POST handler"
"Admin User"
alt
[Validation fails]
[Validation succeeds]
Submit config form
Extract form fields
validateConfigUpdate(updates)
errors[]
Show error list
[]
setSystemConfig(updates, env)
Merge with existing config
put('config:system', JSON)
clearConfigCache(env)
delete('config:system:cache')
"Settings saved."
```

**Validation Rules Enforced by `validateConfigUpdate()`:**

- `weeks_ahead_booking`: Must parse to integer between 1 and 16
- `capacity_per_slot`: Must parse to integer between 1 and 10
- `academy_max_age`: Must parse to integer between 5 and 15
- `email_from`, `admin_email`: Must match email regex
- `site_url`: Must be valid URL starting with `http://` or `https://`
- All string fields: Must be non-empty after trimming

The function returns an array of error messages. An empty array indicates valid input.

**Sources:**[src/pages/admin/settings.astro87-108](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/settings.astro#L87-L108)[tests/unit/db.test.ts236-274](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/db.test.ts#L236-L274)

---

## Age Groups Configuration

### Age Group Data Model

Age groups define how athletes are categorized based on their age on August 31st of the athletics season. Each age group stored in the `age_groups` table determines:

1. **Booking routing**: Whether athletes book taster sessions or join the academy waitlist
2. **Session scheduling**: Which days of the week sessions are offered
3. **Capacity limits**: How many athletes can book per session
4. **Age eligibility**: Minimum and maximum age boundaries

```
classified by

assigned to

defines capacity

age_groups

string

id

PK

string

code

UK

e.g. 'u13', 'u15'

string

label

e.g. 'U13'

enum

booking_type

'taster' or 'waitlist'

int

age_min_aug31

Min age on Aug 31

int

age_max_aug31

Max age on Aug 31

string

session_days

JSON array

string

session_time

e.g. '18:30'

int

capacity_per_session

Null = use system default

int

active

1=active, 0=inactive

int

sort_order

Display order

enquiries

bookings

academy_seasons
```

### Age Group Fields
FieldTypeRequiredPurposeExample`id`TEXTYesUnique identifier`"ag_u13"``code`TEXTYesShort code (unique)`"u13"``label`TEXTYesDisplay name`"U13"``booking_type`ENUMYes`"taster"` or `"waitlist"``"taster"``age_min_aug31`INTEGERYesMinimum age on Aug 31`11``age_max_aug31`INTEGERYesMaximum age on Aug 31`12``session_days`TEXT (JSON)YesDays sessions run`["Tuesday"]``session_time`TEXTNoSession start time`"18:30"``capacity_per_session`INTEGERNoCapacity override (null = use system default)`2``active`INTEGERYes1 = visible, 0 = hidden`1``sort_order`INTEGERYesDisplay/processing order`2`
**Booking Type Routing:**

- **`taster`**: Athletes are sent a booking invite to select from available session dates. After booking, they attend once and can be invited to full membership. Used for U13 and older.
- **`waitlist`**: Athletes are added to the academy waitlist and notified when a season place becomes available. Used for U11 academy. See [Academy Management](/egac-web/egac/4.6-academy-management) for waitlist details.

**Sources:**[db/migrations/0001_initial_schema.sql88-111](https://github.com/egac-web/egac/blob/0ba542fc/db/migrations/0001_initial_schema.sql#L88-L111)[db/seed.sql12-53](https://github.com/egac-web/egac/blob/0ba542fc/db/seed.sql#L12-L53)[src/pages/admin/settings.astro24-87](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/settings.astro#L24-L87)

---

## Age Group Update Process

### Update Flow

```
"D1 age_groups table"
"updateAgeGroup()"
"Form validation"
"settings.astro POST handler"
"Admin User"
"D1 age_groups table"
"updateAgeGroup()"
"Form validation"
"settings.astro POST handler"
"Admin User"
alt
[Invalid age range]
[Invalid session_days JSON]
[Invalid capacity (not 1-20)]
[Valid]
Submit age group form
action='age-group-save'
Check fields
"age range is invalid"
"must be JSON array"
"capacity must be 1-20"
updateAgeGroup(id, updates, env)
Build SET clause dynamically
UPDATE age_groups SET ... WHERE id = ?
SELECT * FROM age_groups WHERE id = ?
Updated row
AgeGroup object
"Age group updated."
```

**Validation Rules for Age Groups:**

```
// Field validations performed in settings.astro:24-68
- age_group_id: Required, non-empty string
- label: Required, non-empty string
- booking_type: Must be exactly 'taster' or 'waitlist'
- age_min_aug31, age_max_aug31: Must be integers, age_max >= age_min, both >= 0
- sort_order: Must be positive integer
- session_days: Must parse as JSON array of strings (e.g. ["Tuesday"])
- capacity_per_session: Optional; if provided, must be integer 1-20
- active: Converted to 1 or 0 based on checkbox state
```

The `updateAgeGroup()` function in `src/lib/repositories/age-groups.ts` uses dynamic SQL construction: only fields present in the `updates` object are included in the `SET` clause. This allows partial updates without overwriting unchanged fields.

**Sources:**[src/pages/admin/settings.astro24-87](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/settings.astro#L24-L87)[src/lib/repositories/age-groups.ts47-117](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/repositories/age-groups.ts#L47-L117)

---

## Admin Interface Layout

### Settings Page Structure

The settings page at `/admin/settings` is organized into five sections:

```
Age Groups Section

Config Form (action=config-save)

Metrics Cards

Top Bar

settings.astro
/admin/settings

H1: Settings

Environment Badge
Shows APP_ENV

Weeks ahead
Display only

Slot capacity
Display only

Academy max age
Display only

Booking defaults
3 fields

Site and venue
4 fields

Email settings
2 fields

Save settings button

H2: Age groups

One form per age group
(action=age-group-save)
```

**Top-Level Structure:**

1. **Header with environment badge**[src/pages/admin/settings.astro117-122](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/settings.astro#L117-L122): Displays current `APP_ENV` (development/staging/production) to prevent accidental changes to production data
2. **Success/error messages**[src/pages/admin/settings.astro124-141](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/settings.astro#L124-L141): Form submission feedback
3. **Metrics overview cards**[src/pages/admin/settings.astro143-156](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/settings.astro#L143-L156): Read-only display of key settings for quick reference
4. **System config form**[src/pages/admin/settings.astro158-228](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/settings.astro#L158-L228): Single form with three sub-sections and one submit button
5. **Age groups section**[src/pages/admin/settings.astro230-291](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/settings.astro#L230-L291): Multiple independent forms, one per age group

**Form Submission Behavior:**

- Both forms use `method="POST"` to the same page
- `action` hidden field differentiates between `config-save` and `age-group-save`
- On success, the page reloads with a success message
- On validation error, the page reloads with error list and form values preserved

**Sources:**[src/pages/admin/settings.astro115-293](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/settings.astro#L115-L293)

---

## Configuration Management API

### Core Functions

The configuration system is implemented across three modules:

```
Cloudflare KV

src/lib/config.js

Read with fallback

Write merged config

Delete

getSystemConfig(env)
Returns: SystemConfig

setSystemConfig(updates, env)
Returns: SystemConfig

validateConfigUpdate(updates)
Returns: string[]

clearConfigCache(env)
Returns: void

Type-safe getters:
getAcademyMaxAge()
getWeeksAhead()
getCapacityPerSlot()

'config:system'
JSON value

'config:system:cache'
Cached JSON
```

### Function Reference Table
FunctionParametersReturn TypePurpose`getSystemConfig(env)``env: Env``Promise<SystemConfig>`Reads config from KV, merges with defaults, fault-tolerant`setSystemConfig(updates, env)``updates: Partial<SystemConfig>`, `env: Env``Promise<SystemConfig>`Validates, merges, writes to KV, returns full config`validateConfigUpdate(updates)``updates: Partial<SystemConfig>``string[]`Returns array of error messages (empty = valid)`clearConfigCache(env)``env: Env``Promise<void>`Deletes cache key to force fresh read on next request`getAcademyMaxAge(config)``config: SystemConfig``number`Parses `academy_max_age` string to integer, fallback 10`getWeeksAhead(config)``config: SystemConfig``number`Parses `weeks_ahead_booking` string to integer, fallback 8`getCapacityPerSlot(config)``config: SystemConfig``number`Parses `capacity_per_slot` string to integer, fallback 2
**Configuration Defaults:**

The `CONFIG_DEFAULTS` object in `src/lib/config.js` defines fallback values used when:

- KV is unavailable
- KV returns null (key not set)
- KV returns invalid JSON
- Specific fields are missing from stored config

**Cache Strategy:**

The system employs an **optional cache layer** at `config:system:cache` for read optimization. After calling `setSystemConfig()`, the cache must be cleared via `clearConfigCache()` to prevent stale reads. The settings page explicitly calls this after each save [src/pages/admin/settings.astro104](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/settings.astro#L104-L104)

**Sources:**[tests/unit/db.test.ts141-231](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/db.test.ts#L141-L231)[src/pages/admin/settings.astro4-9](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/settings.astro#L4-L9)[src/pages/admin/settings.astro100-105](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/settings.astro#L100-L105)

---

## Age Group Repository Functions

### Age Groups API Reference

The `src/lib/repositories/age-groups.ts` module provides typed access to the `age_groups` table:
FunctionParametersReturn TypePurpose`listAgeGroups(env, activeOnly)``env: Env`, `activeOnly: boolean = true``Promise<AgeGroup[]>`Returns all age groups, optionally filtered by `active=1`, sorted by `sort_order``getAgeGroupById(id, env)``id: string`, `env: Env``Promise<AgeGroup | null>`Retrieves single age group by ID`getAgeGroupByCode(code, env)``code: string`, `env: Env``Promise<AgeGroup | null>`Retrieves active age group by code (e.g. "u13")`getTasterAgeGroups(env)``env: Env``Promise<AgeGroup[]>`Returns only age groups with `booking_type='taster'` and `active=1``getWaitlistAgeGroups(env)``env: Env``Promise<AgeGroup[]>`Returns only age groups with `booking_type='waitlist'` and `active=1``updateAgeGroup(id, updates, env)``id: string`, `updates: Partial<AgeGroup>`, `env: Env``Promise<AgeGroup | null>`Updates specified fields, returns updated row
**Dynamic Update Implementation:**

The `updateAgeGroup()` function builds SQL dynamically to avoid overwriting fields not included in the `updates` object:

```
// Pseudo-code from src/lib/repositories/age-groups.ts:47-117
const fields: string[] = [];
const values: unknown[] = [];
 
if (updates.label !== undefined) {
  fields.push('label = ?');
  values.push(updates.label);
}
// ... repeat for each updatable field
 
if (fields.length === 0) return getAgeGroupById(id, env);
 
fields.push('updated_at = ?');
values.push(new Date().toISOString());
 
await db.prepare(`UPDATE age_groups SET ${fields.join(', ')} WHERE id = ?`)
  .bind(...values, id)
  .run();
```

This approach ensures that only explicitly provided fields are updated, preserving other field values.

**Sources:**[src/lib/repositories/age-groups.ts9-117](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/repositories/age-groups.ts#L9-L117)

---

## Environment Awareness

### Multi-Environment Architecture

The settings page displays the current `APP_ENV` value in a badge at the top right [src/pages/admin/settings.astro119-121](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/settings.astro#L119-L121) This visual indicator helps administrators distinguish between environments:
EnvironmentUsageData Isolation`development`Local development with `npm run dev`Uses local D1 database via `wrangler dev``staging`Pre-production testing environmentSeparate D1 database and KV namespace`production`Live public-facing instanceProduction D1 and KV bindings defined in `wrangler.toml`
**Environment Value Sources:**

1. **KV-stored config**: Does not include `APP_ENV` — it's not user-editable
2. **Cloudflare binding**: `env.APP_ENV` comes from `[vars]` in `wrangler.toml`[wrangler.toml22-24](https://github.com/egac-web/egac/blob/0ba542fc/wrangler.toml#L22-L24)
3. **Runtime access**: `Astro.locals.runtime.env.APP_ENV` in pages, `env.APP_ENV` in API handlers

**Data Isolation Enforcement:**

Although the settings page does not directly implement environment filtering (system config in KV is global per namespace, and age groups table does not have an `environment` column), other tables like `enquiries`, `bookings`, and `academy_waitlist` include an `environment` column to enforce multi-tenant isolation. The repository layer automatically adds `AND environment = ?` to all queries.

**Sources:**[src/pages/admin/settings.astro112-122](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/settings.astro#L112-L122)[wrangler.toml22-24](https://github.com/egac-web/egac/blob/0ba542fc/wrangler.toml#L22-L24)[db/migrations/0001_initial_schema.sql36](https://github.com/egac-web/egac/blob/0ba542fc/db/migrations/0001_initial_schema.sql#L36-L36)

---

## Integration with Other Systems

### Configuration Consumers

System configuration and age group definitions are consumed by multiple subsystems:

```
weeks_ahead_booking

capacity_per_slot

academy_max_age

site_name, site_url

venue_name, venue_address

email_from

age ranges

booking_type

session_days

capacity_per_session

Filter by active

Group by age_group_id

System Config
(KV)

Age Groups
(D1)

Enquiry Form
index.astro

Booking Flow
booking.astro

Email Templates
sendBookingInvite()

Age Resolution
resolveAgeGroup()

Session Scheduling
getNextNSessionDates()

Reports Dashboard
reports.astro
```

**Key Integration Points:**

1. **Enquiry routing**[Page 6.2](/egac-web/egac/6.2-enquiry-routing): `resolveAgeGroup()` uses age group definitions to determine if an athlete gets a taster invite or joins the academy waitlist
2. **Session scheduling**[Page 6.3](/egac-web/egac/6.3-session-scheduling): `getNextNSessionDates()` uses `weeks_ahead_booking`, `session_days`, and `capacity_per_session` to generate available booking dates
3. **Email templates**[Page 8.2](/egac-web/egac/8.2-email-templates): All email templates use config values like `{{site_name}}`, `{{venue_name}}`, and `{{venue_address}}`
4. **Capacity enforcement**: When creating bookings, `capacity_per_session` from the age group (or system default) determines if a session is full

**Configuration Change Impact:**

- Changes to `weeks_ahead_booking` immediately affect new booking invites (existing invites remain valid with their original dates)
- Changes to age group `booking_type` affect routing for **new enquiries only** — existing enquiries retain their assigned `age_group_id` and routing
- Changes to `session_days` affect scheduling for new invites but do not invalidate existing bookings
- Deactivating an age group (`active=0`) hides it from the enquiry form and booking flows but does not affect existing bookings

**Sources:**[src/pages/admin/settings.astro110-112](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/settings.astro#L110-L112)[db/migrations/0001_initial_schema.sql88-106](https://github.com/egac-web/egac/blob/0ba542fc/db/migrations/0001_initial_schema.sql#L88-L106)