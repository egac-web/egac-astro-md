---
title: "Data Model"
description: "This page provides a comprehensive reference for the database schema, entity relationships, and data access patterns"
pubDate: 2026-03-18
---
# Data Model
Relevant source files
- [db/migrations/0001_initial_schema.sql](https://github.com/egac-web/egac/blob/0ba542fc/db/migrations/0001_initial_schema.sql)
- [db/seed.sql](https://github.com/egac-web/egac/blob/0ba542fc/db/seed.sql)
- [docs/snagging.md](https://github.com/egac-web/egac/blob/0ba542fc/docs/snagging.md)
- [src/api/booking.ts](https://github.com/egac-web/egac/blob/0ba542fc/src/api/booking.ts)
- [src/api/enquiry.ts](https://github.com/egac-web/egac/blob/0ba542fc/src/api/enquiry.ts)
- [src/api/health.ts](https://github.com/egac-web/egac/blob/0ba542fc/src/api/health.ts)
- [src/lib/repositories/age-groups.ts](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/repositories/age-groups.ts)
- [src/lib/repositories/bookings.ts](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/repositories/bookings.ts)
- [tests/unit/api.test.ts](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/api.test.ts)
- [tests/unit/db.test.ts](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/db.test.ts)
- [vitest.config.ts](https://github.com/egac-web/egac/blob/0ba542fc/vitest.config.ts)
- [wrangler.toml](https://github.com/egac-web/egac/blob/0ba542fc/wrangler.toml)

This document provides a comprehensive reference for the EGAC platform's database schema, entity relationships, and data access patterns. It covers all tables in the D1 database, their fields, indexes, constraints, and the relationships between entities.

For information about how repositories interact with this data model, see [Repository Pattern](/egac-web/egac/7.1-repository-pattern). For details on how age groups are resolved and used in business logic, see [Age Group System](/egac-web/egac/6.1-age-group-system).

---

## Entity Relationship Model

The following diagram shows all database entities and their relationships using the exact table and field names from the schema:

```
generates

creates

joins

classified_by

has

results_in

belongs_to

assigned_to

used

for

enrolled_in

for

defines_capacity_for

enquiries

TEXT

id

PK

TEXT

email

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

TEXT

created_at

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

created_at

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

TEXT

session_time

INTEGER

capacity_per_session

INTEGER

active

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

created_at

TEXT

environment

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

INTEGER

active
```

**Sources:**[db/migrations/0001_initial_schema.sql1-239](https://github.com/egac-web/egac/blob/0ba542fc/db/migrations/0001_initial_schema.sql#L1-L239)[tests/unit/api.test.ts120-194](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/api.test.ts#L120-L194)

---

## Core Entities

### Enquiries Table

The `enquiries` table is the single source of truth for all user interactions with the platform. Every submission of the public enquiry form creates one enquiry record.
FieldTypeConstraintDescription`id`TEXTPRIMARY KEYGenerated with `generateId('enq')`, format: `enq_[20 hex chars]``created_at`TEXTNOT NULLISO 8601 timestamp of submission`name`TEXTnullableParent/guardian name (enquirer)`email`TEXTNOT NULL, indexedContact email (used for de-duplication checks)`phone`TEXTnullableContact phone number`dob`TEXTnullableAthlete's date of birth (ISO date string)`source`TEXTdefault: 'website'Origin of enquiry (website, walk-in, referral)`processed`INTEGERdefault: 0Set to 1 after invite/waitlist email sent`age_group_id`TEXTFK to age_groups, indexedResolved age group based on DOB`note`TEXTnullableAdmin notes or auto-populated info (e.g., "Enquiring about: [athlete_name]")`raw_payload`TEXTnullableJSON string of original form submission for audit`events`TEXTdefault: '[]'JSON array of event objects for event sourcing`invite_id`TEXTFK to invitesReference to generated invite (if taster path)`environment`TEXTdefault: 'production'Multi-tenancy: 'development', 'staging', 'production'
**Indexes:**

- `enquiries_email_idx` on `(email)`
- `enquiries_email_env_idx` on `(email, environment)` for environment-scoped lookups
- `enquiries_created_idx` on `(created_at)` for chronological queries
- `enquiries_processed_idx` on `(processed)` for filtering unprocessed enquiries
- `enquiries_age_group_idx` on `(age_group_id)` for age group reporting

The enquiry record persists throughout the athlete's journey and accumulates events in the `events` JSON field for audit trails.

**Sources:**[db/migrations/0001_initial_schema.sql19-45](https://github.com/egac-web/egac/blob/0ba542fc/db/migrations/0001_initial_schema.sql#L19-L45)[src/api/enquiry.ts131-155](https://github.com/egac-web/egac/blob/0ba542fc/src/api/enquiry.ts#L131-L155)[tests/unit/api.test.ts120-135](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/api.test.ts#L120-L135)

---

### Age Groups Table

The `age_groups` table is the single source of truth for all age group configuration. No age group logic is hardcoded—all rules are stored in D1 and queried at runtime.
FieldTypeConstraintDescription`id`TEXTPRIMARY KEYFormat: `ag_u11`, `ag_u13`, `ag_u15`, etc.`code`TEXTUNIQUE NOT NULLShort code (e.g., 'u11', 'u13')`label`TEXTNOT NULLDisplay label (e.g., 'U13', 'U15 and older')`booking_type`TEXTNOT NULL, CHECK**'taster'** or **'waitlist'**—determines enquiry routing`age_min_aug31`INTEGERNOT NULLMinimum age on 31 August of athletics season`age_max_aug31`INTEGERNOT NULLMaximum age on 31 August of athletics season`session_days`TEXTNOT NULLJSON array: `["Tuesday"]` or `["Saturday","Tuesday"]``session_time`TEXTnullableTime string (e.g., '18:30', '19:30')`capacity_per_session`INTEGERdefault: 2Max bookings per session (taster groups only)`season_capacity`INTEGERnullableTotal capacity for academy season (waitlist groups only)`season_start_month`INTEGERnullableMonth number (1-12) for academy season start`season_end_month`INTEGERnullableMonth number for academy season end`active`INTEGERdefault: 1, indexed1 = active, 0 = archived`sort_order`INTEGERdefault: 0Display order in admin UI`created_at`TEXTNOT NULLISO 8601 timestamp`updated_at`TEXTnullableISO 8601 timestamp of last update
**Key Design:**

- **`booking_type`** is the routing discriminator:

- `'taster'` → enquiry generates an `invites` record and sends booking invite email
- `'waitlist'` → enquiry creates an `academy_waitlist` record and sends waitlist confirmation
- Age is calculated on **31 August** of the athletics season, following UK Athletics standard
- Session days are stored as JSON to support multi-day groups (e.g., U11 Academy runs Saturday + Tuesday)

**Indexes:**

- `age_groups_code_idx` on `(code)`
- `age_groups_active_idx` on `(active)`
- `age_groups_booking_type_idx` on `(booking_type)`

**Sources:**[db/migrations/0001_initial_schema.sql72-111](https://github.com/egac-web/egac/blob/0ba542fc/db/migrations/0001_initial_schema.sql#L72-L111)[db/seed.sql12-53](https://github.com/egac-web/egac/blob/0ba542fc/db/seed.sql#L12-L53)[src/lib/repositories/age-groups.ts9-45](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/repositories/age-groups.ts#L9-L45)

---

### Invites Table

The `invites` table stores taster session booking invites. One invite is created per enquiry when the enquiry is routed to the **taster path** (`age_groups.booking_type = 'taster'`).
FieldTypeConstraintDescription`id`TEXTPRIMARY KEYGenerated with `generateId('inv')``token`TEXTUNIQUE NOT NULLSecure random token (48 hex chars) used in booking URLs`enquiry_id`TEXTFK to enquiriesParent enquiry`created_at`TEXTNOT NULLISO 8601 timestamp`sent_at`TEXTnullableTimestamp when email was successfully sent`accepted_at`TEXTnullableTimestamp when athlete booked a session`status`TEXTCHECK, default: 'pending'`'pending'`, `'sent'`, `'accepted'`, `'expired'``send_attempts`INTEGERdefault: 0Count of email send attempts (max 3)`last_send_error`TEXTnullableError message from last failed send`environment`TEXTdefault: 'production'Multi-tenancy environment
**Status Transitions:**

- `pending` → `sent` when email successfully delivered (via `markInviteSent`)
- `sent` → `accepted` when athlete completes booking (via `markInviteAccepted`)
- `sent` → `expired` after 14 days (via cron job `handleExpireInvites`)

**Token Lifecycle:**

- Generated with `generateToken(24)` (48 hex chars)
- Used in booking URLs: `https://egac.co.uk/booking?token=abc123...`
- Expires after 14 days from `created_at` (enforced by `isTokenExpired` check)
- Single-use: once `status = 'accepted'`, subsequent booking attempts are rejected

**Indexes:**

- `invites_token_idx` on `(token)` for fast lookup by booking URL
- `invites_enq_idx` on `(enquiry_id)`
- `invites_status_idx` on `(status)` for cron job queries
- `invites_env_idx` on `(environment)`

**Sources:**[db/migrations/0001_initial_schema.sql50-69](https://github.com/egac-web/egac/blob/0ba542fc/db/migrations/0001_initial_schema.sql#L50-L69)[src/lib/repositories/invites.ts1-150](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/repositories/invites.ts#L1-L150)[src/api/booking.ts36-44](https://github.com/egac-web/egac/blob/0ba542fc/src/api/booking.ts#L36-L44)

---

### Bookings Table

The `bookings` table records confirmed taster session bookings. One booking is created when an athlete uses their invite token to select a specific session date.
FieldTypeConstraintDescription`id`TEXTPRIMARY KEYGenerated with `generateId('bkg')``enquiry_id`TEXTFK to enquiries, indexedParent enquiry`invite_id`TEXTFK to invites, nullableInvite that generated this booking`age_group_id`TEXTFK to age_groups, indexedDenormalized for reporting stability`session_date`TEXTNOT NULL, indexedISO date (YYYY-MM-DD)`session_time`TEXTnullableTime string copied from age group`status`TEXTCHECK, default: 'confirmed'`'confirmed'`, `'cancelled'`, `'attended'`, `'no_show'``attendance_note`TEXTnullableAdmin notes when marking attendance`created_at`TEXTNOT NULLISO 8601 timestamp`updated_at`TEXTnullableISO 8601 timestamp of last status change`environment`TEXTdefault: 'production'Multi-tenancy environment
**Status Values:**

- `'confirmed'` - initial state after booking creation
- `'attended'` - athlete attended the session (marked by admin)
- `'no_show'` - athlete did not attend (marked by admin)
- `'cancelled'` - booking was cancelled (rare, usually handled by not showing up)

**Constraints:**

- `bookings_invite_date_uniq` UNIQUE INDEX on `(invite_id, session_date)` WHERE `invite_id IS NOT NULL`
- Prevents duplicate bookings for same invite + date
- Admin-created bookings (where `invite_id IS NULL`) bypass this constraint

**Indexes:**

- `bookings_date_idx` on `(session_date)` for calendar queries
- `bookings_ag_date_idx` on `(age_group_id, session_date)` for capacity checks
- `bookings_enquiry_idx` on `(enquiry_id)` for athlete history
- `bookings_status_idx` on `(status)` for reporting
- `bookings_env_idx` on `(environment)`

**Capacity Management:**
The `countBookingsForDateAndGroup` function checks current bookings against `age_groups.capacity_per_session` before allowing new bookings.

**Sources:**[db/migrations/0001_initial_schema.sql117-142](https://github.com/egac-web/egac/blob/0ba542fc/db/migrations/0001_initial_schema.sql#L117-L142)[src/lib/repositories/bookings.ts9-36](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/repositories/bookings.ts#L9-L36)[src/api/booking.ts76-97](https://github.com/egac-web/egac/blob/0ba542fc/src/api/booking.ts#L76-L97)

---

### Academy System: Seasons and Waitlist

The academy system consists of two tables: `academy_seasons` (defining program periods) and `academy_waitlist` (tracking athlete enrollment).

#### Academy Seasons Table
FieldTypeConstraintDescription`id`TEXTPRIMARY KEYGenerated with `generateId('acs')``code`TEXTUNIQUE NOT NULLSeason code (e.g., 'spring_2026', 'autumn_2026')`label`TEXTNOT NULLDisplay name (e.g., 'Spring 2026 Academy')`start_date`TEXTNOT NULLISO date of season start`end_date`TEXTNOT NULLISO date of season end`capacity`INTEGERNOT NULL, default: 40Total enrollment capacity`status`TEXTCHECK, default: 'upcoming'`'upcoming'`, `'open'`, `'closed'`, `'archived'``age_group_id`TEXTFK to age_groupsLinks to waitlist age group (e.g., `ag_u11`)`created_at`TEXTNOT NULLISO 8601 timestamp`updated_at`TEXTnullableISO 8601 timestamp
**Status Lifecycle:**

- `upcoming` → season not yet started
- `open` → accepting enrollments
- `closed` → season ended (triggered by cron job on 1st of month)
- `archived` → historical record

**Indexes:**

- `academy_seasons_status_idx` on `(status)`
- `academy_seasons_ag_idx` on `(age_group_id)`

#### Academy Waitlist Table
FieldTypeConstraintDescription`id`TEXTPRIMARY KEYGenerated with `generateId('awl')``enquiry_id`TEXTFK to enquiries, indexedParent enquiry`season_id`TEXTFK to academy_seasons, nullableSeason being considered for`token`TEXTUNIQUE NOT NULLSecure token for accept/decline URLs`position`INTEGERnullable, indexedQueue position within season (lower = earlier)`status`TEXTCHECK, default: 'waiting'See status values below`is_returning`INTEGERdefault: 01 if athlete attended previous season`rolled_over_from_season_id`TEXTnullablePrevious season ID if rolled over`invited_at`TEXTnullableTimestamp when place offered`responded_at`TEXTnullableTimestamp of accept/decline response`response`TEXTCHECK, nullable`'yes'` or `'no'``send_attempts`INTEGERdefault: 0Email send attempt count`last_send_error`TEXTnullableError from last send failure`created_at`TEXTNOT NULLISO 8601 timestamp`environment`TEXTdefault: 'production'Multi-tenancy environment
**Status Values:**

- `'waiting'` - on waitlist, not yet offered a place
- `'invited'` - place offered, awaiting response
- `'accepted'` - parent accepted the place
- `'declined'` - parent declined the place
- `'ineligible'` - no longer eligible (aged out)
- `'withdrawn'` - parent withdrew from waitlist

**Waitlist Flow:**

1. Enquiry created with `age_groups.booking_type = 'waitlist'`
2. `addToWaitlist` creates `academy_waitlist` entry with status `'waiting'`
3. Admin manually sends academy invite or cron job triggers season rollover
4. Status changes to `'invited'`, email sent with accept/decline URLs
5. Parent clicks URL, `handleAcademyRespond` updates `status` and `response`

**Rollover Logic:**
The monthly cron job (`handleAcademyRollover`) closes expired seasons and migrates waitlist entries:

- Copies `academy_waitlist` rows to new season
- Sets `is_returning = 1` for athletes who attended previous season
- Sets `rolled_over_from_season_id` to previous season ID

**Indexes:**

- `acad_wl_enquiry_idx` on `(enquiry_id)`
- `acad_wl_season_idx` on `(season_id)`
- `acad_wl_token_idx` on `(token)`
- `acad_wl_status_idx` on `(status)`
- `acad_wl_env_idx` on `(environment)`
- `acad_wl_position_idx` on `(season_id, position)` for queue ordering

**Sources:**[db/migrations/0001_initial_schema.sql145-203](https://github.com/egac-web/egac/blob/0ba542fc/db/migrations/0001_initial_schema.sql#L145-L203)[src/lib/repositories/academy.ts1-150](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/repositories/academy.ts#L1-L150)[src/api/academy.ts1-100](https://github.com/egac-web/egac/blob/0ba542fc/src/api/academy.ts#L1-L100)

---

### Email Templates Table

The `email_templates` table stores all email content, allowing runtime editing without code deployment.
FieldTypeConstraintDescription`id`TEXTPRIMARY KEYFormat: `tmpl_[template_key]``key`TEXTUNIQUE NOT NULLLookup key (e.g., 'booking_invite', 'academy_waitlist')`name`TEXTNOT NULLHuman-readable name for admin UI`subject`TEXTNOT NULLEmail subject line (may contain `{{variables}}`)`html`TEXTNOT NULLHTML email body with `{{variable}}` placeholders`text_content`TEXTnullablePlain text fallback`variables`TEXTdefault: '[]'JSON array of required variable names`active`INTEGERdefault: 11 = active, 0 = disabled`created_at`TEXTNOT NULLISO 8601 timestamp`updated_at`TEXTnullableISO 8601 timestamp
**Template Keys:**

- `booking_invite` - sent when enquiry routed to taster path
- `booking_confirmation` - sent after booking created
- `booking_reminder` - sent by cron job before session
- `academy_waitlist` - sent when enquiry routed to waitlist path
- `academy_invite` - sent when academy place offered
- `membership_invite` - sent after attendance marked

**Variable Substitution:**
Templates use `{{variable_name}}` placeholders. The `renderTemplate` function replaces these with context-specific values. Missing required variables throw errors to prevent malformed emails.

**Indexes:**

- `email_templates_key_idx` on `(key)`
- `email_templates_active_idx` on `(active)`

**Sources:**[db/migrations/0001_initial_schema.sql206-222](https://github.com/egac-web/egac/blob/0ba542fc/db/migrations/0001_initial_schema.sql#L206-L222)[db/seed.sql59-223](https://github.com/egac-web/egac/blob/0ba542fc/db/seed.sql#L59-L223)[src/lib/email/client.ts50-80](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/email/client.ts#L50-L80)

---

### Membership OTPs Table

The `membership_otps` table stores one-time tokens for membership form access after attendance is marked.
FieldTypeConstraintDescription`id`TEXTPRIMARY KEYGenerated with `generateId('motp')``enquiry_id`TEXTFK to enquiries, indexedParent enquiry`token`TEXTUNIQUE NOT NULLSecure random token (48 hex chars)`created_at`TEXTNOT NULLISO 8601 timestamp`used_at`TEXTnullableTimestamp when token was consumed`environment`TEXTdefault: 'production'Multi-tenancy environment
**Usage:**

1. Admin marks booking as `'attended'` in bookings calendar
2. System creates `membership_otps` entry and sends email
3. Email contains link: `https://egac.co.uk/membership?token=abc123...`
4. Token is valid for 7 days (enforced in membership form handler)
5. Once used, `used_at` is set and token is invalidated

**Indexes:**

- `membership_otps_token_idx` on `(token)`
- `membership_otps_enq_idx` on `(enquiry_id)`
- `membership_otps_env_idx` on `(environment)`

**Sources:**[db/migrations/0001_initial_schema.sql224-239](https://github.com/egac-web/egac/blob/0ba542fc/db/migrations/0001_initial_schema.sql#L224-L239)

---

## Data Flow: Enquiry to Booking

The following diagram shows the data flow from enquiry submission through to booking creation, mapping API handlers to repository functions and database tables:

```
booking_type = 'taster'

booking_type = 'waitlist'

expired

accepted

valid

full

available

POST /api/enquiry
(handleEnquiry)

validateEnquiryData

insertEnquiry()
→ enquiries table

resolveAgeGroup()
queries age_groups table

updateEnquiry()
set age_group_id

routeEnquiry()
checks booking_type

createInviteForEnquiry()
→ invites table

addToWaitlist()
→ academy_waitlist table

sendBookingInvite()
template: booking_invite

sendAcademyWaitlist()
template: academy_waitlist

markInviteSent()
update invites.status

markWaitlistInviteSent()

User receives email
with booking URL

User clicks link
POST /api/booking

getInviteByToken()
lookup invites table

Check status
& expiry

410 INVITE_EXPIRED

409 INVITE_USED

countBookingsForDateAndGroup()
compare to capacity_per_session

409 SLOT_FULL

createBooking()
→ bookings table

markInviteAccepted()
update invites.status

sendBookingConfirmation()
template: booking_confirmation

200 OK
booking confirmed
```

**Sources:**[src/api/enquiry.ts86-223](https://github.com/egac-web/egac/blob/0ba542fc/src/api/enquiry.ts#L86-L223)[src/api/booking.ts25-146](https://github.com/egac-web/egac/blob/0ba542fc/src/api/booking.ts#L25-L146)[src/lib/repositories/enquiries.ts1-100](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/repositories/enquiries.ts#L1-L100)[src/lib/repositories/invites.ts1-150](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/repositories/invites.ts#L1-L150)[src/lib/repositories/bookings.ts9-36](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/repositories/bookings.ts#L9-L36)

---

## Multi-Tenancy and Environment Isolation

The EGAC platform implements **multi-tenancy at the data layer** using an `environment` field on all core tables. This enables isolated development, staging, and production datasets within a single D1 database.

### Environment Field
TableEnvironment FieldDefault Value`enquiries``environment``'production'``invites``environment``'production'``bookings``environment``'production'``academy_waitlist``environment``'production'``membership_otps``environment``'production'`
**Note:** The `age_groups`, `academy_seasons`, and `email_templates` tables are **shared across all environments** (no `environment` field). This allows:

- Single source of truth for age group configuration
- Shared email template management
- Simplified academy season setup

### Environment Resolution

The `getEnv` function determines the current environment from the `Env.APP_ENV` binding:

```
export function getEnv(env: Env): string {
  return env.APP_ENV ?? 'production';
}
```

All repository queries automatically filter by environment using the `getEnv` helper. For example, `listEnquiries` in the enquiries repository:

```
const environment = getEnv(env);
const sql = `SELECT * FROM enquiries WHERE environment = ? ORDER BY created_at DESC`;
```

### Environment Values
ValuePurposeConfiguration`development`Local development with WranglerSet in `.dev.vars``staging`Pre-production testingSet in Pages project settings (staging branch)`production`Live systemSet in `wrangler.toml` vars
**Sources:**[src/lib/db/client.ts80-90](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/db/client.ts#L80-L90)[db/migrations/0001_initial_schema.sql36](https://github.com/egac-web/egac/blob/0ba542fc/db/migrations/0001_initial_schema.sql#L36-L36)[tests/unit/db.test.ts124-135](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/db.test.ts#L124-L135)

---

## Event Sourcing on Enquiries

The `enquiries.events` field implements **event sourcing** for audit trails. It stores a JSON array of event objects that record state changes and actions throughout the enquiry lifecycle.

### Event Structure

Each event is a JSON object with the following fields:

```
{
  type: string;           // Event type (e.g., 'booking_invite_sent')
  timestamp: string;      // ISO 8601 timestamp
  [key: string]: unknown; // Additional event-specific fields
}
```

### Event Types
Event TypeTriggered ByAdditional Fields`booking_invite_sent`Invite email successfully sent`invite_id``booking_created`Booking confirmed`booking_id`, `session_date``academy_waitlist_added`Added to academy waitlist-`academy_response_received`Parent accepted/declined place`response` ('yes' or 'no')`reminder_sent`Booking reminder sent`booking_id``membership_invite_sent`Membership form link sent`otp_id`
### Appending Events

The `appendEnquiryEvent` function adds events to the array:

```
// From src/lib/repositories/enquiries.ts
export async function appendEnquiryEvent(
  enquiryId: string,
  event: { type: string; [key: string]: unknown },
  env: Env
): Promise<void> {
  const enquiry = await getEnquiryById(enquiryId, env);
  if (!enquiry) throw new Error(`Enquiry not found: ${enquiryId}`);
 
  const events = JSON.parse(enquiry.events || '[]');
  events.push({
    ...event,
    timestamp: new Date().toISOString(),
  });
 
  await updateEnquiry(enquiryId, { events: JSON.stringify(events) }, env);
}
```

### Use Cases

1. **Idempotency:** Check if reminder already sent before sending another

```
const events = JSON.parse(enquiry.events);
const reminderSent = events.some(e => 
  e.type === 'reminder_sent' && e.booking_id === booking.id
);
```
2. **Audit Trails:** Track all interactions for admin dashboard timeline view
3. **Debugging:** Reconstruct full enquiry lifecycle for support issues

**Sources:**[db/migrations/0001_initial_schema.sql34](https://github.com/egac-web/egac/blob/0ba542fc/db/migrations/0001_initial_schema.sql#L34-L34)[src/lib/repositories/enquiries.ts80-110](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/repositories/enquiries.ts#L80-L110)[src/api/enquiry.ts175-196](https://github.com/egac-web/egac/blob/0ba542fc/src/api/enquiry.ts#L175-L196)

---

## JSON Fields and Serialized Data

Several tables use TEXT fields to store JSON-serialized data structures. This provides schema flexibility while maintaining relational integrity.

### Age Groups: session_days

Stores an array of weekday names:

```
["Tuesday"]
```

or

```
["Saturday", "Tuesday"]
```

**Usage:** Parsed in `getNextNSessionDates` to generate valid session dates.

### Enquiries: raw_payload

Stores the complete original form submission:

```
{
  "enquiry_for": "other",
  "enquirer_name": "Parent Name",
  "enquirer_email": "parent@example.com",
  "athlete_name": "Child Name",
  "athlete_dob": "2015-03-20"
}
```

**Purpose:** Preserve original data for audit and support escalations.

### Email Templates: variables

Stores array of required variable names:

```
["contact_name", "booking_url", "available_dates_list"]
```

**Purpose:** Template validation to prevent sending emails with missing placeholders.

### Enquiries: events

Stores event sourcing log (see [Event Sourcing](https://github.com/egac-web/egac/blob/0ba542fc/Event Sourcing) section above).

**Sources:**[db/migrations/0001_initial_schema.sql96](https://github.com/egac-web/egac/blob/0ba542fc/db/migrations/0001_initial_schema.sql#L96-L96)[src/api/enquiry.ts141-149](https://github.com/egac-web/egac/blob/0ba542fc/src/api/enquiry.ts#L141-L149)[src/lib/email/client.ts60-75](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/email/client.ts#L60-L75)

---

## Database Schema Types

The codebase defines TypeScript types for all database entities in [src/lib/db/schema.ts](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/db/schema.ts) These types mirror the SQL schema and provide compile-time type safety.

### Core Entity Types

```
// Enquiry record
export interface Enquiry {
  id: string;
  created_at: string;
  name: string | null;
  email: string;
  phone: string | null;
  dob: string | null;
  source: string;
  processed: number;
  age_group_id: string | null;
  note: string | null;
  raw_payload: string | null;
  events: string;
  invite_id: string | null;
  environment: 'development' | 'staging' | 'production';
}
 
// Invite record
export interface Invite {
  id: string;
  token: string;
  enquiry_id: string;
  created_at: string;
  sent_at: string | null;
  accepted_at: string | null;
  status: InviteStatus;
  send_attempts: number;
  last_send_error: string | null;
  environment: 'development' | 'staging' | 'production';
}
 
export type InviteStatus = 'pending' | 'sent' | 'accepted' | 'expired';
 
// Booking record
export interface Booking {
  id: string;
  enquiry_id: string;
  invite_id: string | null;
  age_group_id: string;
  session_date: string;
  session_time: string | null;
  status: BookingStatus;
  attendance_note: string | null;
  created_at: string;
  updated_at: string | null;
  environment: 'development' | 'staging' | 'production';
}
 
export type BookingStatus = 'confirmed' | 'cancelled' | 'attended' | 'no_show';
 
// Age group configuration
export interface AgeGroup {
  id: string;
  code: string;
  label: string;
  booking_type: 'taster' | 'waitlist';
  age_min_aug31: number;
  age_max_aug31: number;
  session_days: string; // JSON array
  session_time: string | null;
  capacity_per_session: number | null;
  season_capacity: number | null;
  season_start_month: number | null;
  season_end_month: number | null;
  active: number;
  sort_order: number;
  created_at: string;
  updated_at: string | null;
}
```

### Type Usage in Repositories

Repository functions use these types for return values and parameters:

```
// From src/lib/repositories/enquiries.ts
export async function insertEnquiry(
  data: Omit<Enquiry, 'id' | 'created_at'> & { id: string; created_at: string },
  env: Env
): Promise<Enquiry>
 
export async function getEnquiryById(
  id: string,
  env: Env
): Promise<Enquiry | null>
```

### Env Binding Type

The `Env` type defines all Cloudflare runtime bindings:

```
export interface Env {
  DB: D1Database;
  KV: KVNamespace;
  RESEND_API_KEY: string;
  ADMIN_TOKEN: string;
  ADMIN_EMAIL: string;
  EMAIL_FROM: string;
  CRON_SECRET: string;
  APP_ENV?: 'development' | 'staging' | 'production';
}
```

**Sources:**[src/lib/db/schema.ts1-200](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/db/schema.ts#L1-L200)[tests/unit/api.test.ts80-99](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/api.test.ts#L80-L99)

---

## Index Strategy

The schema includes strategic indexes to optimize common query patterns:

### Composite Indexes
Index NameColumnsPurpose`enquiries_email_env_idx``(email, environment)`Environment-scoped duplicate email checks`bookings_ag_date_idx``(age_group_id, session_date)`Capacity counting per age group per date`acad_wl_position_idx``(season_id, position)`Waitlist queue ordering within season
### Unique Constraints
ConstraintColumnsPurpose`invites.token``(token)`Globally unique booking URLs`academy_waitlist.token``(token)`Globally unique accept/decline URLs`age_groups.code``(code)`Unique age group codes`email_templates.key``(key)`Unique template lookup keys`bookings_invite_date_uniq``(invite_id, session_date)`Prevent duplicate bookings per invite
### Query Optimization Examples

**Count bookings for capacity check:**

```
-- Optimized by bookings_ag_date_idx
SELECT COUNT(*) FROM bookings 
WHERE age_group_id = ? 
  AND session_date = ? 
  AND status = 'confirmed' 
  AND environment = ?
```

**List upcoming bookings for calendar:**

```
-- Optimized by bookings_date_idx
SELECT * FROM bookings 
WHERE session_date >= ? 
  AND session_date <= ?
  AND status IN ('confirmed','attended','no_show')
  AND environment = ?
ORDER BY session_date ASC
```

**Find invite by token:**

```
-- Optimized by invites_token_idx (UNIQUE)
SELECT * FROM invites 
WHERE token = ?
```

**Sources:**[db/migrations/0001_initial_schema.sql39-142](https://github.com/egac-web/egac/blob/0ba542fc/db/migrations/0001_initial_schema.sql#L39-L142)[src/lib/repositories/bookings.ts62-78](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/repositories/bookings.ts#L62-L78)
