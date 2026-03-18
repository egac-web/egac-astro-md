---
title: "API Reference"
description: "This document provides a complete reference for all HTTP API endpoints exposed by the EGAC platform"
pubDate: 2026-03-18
---

# API Reference
Relevant source files
- [src/api/booking.ts](https://github.com/egac-web/egac/blob/0ba542fc/src/api/booking.ts)
- [src/api/enquiry.ts](https://github.com/egac-web/egac/blob/0ba542fc/src/api/enquiry.ts)
- [src/api/health.ts](https://github.com/egac-web/egac/blob/0ba542fc/src/api/health.ts)
- [src/env.d.ts](https://github.com/egac-web/egac/blob/0ba542fc/src/env.d.ts)
- [tests/unit/admin.test.ts](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/admin.test.ts)
- [tests/unit/api.test.ts](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/api.test.ts)
- [tests/unit/cron.test.ts](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/cron.test.ts)

This document provides a complete reference for all HTTP API endpoints exposed by the EGAC platform. It covers request/response formats, authentication requirements, status codes, and error handling patterns. For detailed information about individual endpoint groups, see [Public APIs](/egac-web/egac/5.1-public-apis), [Admin APIs](/egac-web/egac/5.2-admin-apis), and [Cron APIs](/egac-web/egac/5.3-cron-apis).

---

## API Architecture Overview

The EGAC platform exposes three distinct categories of HTTP endpoints, each with different authentication requirements and use cases:
CategoryBase PathAuthenticationPurposePublic APIs`/api/*`NonePublic-facing enquiry submission and booking flowsAdmin APIs`/api/admin/*`Bearer token or `x-admin-token` headerAdministrative operations for club staffCron APIs`/api/cron/*``CRON_SECRET` bearer tokenScheduled job endpoints triggered by external worker
### API Handler Architecture

```
Response Layer

Data Layer

Business Logic

Auth Layer

Handler Functions

Request Layer

HTTP Request

parseBody()
helpers.ts

corsPreflightResponse()

handleEnquiry()
src/api/enquiry.ts

handleBooking()
src/api/booking.ts

handleAcademyRespond()
src/api/academy.ts

handleHealth()
src/api/health.ts

handleAdmin*()
src/api/admin/*.ts

handle*()
src/api/cron/*.ts

checkAdminToken()
Bearer or x-admin-token

checkCronSecret()
Bearer CRON_SECRET

validateEnquiryData()
validateBookingRequest()

routeEnquiry()
resolveAgeGroup()

isInviteExpired()
generateSecureToken()

Repositories
enquiries.js
invites.js
bookings.js
academy.js

D1 Database

ok()
201/200 success

err()
4xx/5xx error
```

**Sources:**[src/api/enquiry.ts1-224](https://github.com/egac-web/egac/blob/0ba542fc/src/api/enquiry.ts#L1-L224)[src/api/booking.ts1-147](https://github.com/egac-web/egac/blob/0ba542fc/src/api/booking.ts#L1-L147)[src/api/helpers.ts](https://github.com/egac-web/egac/blob/0ba542fc/src/api/helpers.ts) (inferred), [tests/unit/api.test.ts1-699](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/api.test.ts#L1-L699)

---

## Common Patterns

### Request and Response Format

All endpoints accept and return JSON with the following conventions:

**Successful Response (2xx):**

```
{
  "ok": true,
  "data": { /* endpoint-specific data */ }
}
```

**Error Response (4xx/5xx):**

```
{
  "ok": false,
  "error": "Human-readable error message",
  "code": "ERROR_CODE"
}
```

**Sources:**[src/api/helpers.ts](https://github.com/egac-web/egac/blob/0ba542fc/src/api/helpers.ts) (inferred from usage), [tests/unit/api.test.ts224-229](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/api.test.ts#L224-L229)

### Content Type Support

Public endpoints accept both `application/json` and `application/x-www-form-urlencoded` for backwards compatibility with HTML forms:

```
HTML Form
application/x-www-form-urlencoded

JSON Body
application/json

parsePublicEnquiryBody()
src/api/enquiry.ts:39-84

handleEnquiry()
```

**Sources:**[src/api/enquiry.ts39-84](https://github.com/egac-web/egac/blob/0ba542fc/src/api/enquiry.ts#L39-L84)[tests/unit/api.test.ts232-252](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/api.test.ts#L232-L252)

### CORS Support

All public endpoints support CORS preflight requests:
MethodStatusHeaders`OPTIONS`204`Access-Control-Allow-Origin: *``Access-Control-Allow-Methods: GET, POST, OPTIONS``Access-Control-Allow-Headers: Content-Type`
**Sources:**[tests/unit/api.test.ts396-400](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/api.test.ts#L396-L400)

### Error Code Reference
HTTP StatusCodeMeaning400`INVALID_JSON`Request body is not valid JSON401`UNAUTHORIZED`Missing or invalid authentication token404`NOT_FOUND`Resource does not exist405-HTTP method not allowed409`INVITE_USED`Invite has already been accepted409`INVITE_EXPIRED`Invite has expired (>14 days old)409`SLOT_FULL`Session has reached capacity409`DUPLICATE_BOOKING`Booking already exists for this date410`INVITE_EXPIRED`Invite status is explicitly 'expired'422`VALIDATION_ERROR`Request data failed validation422`MISSING_TOKEN`Token parameter is required422`MISSING_DATE`Date parameter is required422`NO_AGE_GROUP`Cannot determine age group for athlete503`DB_UNAVAILABLE`Database connection failed
**Sources:**[src/api/enquiry.ts86-223](https://github.com/egac-web/egac/blob/0ba542fc/src/api/enquiry.ts#L86-L223)[src/api/booking.ts25-146](https://github.com/egac-web/egac/blob/0ba542fc/src/api/booking.ts#L25-L146)[tests/unit/api.test.ts254-501](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/api.test.ts#L254-L501)

---

## Authentication

### Public Endpoints

Public endpoints (`/api/enquiry`, `/api/booking`, `/api/academy/respond`, `/api/health`) require **no authentication**. They are designed for direct access by website visitors.

### Admin Endpoints

Admin endpoints require authentication via one of two methods:

**Method 1: Bearer Token (Production)**

```
Authorization: Bearer <ADMIN_TOKEN>
```

**Method 2: X-Admin-Token Header (Alternative)**

```
X-Admin-Token: <ADMIN_TOKEN>
```

**Development Shortcut:**
In non-production environments (`APP_ENV !== 'production'`), the special token `"dev"` is accepted as a valid admin token for local testing.

```
Yes

No

No

Yes

Yes

No

Admin API Request

Extract token from
Authorization or x-admin-token

APP_ENV === 'production'?

token === 'dev'?

token === ADMIN_TOKEN?

200 OK
Process request

401 UNAUTHORIZED
```

**Sources:**[tests/unit/admin.test.ts167-215](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/admin.test.ts#L167-L215)[src/env.d.ts1-14](https://github.com/egac-web/egac/blob/0ba542fc/src/env.d.ts#L1-L14)

### Cron Endpoints

Cron endpoints authenticate using the `CRON_SECRET` environment variable:

```
Authorization: Bearer <CRON_SECRET>
```

Requests without a valid `CRON_SECRET` return `401 UNAUTHORIZED` with code `UNAUTHORIZED`.

**Sources:**[tests/unit/cron.test.ts189-203](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/cron.test.ts#L189-L203)[tests/unit/cron.test.ts270-274](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/cron.test.ts#L270-L274)

---

## Public API Endpoints

### POST /api/enquiry

Submits a new enquiry for an athlete. Automatically determines age group eligibility and routes to either taster session invites or academy waitlist.

**Request Body (JSON or form-encoded):**
FieldTypeRequiredDescription`enquiry_for``'self' | 'other'`NoWhether enquirer is the athlete or a parent/guardian`enquirer_name` / `name``string`YesContact person name`enquirer_email` / `email``string`YesContact email (validated format)`enquirer_phone` / `phone``string`NoContact phone number`athlete_name``string`ConditionalRequired if `enquiry_for === 'other'``athlete_dob` / `dob``string`YesDate of birth in `YYYY-MM-DD` format`interest``string`NoFree text about athlete's interest`training_days``string[]`NoPreferred training days`source``string`NoDefaults to `"website"`
**On-Behalf Enquiry Example:**

```
{
  "enquiry_for": "other",
  "enquirer_name": "Parent Name",
  "enquirer_email": "parent@example.com",
  "enquirer_phone": "07700900000",
  "athlete_name": "Child Name",
  "athlete_dob": "2015-03-20"
}
```

**Processing Flow:**

```
taster

waitlist

POST /api/enquiry

parsePublicEnquiryBody()

validateEnquiryData()

insertEnquiry()
Generate enq_* ID

resolveAgeGroup()
from D1 age_groups

routeEnquiry()
booking_type?

booking_type = 'taster'

booking_type = 'waitlist'

createInviteForEnquiry()
Generate inv_* ID + token

sendBookingInvite()
Email with available dates

markInviteSent()
status = 'sent'

addToWaitlist()
Generate awl_* ID

sendAcademyWaitlist()
Waitlist confirmation

markWaitlistInviteSent()

201 Created
{ok: true, message: 'Enquiry received'}
```

**Success Response (201 Created):**

```
{
  "ok": true,
  "message": "Enquiry received"
}
```

**Email Failure Handling:**
The endpoint **always returns 201** even if email sending fails. The enquiry is persisted to the database, and failed invites can be retried via the `/api/cron/retry-invites` job or manually resent via admin interface.

**Error Responses:**
StatusCodeCondition400`INVALID_JSON`Body parsing failed405-Method is not POST422`VALIDATION_ERROR`Missing required fields or invalid email format
**Sources:**[src/api/enquiry.ts86-223](https://github.com/egac-web/egac/blob/0ba542fc/src/api/enquiry.ts#L86-L223)[tests/unit/api.test.ts199-401](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/api.test.ts#L199-L401)

---

### POST /api/booking

Creates a taster session booking using an invite token. Validates capacity, age eligibility, and session date before confirming.

**Request Body:**
FieldTypeRequiredDescription`token``string`YesInvite token from booking email`date``string`YesSession date in `YYYY-MM-DD` format`age_group_id``string`NoOptional override for admin use
**Example:**

```
{
  "token": "inv_abc123xyz",
  "date": "2026-06-10"
}
```

**Validation Chain:**

```
null

found

accepted

expired

pending/sent

>14 days

valid

null

found

invalid

valid

exists

none

at capacity

available

POST /api/booking
{token, date}

getInviteByToken(token)

invite.status?

isInviteExpired(created_at)

getEnquiryById(enquiry_id)

resolveAgeGroup(dob, date)

validateBookingRequest()
date, dob, ageGroup

getBookingByInviteAndDate()

countBookingsForDateAndGroup()

createBooking()
Generate bkg_* ID

markInviteAccepted()
status = 'accepted'

sendBookingConfirmation()
Email with calendar link

200 OK
{booking_id, session_date, ...}

404
INVITE_NOT_FOUND

409
INVITE_USED

410
INVITE_EXPIRED

422
NO_AGE_GROUP or SLOT_INVALID

409
DUPLICATE_BOOKING

409
SLOT_FULL
```

**Success Response (200 OK):**

```
{
  "ok": true,
  "data": {
    "booking_id": "bkg_abc123",
    "session_date": "2026-06-10",
    "session_time": "18:30",
    "age_group": "U13",
    "venue": "East Grinstead Leisure Centre",
    "message": "Booking confirmed. A confirmation email is on its way."
  }
}
```

**Error Responses:**
StatusCodeCondition404`INVITE_NOT_FOUND`Token does not exist in database409`INVITE_USED`Invite already accepted410`INVITE_EXPIRED`Invite older than 14 days or status is `'expired'`422`MISSING_TOKEN`Token not provided422`MISSING_DATE`Date not provided422`NO_AGE_GROUP`Cannot determine age group for athlete on this date422`SLOT_INVALID`Date validation failed (past date, wrong day of week, etc.)409`DUPLICATE_BOOKING`Booking already exists for this invite and date409`SLOT_FULL`Session has reached capacity
**Side Effects:**

1. Creates `bookings` row with status `'confirmed'`
2. Updates `invites.status` to `'accepted'`
3. Updates `invites.accepted_at` timestamp
4. Appends `booking_created` event to `enquiries.events` JSON
5. Sends confirmation email with calendar `.ics` attachment (non-blocking)

**Sources:**[src/api/booking.ts25-146](https://github.com/egac-web/egac/blob/0ba542fc/src/api/booking.ts#L25-L146)[tests/unit/api.test.ts407-534](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/api.test.ts#L407-L534)

---

### POST /api/academy/respond

Records a waitlist response (accept or decline) for an academy season invitation.

**Request Body:**
FieldTypeRequiredDescription`token``string`YesAcademy waitlist token from email`response``'yes' | 'no'`YesWhether accepting or declining the spot
**Example:**

```
{
  "token": "awl_token_xyz",
  "response": "yes"
}
```

**Processing:**

```
null

found

yes

no

invalid

valid

POST /api/academy/respond
{token, response}

getWaitlistEntryByToken(token)

Already responded?

response === 'yes' or 'no'?

recordWaitlistResponse()
Update status + response + responded_at

appendEnquiryEvent()
academy_response_received

200 OK
{response}

200 OK
{already_responded: true}

404
Token not found

422
Invalid response
```

**Success Response (200 OK):**

```
{
  "ok": true,
  "data": {
    "response": "yes"
  }
}
```

**Idempotent Response (already responded):**

```
{
  "ok": true,
  "data": {
    "response": "yes",
    "already_responded": true
  }
}
```

**Error Responses:**
StatusCodeCondition404`NOT_FOUND`Token does not exist422`VALIDATION_ERROR`Response is not `'yes'` or `'no'`
**Side Effects:**

1. Updates `academy_waitlist.status` to `'accepted'` or `'declined'`
2. Updates `academy_waitlist.response` to `'yes'` or `'no'`
3. Updates `academy_waitlist.responded_at` timestamp
4. Appends `academy_response_received` event to `enquiries.events`

**Sources:**[tests/unit/api.test.ts540-659](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/api.test.ts#L540-L659)

---

### GET /api/health

Health check endpoint for monitoring and deployment verification.

**Request:**

```
GET /api/health
```

**Success Response (200 OK):**

```
{
  "ok": true,
  "data": {
    "status": "ok",
    "environment": "production",
    "timestamp": "2026-06-09T10:30:00.000Z"
  }
}
```

**Error Response (503 Service Unavailable):**
Returned when D1 database query fails.

```
{
  "ok": false,
  "error": "Database unavailable",
  "code": "DB_UNAVAILABLE"
}
```

**Implementation:**
Executes `SELECT 1` query against D1 to verify database connectivity. Used by GitHub Actions smoke tests and uptime monitoring.

**Sources:**[src/api/health.ts17-40](https://github.com/egac-web/egac/blob/0ba542fc/src/api/health.ts#L17-L40)[tests/unit/api.test.ts665-698](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/api.test.ts#L665-L698)

---

## Admin API Endpoints

Admin endpoints provide full CRUD operations for managing enquiries, bookings, invites, system configuration, and academy seasons. All admin endpoints require authentication (see [Authentication](https://github.com/egac-web/egac/blob/0ba542fc/Authentication)).

For detailed documentation of admin endpoints, see [Admin APIs](/egac-web/egac/5.2-admin-apis).

**Admin Endpoint Summary:**
MethodPathPurposeGET`/api/admin/enquiries`List enquiries with pagination and filteringPOST`/api/admin/enquiries`Manually create enquiryGET`/api/admin/enquiries/:id`Get enquiry detail with events and bookingsPATCH`/api/admin/enquiries/:id`Update enquiry notePOST`/api/admin/invites/resend`Resend or create booking inviteGET`/api/admin/bookings`List upcoming bookingsPOST`/api/admin/bookings/:id/attendance`Mark attendance for bookingGET`/api/admin/config`Get system configurationPATCH`/api/admin/config`Update system configurationGET`/api/admin/age-groups`List age groupsPATCH`/api/admin/age-groups/:id`Update age group
**Sources:**[tests/unit/admin.test.ts1-583](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/admin.test.ts#L1-L583)

---

## Cron API Endpoints

Cron endpoints are triggered by the separate `egac-cron` Cloudflare Worker on scheduled intervals. All cron endpoints require `CRON_SECRET` authentication (see [Authentication](https://github.com/egac-web/egac/blob/0ba542fc/Authentication)).

For detailed documentation of cron jobs and their schedules, see [Cron APIs](/egac-web/egac/5.3-cron-apis) and [Scheduled Jobs](/egac-web/egac/9-scheduled-jobs).

**Cron Endpoint Summary:**
PathSchedulePurpose`/api/cron/expire-invites`Daily 09:00 UTCMark invites older than 14 days as expired`/api/cron/retry-invites`Every 30 minutesRetry sending failed invite emails (max 3 attempts)`/api/cron/send-reminders`Tuesday 18:00 UTCSend reminder emails for sessions 7 days ahead`/api/cron/academy-rollover`1st of month 09:00 UTCClose expired seasons and roll waitlist to new season
**Cron Authentication Flow:**

```
"D1 Database"
"Cron Handler"
"Auth Middleware"
"egac-cron Worker"
"D1 Database"
"Cron Handler"
"Auth Middleware"
"egac-cron Worker"
alt
[Valid CRON_SECRET]
[Invalid or missing secret]
POST /api/cron/*
Authorization: Bearer CRON_SECRET
Forward request
Execute job logic
Results
200 OK {ok: true, data: {...}}
401 UNAUTHORIZED {ok: false, code: 'UNAUTHORIZED'}
```

**Sources:**[tests/unit/cron.test.ts1-607](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/cron.test.ts#L1-L607)

---

## Environment Configuration

The following environment variables control API behavior:
VariableTypeRequiredDescription`DB``D1Database`YesCloudflare D1 database binding`KV``KVNamespace`YesCloudflare KV namespace binding for config`RESEND_API_KEY``string`YesResend API key for sending emails`ADMIN_TOKEN``string`YesSecret token for admin authentication`ADMIN_EMAIL``string`YesAdmin email address for notifications`EMAIL_FROM``string`YesSender email address`CRON_SECRET``string`YesSecret token for cron job authentication`APP_ENV``string`YesEnvironment name (`development`, `staging`, `production`)
**Type Definition:**

```
interface Env {
  DB: D1Database;
  KV: KVNamespace;
  RESEND_API_KEY: string;
  ADMIN_TOKEN: string;
  ADMIN_EMAIL: string;
  EMAIL_FROM: string;
  CRON_SECRET: string;
  APP_ENV: string;
}
```

**Sources:**[src/env.d.ts1-14](https://github.com/egac-web/egac/blob/0ba542fc/src/env.d.ts#L1-L14)[tests/unit/api.test.ts86-99](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/api.test.ts#L86-L99)

---

## Rate Limiting and Abuse Prevention

The platform relies on Cloudflare's built-in DDoS protection and rate limiting. Additional considerations:

1. **Duplicate Enquiry Prevention:** While the same email can submit multiple enquiries, each submission creates a new database record for audit purposes. [src/api/enquiry.ts131-155](https://github.com/egac-web/egac/blob/0ba542fc/src/api/enquiry.ts#L131-L155)
2. **Invite Expiry:** All booking invites automatically expire after 14 days to prevent stale links from accumulating. [tests/unit/cron.test.ts246-250](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/cron.test.ts#L246-L250)
3. **Capacity Enforcement:** Session capacity checks are enforced at booking time with row-level counting to prevent overbooking. [src/api/booking.ts80-87](https://github.com/egac-web/egac/blob/0ba542fc/src/api/booking.ts#L80-L87)
4. **Idempotency:** Duplicate bookings for the same invite and date return `409 DUPLICATE_BOOKING` rather than creating duplicate records. [src/api/booking.ts77-78](https://github.com/egac-web/egac/blob/0ba542fc/src/api/booking.ts#L77-L78)

---

## Error Handling Philosophy

All API handlers follow consistent error handling patterns:

1. **Database errors never block enquiry submission:** Email send failures return 201 and log errors for retry. [src/api/enquiry.ts199-220](https://github.com/egac-web/egac/blob/0ba542fc/src/api/enquiry.ts#L199-L220)
2. **Email failures are non-blocking:** Confirmation emails use fire-and-forget pattern with error logging. [src/api/booking.ts108-136](https://github.com/egac-web/egac/blob/0ba542fc/src/api/booking.ts#L108-L136)
3. **Validation errors return 422:** All validation failures include human-readable error messages and structured error codes. [src/api/enquiry.ts122-125](https://github.com/egac-web/egac/blob/0ba542fc/src/api/enquiry.ts#L122-L125)
4. **Authentication errors return 401:** Missing or invalid tokens always return 401 with `UNAUTHORIZED` code. [tests/unit/admin.test.ts172-182](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/admin.test.ts#L172-L182)
5. **Resource not found returns 404:** Non-existent enquiries, invites, or tokens return 404. [tests/unit/api.test.ts443-448](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/api.test.ts#L443-L448)

**Sources:**[src/api/enquiry.ts86-223](https://github.com/egac-web/egac/blob/0ba542fc/src/api/enquiry.ts#L86-L223)[src/api/booking.ts25-146](https://github.com/egac-web/egac/blob/0ba542fc/src/api/booking.ts#L25-L146)[tests/unit/api.test.ts1-699](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/api.test.ts#L1-L699)
