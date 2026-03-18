---
title: "EGAC Platform Overview"
description: "This page provides an overview of the platform"
pubDate: 2026-03-18
---
# EGAC Platform Overview
Relevant source files
- [db/migrations/0001_initial_schema.sql](https://github.com/egac-web/egac/blob/0ba542fc/db/migrations/0001_initial_schema.sql)
- [db/seed.sql](https://github.com/egac-web/egac/blob/0ba542fc/db/seed.sql)
- [package.json](https://github.com/egac-web/egac/blob/0ba542fc/package.json)
- [src/api/booking.ts](https://github.com/egac-web/egac/blob/0ba542fc/src/api/booking.ts)
- [src/api/enquiry.ts](https://github.com/egac-web/egac/blob/0ba542fc/src/api/enquiry.ts)
- [src/api/health.ts](https://github.com/egac-web/egac/blob/0ba542fc/src/api/health.ts)
- [tests/unit/api.test.ts](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/api.test.ts)
- [tests/unit/db.test.ts](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/db.test.ts)
- [tsconfig.json](https://github.com/egac-web/egac/blob/0ba542fc/tsconfig.json)
- [vitest.config.ts](https://github.com/egac-web/egac/blob/0ba542fc/vitest.config.ts)
- [wrangler.toml](https://github.com/egac-web/egac/blob/0ba542fc/wrangler.toml)

This document provides a high-level introduction to the East Grinstead Athletics Club (EGAC) platform, explaining its purpose, architecture, and key features. It serves as an entry point for developers, covering what the platform does, how it is structured, and how its major components interact.

For detailed architecture and component relationships, see [System Architecture](/egac-web/egac/1.1-system-architecture). For the complete data model and entity relationships, see [Data Model](/egac-web/egac/1.2-data-model).

---

## Purpose and Scope

The EGAC platform is a web-based enquiry and booking system for East Grinstead Athletics Club. It manages the end-to-end journey from initial enquiry through taster session booking and academy waitlist management. The platform automatically routes enquiries based on age group eligibility and provides administrative tools for managing bookings, tracking attendance, and generating reports.

**What this platform handles:**
User JourneyDescriptionKey Features**Public Enquiry**Athletes or parents submit enquiries via web formAge group resolution, automatic routing, email invites**Taster Bookings**Athletes 11+ book taster sessions using invite tokensDate selection, capacity management, confirmations**Academy Waitlist**Athletes 9-10 join waitlist for academy seasonsWaitlist queue, season rollover, acceptance tracking**Admin Management**Club administrators manage enquiries and bookingsCalendar view, attendance tracking, reports, settings**Automated Jobs**Background tasks handle lifecycle eventsInvite expiry, email retries, reminders, season rollover
**Sources:**[wrangler.toml1-49](https://github.com/egac-web/egac/blob/0ba542fc/wrangler.toml#L1-L49)[package.json1-48](https://github.com/egac-web/egac/blob/0ba542fc/package.json#L1-L48)[tests/unit/api.test.ts1-50](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/api.test.ts#L1-L50)

---

## Platform Architecture

The EGAC platform is a Cloudflare Pages application built with Astro, comprising public-facing pages, an admin interface, and API endpoints. A separate Cloudflare Worker handles scheduled jobs. All data is persisted in Cloudflare D1 (SQLite) and KV storage, with transactional emails sent via Resend.

### High-Level Architecture Diagram

```
External Services

Data Storage

Cloudflare Worker

Cloudflare Pages Application

Client Layer

Infrastructure

Email System

Data Access Layer

Business Logic Layer

API Handlers

Public Web Interface
(index.astro, enquiry form)

Admin Web Interface
(admin/*.astro)

handleEnquiry
/api/enquiry

handleBooking
/api/booking

handleAcademyRespond
/api/academy/respond

handleHealth
/api/health

Admin APIs
/api/admin/*

Cron APIs
/api/cron/*

validateEnquiryData
routeEnquiry

resolveAgeGroup
ageForAthletics
getNextNTuesdayDates

validateBookingRequest
buildBookingConfirmation

generateSecureToken
isInviteExpired

enquiries repository
insertEnquiry
getEnquiryByEmail

invites repository
createInviteForEnquiry
markInviteSent

bookings repository
createBooking
countBookingsForDateAndGroup

academy repository
addToWaitlist
recordWaitlistResponse

age-groups repository
listAgeGroups
getAgeGroupById

Email Templates
sendBookingInvite
sendBookingConfirmation
sendAcademyWaitlist

sendEmailWithRetry
Resend API integration

getDB
envQuery

getSystemConfig
setSystemConfig

egac-cron worker
Scheduled event handler

D1 Database
(SQLite)
enquiries, bookings,
invites, academy_waitlist,
age_groups, templates

KV Namespace
config:system

Resend Email API
api.resend.com
```

**Sources:**[tests/unit/api.test.ts1-699](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/api.test.ts#L1-L699)[src/api/enquiry.ts1-224](https://github.com/egac-web/egac/blob/0ba542fc/src/api/enquiry.ts#L1-L224)[src/api/booking.ts1-147](https://github.com/egac-web/egac/blob/0ba542fc/src/api/booking.ts#L1-L147)[wrangler.toml1-49](https://github.com/egac-web/egac/blob/0ba542fc/wrangler.toml#L1-L49)

---

## Technology Stack

The platform is built on Cloudflare's edge infrastructure using modern web technologies.
ComponentTechnologyPurpose**Frontend Framework**Astro 4.0Server-side rendering, component-based UI**Styling**Tailwind CSSUtility-first CSS framework**Runtime**Cloudflare PagesEdge hosting, serverless functions**Database**Cloudflare D1Relational database (SQLite)**Cache/Config**Cloudflare KVKey-value storage for system configuration**Email**Resend APITransactional email delivery**Scheduled Jobs**Cloudflare WorkerSeparate worker for cron triggers**Language**TypeScript 5.5Type-safe development**Testing**Vitest 2.0Unit and integration testing
**Build and deployment:**

- Pages build output directory: `dist` (configured in [wrangler.toml7](https://github.com/egac-web/egac/blob/0ba542fc/wrangler.toml#L7-L7))
- Node.js requirement: `>=18.0.0` ([package.json45-47](https://github.com/egac-web/egac/blob/0ba542fc/package.json#L45-L47))
- Environment-aware: `APP_ENV` controls development/staging/production isolation

**Sources:**[package.json1-48](https://github.com/egac-web/egac/blob/0ba542fc/package.json#L1-L48)[tsconfig.json1-20](https://github.com/egac-web/egac/blob/0ba542fc/tsconfig.json#L1-L20)[wrangler.toml1-49](https://github.com/egac-web/egac/blob/0ba542fc/wrangler.toml#L1-L49)[vitest.config.ts1-25](https://github.com/egac-web/egac/blob/0ba542fc/vitest.config.ts#L1-L25)

---

## Core Data Model

The platform centers on five primary entities, with age groups serving as the routing mechanism for enquiries.

### Entity Relationship Overview

```
generates

creates

joins

classified by

results in

assigned to

enrolled in

defines

enquiries

text

id

PK

enq_*

text

email

UK

text

name

text

dob

text

age_group_id

FK

text

invite_id

FK

integer

processed

text

events

JSON array

text

environment

invites

text

id

PK

inv_*

text

token

UK

text

enquiry_id

FK

text

status

pending|sent|accepted|expired

integer

send_attempts

datetime

created_at

datetime

sent_at

bookings

text

id

PK

bkg_*

text

enquiry_id

FK

text

invite_id

FK

text

age_group_id

FK

date

session_date

text

status

confirmed|attended|no_show|cancelled

academy_waitlist

text

id

PK

awl_*

text

enquiry_id

FK

text

season_id

FK

text

token

UK

integer

position

text

status

waiting|invited|accepted|declined

integer

is_returning

age_groups

text

id

PK

ag_*

text

code

UK

u11, u13, u15, etc

text

booking_type

taster|waitlist

integer

age_min_aug31

integer

age_max_aug31

text

session_days

JSON array

integer

capacity_per_session

integer

active

academy_seasons

text

id

PK

asn_*

text

age_group_id

FK

date

start_date

date

end_date

integer

capacity

text

status

upcoming|open|closed|archived
```

**Key design principles:**

1. **Single Source of Truth:**`enquiries` table stores all contact information. `age_group_id` is resolved once and stored ([db/migrations/0001_initial_schema.sql22-45](https://github.com/egac-web/egac/blob/0ba542fc/db/migrations/0001_initial_schema.sql#L22-L45)).
2. **Age Groups as Configuration:** No hardcoded slot codes. All age group logic reads from `age_groups` table ([db/migrations/0001_initial_schema.sql88-111](https://github.com/egac-web/egac/blob/0ba542fc/db/migrations/0001_initial_schema.sql#L88-L111)).
3. **Routing via `booking_type`:** Age groups with `booking_type = 'taster'` trigger invite creation; `booking_type = 'waitlist'` adds to academy waitlist ([db/seed.sql12-53](https://github.com/egac-web/egac/blob/0ba542fc/db/seed.sql#L12-L53)).
4. **Event Sourcing:** The `events` JSON field on enquiries provides an audit trail of all state changes ([db/migrations/0001_initial_schema.sql34](https://github.com/egac-web/egac/blob/0ba542fc/db/migrations/0001_initial_schema.sql#L34-L34)).
5. **Environment Isolation:** Every table has an `environment` column to separate development/staging/production data ([db/migrations/0001_initial_schema.sql36](https://github.com/egac-web/egac/blob/0ba542fc/db/migrations/0001_initial_schema.sql#L36-L36)).

**Sources:**[db/migrations/0001_initial_schema.sql1-239](https://github.com/egac-web/egac/blob/0ba542fc/db/migrations/0001_initial_schema.sql#L1-L239)[db/seed.sql1-224](https://github.com/egac-web/egac/blob/0ba542fc/db/seed.sql#L1-L224)

---

## API Endpoint Structure

The platform exposes three categories of API endpoints, each with distinct authentication requirements and purposes.

### API Routes by Type
EndpointMethodHandlerAuthenticationPurpose**Public APIs**`/api/enquiry`POST`handleEnquiry`NoneSubmit new enquiry, triggers age group routing`/api/booking`POST`handleBooking`Token-basedCreate taster session booking via invite token`/api/academy/respond`POST`handleAcademyRespond`Token-basedAccept/decline academy waitlist invitation`/api/health`GET`handleHealth`NoneHealth check for monitoring**Admin APIs**`/api/admin/enquiries`GET`handleAdminEnquiries``ADMIN_TOKEN`List and filter enquiries`/api/admin/bookings`GET/PATCH`handleAdminBookings``ADMIN_TOKEN`Manage bookings, mark attendance`/api/admin/invites/resend`POST`handleAdminInviteResend``ADMIN_TOKEN`Manually retry invite email**Cron APIs**`/api/cron/expire-invites`POST`handleExpireInvites``CRON_SECRET`Mark invites >14 days as expired`/api/cron/retry-invites`POST`handleRetryInvites``CRON_SECRET`Retry failed invite sends (max 3 attempts)`/api/cron/send-reminders`POST`handleSendReminders``CRON_SECRET`Send booking reminders for upcoming sessions`/api/cron/academy-rollover`POST`handleAcademyRollover``CRON_SECRET`Close expired seasons, migrate waitlist
**Authentication mechanism:**

- Public endpoints: No authentication required
- Token-based: Secure tokens embedded in URLs (e.g., `?token=abc123...`)
- Admin endpoints: Require `Authorization: Bearer {ADMIN_TOKEN}` header
- Cron endpoints: Require `x-cron-secret: {CRON_SECRET}` header

**Sources:**[tests/unit/api.test.ts196-698](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/api.test.ts#L196-L698)[src/api/enquiry.ts86-223](https://github.com/egac-web/egac/blob/0ba542fc/src/api/enquiry.ts#L86-L223)[src/api/booking.ts25-146](https://github.com/egac-web/egac/blob/0ba542fc/src/api/booking.ts#L25-L146)[src/api/health.ts17-40](https://github.com/egac-web/egac/blob/0ba542fc/src/api/health.ts#L17-L40)

---

## Enquiry Routing System

The enquiry routing system is the core business logic that determines whether an athlete is directed to taster sessions or the academy waitlist. This decision is based on age group configuration stored in the `age_groups` table.

### Routing Logic Flow

```
taster

waitlist

POST /api/enquiry
{name, email, dob}

parsePublicEnquiryBody
Handle JSON or form data

validateEnquiryData
Check required fields

insertEnquiry
Create enquiry record

listAgeGroups
Load all age groups from D1

resolveAgeGroup(dob, date, groups)
Calculate age on Aug 31

updateEnquiry
Set age_group_id

routeEnquiry(ageGroup)
Check booking_type

booking_type = 'taster'

booking_type = 'waitlist'

createInviteForEnquiry
Generate secure token

getNextNTuesdayDates
Calculate available dates

sendBookingInvite
Email with booking link

markInviteSent
Update invite status

addToWaitlist
Create academy_waitlist entry

sendAcademyWaitlist
Confirmation email

markWaitlistInviteSent
Update send status

appendEnquiryEvent
Record to events JSON

updateEnquiry
Set processed = 1

201 Created
{ok: true}
```

**Age calculation rule:**

- Age is determined on **31 August** of the athletics season the session falls in
- Example: A session on 14 October 2026 uses 31 August 2026 as the reference date
- Athlete born 15 June 2014 would be 12 years old on 31 Aug 2026 → U13 group

**Routing logic implementation:**

1. `resolveAgeGroup(dob, sessionDate, ageGroups)` calculates age on Aug 31 ([src/api/enquiry.ts159](https://github.com/egac-web/egac/blob/0ba542fc/src/api/enquiry.ts#L159-L159))
2. `routeEnquiry(ageGroup)` returns `'taster'` or `'waitlist'` based on `booking_type` ([src/api/enquiry.ts166](https://github.com/egac-web/egac/blob/0ba542fc/src/api/enquiry.ts#L166-L166))
3. Email always sent, but failures don't block enquiry creation ([src/api/enquiry.ts168-220](https://github.com/egac-web/egac/blob/0ba542fc/src/api/enquiry.ts#L168-L220))

**Sources:**[src/api/enquiry.ts86-223](https://github.com/egac-web/egac/blob/0ba542fc/src/api/enquiry.ts#L86-L223)[tests/unit/api.test.ts199-401](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/api.test.ts#L199-L401)[db/seed.sql12-53](https://github.com/egac-web/egac/blob/0ba542fc/db/seed.sql#L12-L53)

---

## Scheduled Jobs Architecture

Background tasks are handled by a separate Cloudflare Worker (`egac-cron`) that triggers API endpoints on a schedule. This compensates for Cloudflare Pages' lack of native cron support.

### Cron Worker Trigger Flow

```
D1 Database
Repository Layer
Cron API Handler
Pages Project
egac-cron Worker
Cloudflare Cron
D1 Database
Repository Layer
Cron API Handler
Pages Project
egac-cron Worker
Cloudflare Cron
Daily 09:00 UTC - Expire Invites
Every 30 minutes - Retry Failed Invites
Tuesday 18:00 UTC - Send Reminders
Trigger: 0 9 * * *
Lookup route: expire-invites
POST /api/cron/expire-invites
Header: x-cron-secret
handleExpireInvites
Find invites created >14 days ago
SELECT * FROM invites
WHERE created_at < date('now', '-14 days')
Expired invites
UPDATE invites SET status = 'expired'
200 OK {expired_count: n}
Response
Trigger: */30 * * * *
POST /api/cron/retry-invites
Find invites with status='pending'
AND send_attempts < 3
SELECT from invites
sendBookingInvite for each
markInviteSent OR markInviteFailed
UPDATE send_attempts, status
Trigger: 0 18 * * 2
POST /api/cron/send-reminders
Find bookings for next Tuesday
Check if reminder already sent
(events JSON)
sendBookingReminder
appendEnquiryEvent('reminder_sent')
```

**Cron schedule configuration:**

- Configured in Cloudflare Pages dashboard (Pages → Settings → Functions → Cron Triggers)
- Schedule syntax follows standard cron format
- Worker validates `x-cron-secret` header to prevent unauthorized triggers

**Job inventory:**
ScheduleEndpointPurposeIdempotency`0 9 * * *``/api/cron/expire-invites`Mark invites created >14 days ago as expiredYes - checks `status != 'expired'``*/30 * * * *``/api/cron/retry-invites`Retry failed email sends (max 3 attempts)Yes - checks `send_attempts < 3``0 18 * * 2``/api/cron/send-reminders`Send reminder emails before Tuesday sessionsYes - checks `events` for `reminder_sent``0 9 1 * *``/api/cron/academy-rollover`Close expired seasons, migrate waitlist entriesYes - checks season status
**Sources:**[wrangler.toml42-48](https://github.com/egac-web/egac/blob/0ba542fc/wrangler.toml#L42-L48)[tests/unit/api.test.ts1-50](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/api.test.ts#L1-L50)

---

## Email System

The email system uses Resend API for delivery and supports templating with variable substitution. All templates are stored in the `email_templates` table with HTML and plain-text versions.

### Email Template Types
Template KeyTriggerVariablesPurpose`booking_invite`Enquiry submitted (taster path)`contact_name`, `booking_url`, `available_dates_list`Invite to book taster session`booking_confirmation`Booking created`contact_name`, `session_date`, `session_time`, `age_group_label`, `venue`, `add_to_calendar_url`Confirm session details`academy_waitlist`Enquiry submitted (waitlist path)`parent_name`, `child_name`Acknowledge waitlist placement`academy_invite`Season opens, place available`parent_name`, `child_name`, `season_label`, `season_start`, `season_end`, `accept_url`, `decline_url`Offer academy place`membership_invite`Admin marks attendance`contact_name`, `membership_url`Invite to join as member`booking_reminder`Cron job (Tuesday 18:00)`contact_name`, `session_date`, `session_time`, `age_group_label`, `venue`Remind about upcoming session
**Template rendering process:**

1. `getEmailTemplateByKey(key, env)` fetches template from D1
2. Build variable map with context-specific data (URLs, dates, names)
3. `renderTemplate(template, variables)` replaces `{{variable}}` placeholders
4. `sendEmailWithRetry(email, subject, html, text, env)` sends via Resend with retry logic

**Fault tolerance:**

- Email failures never block core operations (enquiry creation, booking confirmation)
- Failed sends are logged with `level: 'error'` for monitoring
- Retry mechanism: cron job attempts resend for `status='pending'` invites up to 3 times
- Missing template variables throw validation errors to prevent malformed emails

**Sources:**[db/seed.sql55-224](https://github.com/egac-web/egac/blob/0ba542fc/db/seed.sql#L55-L224)[src/api/enquiry.ts168-220](https://github.com/egac-web/egac/blob/0ba542fc/src/api/enquiry.ts#L168-L220)[src/api/booking.ts108-136](https://github.com/egac-web/egac/blob/0ba542fc/src/api/booking.ts#L108-L136)

---

## Configuration Management

System configuration is stored in Cloudflare KV under the key `config:system`. Default values are defined in code, with KV overrides merged at runtime.

### Configuration Schema

```
Typed Parsers

Access Layer

Configuration Sources

CONFIG_DEFAULTS
(in code)

KV Namespace
config:system

getSystemConfig(env)
Merges defaults + KV

setSystemConfig(updates, env)
Writes to KV

validateConfigUpdate(updates)
Type and range checks

getAcademyMaxAge(config)
Returns integer

getWeeksAhead(config)
Returns integer

getCapacityPerSlot(config)
Returns integer
```

**Configuration keys:**
KeyTypeDefaultValidationUsage`academy_max_age`string`"10"`IntegerAge threshold for academy waitlist routing`weeks_ahead_booking`string`"8"`1-16How many weeks ahead to show available dates`capacity_per_slot`string`"2"`1-10Default session capacity (overridden by age group)`site_name`string`"East Grinstead Athletics Club"`Non-emptySite branding`site_url`string`"https://eastgrinsteadac.co.uk"`Valid URLBase URL for email links`email_from`string`"no-reply@updates.eastgrinsteadac.co.uk"`Valid emailEmail sender address`venue_name`string`"East Grinstead Leisure Centre"`Non-emptySession location name`venue_address`string`"1 Station Road"`Non-emptySession location address`admin_email`string`"admin@eastgrinsteadac.co.uk"`Valid emailAdmin contact
**Admin interface integration:**

- Configuration page at `/admin/settings` allows editing these values
- `validateConfigUpdate(updates)` enforces constraints before writing
- Changes are immediate (no deployment required)

**Sources:**[tests/unit/db.test.ts139-301](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/db.test.ts#L139-L301)[wrangler.toml22-25](https://github.com/egac-web/egac/blob/0ba542fc/wrangler.toml#L22-L25)

---

## Environment Isolation

The platform supports multi-tenancy via the `environment` column on all tables. This enables development, staging, and production data to coexist in the same D1 database.

### Environment Strategy
Environment`APP_ENV` ValueControlled ByPurpose**Development**`development`[wrangler.toml23](https://github.com/egac-web/egac/blob/0ba542fc/wrangler.toml#L23-L23) (can be overridden locally)Local development, testing**Staging**`staging`Cloudflare Pages environment variablePre-production verification**Production**`production`Cloudflare Pages environment variableLive club operations
**Implementation details:**

- All repository queries use `envQuery` builder that automatically filters by `environment` ([tests/unit/db.test.ts321-327](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/db.test.ts#L321-L327))
- `getEnv(env)` helper returns current environment with fallback to `'production'` ([tests/unit/db.test.ts124-135](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/db.test.ts#L124-L135))
- Email templates use same `environment` filter to prevent cross-environment pollution
- `generateId(prefix)` creates environment-agnostic IDs (no environment in ID string)

**Sources:**[tests/unit/db.test.ts304-327](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/db.test.ts#L304-L327)[db/migrations/0001_initial_schema.sql36](https://github.com/egac-web/egac/blob/0ba542fc/db/migrations/0001_initial_schema.sql#L36-L36)[wrangler.toml22-25](https://github.com/egac-web/egac/blob/0ba542fc/wrangler.toml#L22-L25)

---

## Key File Structure

The codebase follows a layered architecture with clear separation of concerns.
Directory/FilePurposeKey Exports**API Layer**`src/api/enquiry.ts`Enquiry submission handler`handleEnquiry``src/api/booking.ts`Booking creation handler`handleBooking``src/api/health.ts`Health check endpoint`handleHealth`**Business Logic**`src/lib/business/age.ts`Age group calculation`resolveAgeGroup`, `ageForAthletics`, `getNextNTuesdayDates``src/lib/business/enquiry.ts`Enquiry validation and routing`validateEnquiryData`, `routeEnquiry``src/lib/business/booking.ts`Booking validation`validateBookingRequest`, `buildBookingConfirmation``src/lib/business/tokens.ts`Token generation and validation`generateSecureToken`, `isInviteExpired`**Data Access**`src/lib/repositories/enquiries.js`Enquiry CRUD operations`insertEnquiry`, `getEnquiryByEmail`, `appendEnquiryEvent``src/lib/repositories/invites.js`Invite lifecycle management`createInviteForEnquiry`, `markInviteSent`, `markInviteExpired``src/lib/repositories/bookings.js`Booking data access`createBooking`, `countBookingsForDateAndGroup``src/lib/repositories/academy.js`Academy waitlist operations`addToWaitlist`, `recordWaitlistResponse``src/lib/repositories/age-groups.js`Age group queries`listAgeGroups`, `getAgeGroupById`**Email**`src/lib/email/templates.js`Email template rendering and sending`sendBookingInvite`, `sendBookingConfirmation``src/lib/email/client.js`Resend API integration`sendEmailWithRetry`**Infrastructure**`src/lib/db/client.js`Database client helpers`getDB`, `generateId`, `generateToken`, `envQuery``src/lib/config.js`System configuration management`getSystemConfig`, `setSystemConfig``src/lib/db/schema.ts`TypeScript type definitions`Env`, `Enquiry`, `Booking`, `AgeGroup`**Database**`db/migrations/0001_initial_schema.sql`Initial database schemaTable definitions, indexes`db/seed.sql`Seed data for age groups and email templatesInitial records**Configuration**`wrangler.toml`Cloudflare deployment configD1 binding, KV binding, environment variables`package.json`NPM dependencies and scriptsBuild, test, migration scripts
**Sources:**[src/api/enquiry.ts1-10](https://github.com/egac-web/egac/blob/0ba542fc/src/api/enquiry.ts#L1-L10)[src/api/booking.ts1-10](https://github.com/egac-web/egac/blob/0ba542fc/src/api/booking.ts#L1-L10)[tests/unit/api.test.ts66-78](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/api.test.ts#L66-L78)[db/migrations/0001_initial_schema.sql1-17](https://github.com/egac-web/egac/blob/0ba542fc/db/migrations/0001_initial_schema.sql#L1-L17)

---

## Next Steps

This overview provides the foundation for understanding the EGAC platform. For deeper dives into specific areas:

- **Architecture details:** See [System Architecture](/egac-web/egac/1.1-system-architecture) for component relationships, dependency flow, and integration patterns
- **Data model:** See [Data Model](/egac-web/egac/1.2-data-model) for complete entity schemas, indexes, and relationship constraints
- **Getting started:** See [Installation & Setup](/egac-web/egac/2.1-installation-and-setup) to run the platform locally
- **API reference:** See [Public APIs](/egac-web/egac/5.1-public-apis), [Admin APIs](/egac-web/egac/5.2-admin-apis), and [Cron APIs](/egac-web/egac/5.3-cron-apis) for endpoint specifications
- **Business logic:** See [Age Group System](/egac-web/egac/6.1-age-group-system) and [Enquiry Routing](/egac-web/egac/6.2-enquiry-routing) for core domain logic
- **Scheduled jobs:** See [Cron Worker Architecture](/egac-web/egac/9.1-cron-worker-architecture) for background task details

**Sources:** All sections above
