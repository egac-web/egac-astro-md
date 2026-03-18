# Public APIs
Relevant source files
- [src/api/booking.ts](https://github.com/egac-web/egac/blob/0ba542fc/src/api/booking.ts)
- [src/api/enquiry.ts](https://github.com/egac-web/egac/blob/0ba542fc/src/api/enquiry.ts)
- [src/api/health.ts](https://github.com/egac-web/egac/blob/0ba542fc/src/api/health.ts)
- [src/lib/business/age.ts](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/business/age.ts)
- [src/lib/business/enquiry.ts](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/business/enquiry.ts)
- [src/lib/business/templates.ts](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/business/templates.ts)
- [src/lib/business/tokens.ts](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/business/tokens.ts)
- [src/pages/index.astro](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/index.astro)
- [tests/unit/api.test.ts](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/api.test.ts)
- [tests/unit/business.test.ts](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/business.test.ts)

This document describes the public-facing, unauthenticated API endpoints that handle enquiry submission, taster session booking, academy waitlist responses, and health checks. These endpoints are invoked by the public enquiry form at [src/pages/index.astro](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/index.astro) and the booking page.

For admin-authenticated endpoints (enquiry management, booking administration, reporting), see [Admin APIs](/egac-web/egac/5.2-admin-apis). For scheduled job endpoints triggered by the cron worker, see [Cron APIs](/egac-web/egac/5.3-cron-apis).

---

## Endpoint Overview

The platform exposes four public API endpoints, all implemented as individual handler functions that are wired into the Astro Pages routing system:
EndpointMethodPurposeHandler`/api/enquiry`POSTSubmit new athlete enquiry`handleEnquiry` in [src/api/enquiry.ts](https://github.com/egac-web/egac/blob/0ba542fc/src/api/enquiry.ts)`/api/booking`POSTCreate taster session booking via invite token`handleBooking` in [src/api/booking.ts](https://github.com/egac-web/egac/blob/0ba542fc/src/api/booking.ts)`/api/academy/respond`POSTAccept or decline academy waitlist invitation`handleAcademyRespond` (tested in [tests/unit/api.test.ts537-659](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/api.test.ts#L537-L659))`/api/health`GETDatabase liveness and readiness check`handleHealth` in [src/api/health.ts](https://github.com/egac-web/egac/blob/0ba542fc/src/api/health.ts)
**Sources:**[tests/unit/api.test.ts1-699](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/api.test.ts#L1-L699)[src/api/enquiry.ts1-224](https://github.com/egac-web/egac/blob/0ba542fc/src/api/enquiry.ts#L1-L224)[src/api/booking.ts1-147](https://github.com/egac-web/egac/blob/0ba542fc/src/api/booking.ts#L1-L147)[src/api/health.ts1-41](https://github.com/egac-web/egac/blob/0ba542fc/src/api/health.ts#L1-L41)

---

## Public API Flow Architecture

```
D1 Database

Email System

Repository Layer

Business Logic

API Layer

Public User Journey

Submit form

Click invite link

Click waitlist link

POST JSON/form-data

POST JSON

POST JSON

Public User

Enquiry Form
(index.astro)

Booking Page
(invite link)

Academy Response Page
(waitlist token)

/api/enquiry
handleEnquiry

/api/booking
handleBooking

/api/academy/respond
handleAcademyRespond

/api/health
handleHealth

validateEnquiryData

resolveAgeGroup

routeEnquiry

validateBookingRequest

insertEnquiry

createInviteForEnquiry

addToWaitlist

createBooking

recordWaitlistResponse

sendBookingInvite

sendBookingConfirmation

sendAcademyWaitlist

enquiries

invites

bookings

academy_waitlist

age_groups
```

**Sources:**[tests/unit/api.test.ts66-401](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/api.test.ts#L66-L401)[src/api/enquiry.ts86-223](https://github.com/egac-web/egac/blob/0ba542fc/src/api/enquiry.ts#L86-L223)[src/api/booking.ts25-146](https://github.com/egac-web/egac/blob/0ba542fc/src/api/booking.ts#L25-L146)

---

## POST /api/enquiry

### Purpose

Accepts new athlete enquiries from the public enquiry form. This endpoint:

1. Validates contact and athlete details
2. Resolves the athlete's age group from date of birth using UK Athletics season rules
3. Routes to either taster session invite (age 10+) or academy waitlist (under 10)
4. Creates invite or waitlist entry and sends corresponding email
5. Always returns 201 success even if email fails (enquiry is never lost)

### Request Format

**Method:**`POST`**Path:**`/api/enquiry`**Content-Type:**`application/json` or `application/x-www-form-urlencoded`

The endpoint accepts two payload formats to support both JavaScript fetch and native form submission:

#### Self-Enquiry Payload
FieldTypeRequiredDescription`enquiry_for``"self"`NoDefaults to `"self"` if omitted`name``string`YesFull name of enquirer/athlete`email``string`YesValid email address`phone``string`NoContact phone number`dob``string`YesDate of birth in `YYYY-MM-DD` format`source``string`NoEnquiry source, defaults to `"website"`
#### On-Behalf Enquiry Payload
FieldTypeRequiredDescription`enquiry_for``"other"`YesIndicates parent/guardian enquiry`enquirer_name``string`YesParent/guardian name`enquirer_email``string`YesParent/guardian email`enquirer_phone``string`NoParent/guardian phone`athlete_name``string`YesAthlete's name`athlete_dob``string`YesAthlete's date of birth in `YYYY-MM-DD` format
**Legacy Compatibility:** The endpoint also accepts legacy field names (`name`, `email`, `phone`, `dob`) for backwards compatibility, mapped internally to the enquirer fields.

**Sources:**[src/api/enquiry.ts23-84](https://github.com/egac-web/egac/blob/0ba542fc/src/api/enquiry.ts#L23-L84)[tests/unit/api.test.ts217-330](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/api.test.ts#L217-L330)[src/pages/index.astro289-306](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/index.astro#L289-L306)

### Response Format

**Success (201 Created):**

```
{
  "ok": true,
  "message": "Enquiry received"
}
```

The response is intentionally generic—it does not reveal whether the enquiry was routed to taster or waitlist. The confirmation message in the UI is determined client-side based on the message text pattern.

**Sources:**[src/api/enquiry.ts222](https://github.com/egac-web/egac/blob/0ba542fc/src/api/enquiry.ts#L222-L222)[tests/unit/api.test.ts217-230](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/api.test.ts#L217-L230)

### Status Codes and Error Responses
StatusCodeMeaningExample Response201-Enquiry created successfully`{"ok": true, "message": "Enquiry received"}`400`INVALID_JSON`Request body malformed or unparseable`{"ok": false, "error": "Request body is required", "code": "INVALID_JSON"}`405-Method not allowed (only POST accepted)`{"ok": false, "error": "Method not allowed"}`422`VALIDATION_ERROR`Validation failed (missing fields, invalid email, implausible DOB)`{"ok": false, "error": "athlete_dob is required", "code": "VALIDATION_ERROR"}`
**Sources:**[tests/unit/api.test.ts254-282](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/api.test.ts#L254-L282)[src/api/enquiry.ts86-125](https://github.com/egac-web/egac/blob/0ba542fc/src/api/enquiry.ts#L86-L125)

### Enquiry Processing Flow

```
"Email System"
"D1 Database"
"routeEnquiry"
"resolveAgeGroup"
"insertEnquiry"
"validateEnquiryData"
"handleEnquiry"
"index.astro form"
"Email System"
"D1 Database"
"routeEnquiry"
"resolveAgeGroup"
"insertEnquiry"
"validateEnquiryData"
"handleEnquiry"
"index.astro form"
"Accepts JSON or form-encoded"
alt
["Validation fails"]
"Email failure logged but doesn't block"
alt
["booking_type = 'taster'"]
["booking_type = 'waitlist'"]
"Show confirmation message"
"POST /api/enquiry"
"parsePublicEnquiryBody"
"validateEnquiryData"
"throw Error"
"422 VALIDATION_ERROR"
"insertEnquiry(enquiry)"
"INSERT INTO enquiries"
"enquiry record"
"resolveAgeGroup(dob, date, allGroups)"
"AgeGroupConfig | null"
"routeEnquiry(ageGroup)"
"'taster'"
"createInviteForEnquiry"
"sendBookingInvite"
"'waitlist'"
"addToWaitlist"
"sendAcademyWaitlist"
"updateEnquiry(processed=1)"
"201 Created"
```

**Sources:**[src/api/enquiry.ts86-223](https://github.com/egac-web/egac/blob/0ba542fc/src/api/enquiry.ts#L86-L223)[tests/unit/api.test.ts199-401](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/api.test.ts#L199-L401)

### Age Group Resolution Logic

The endpoint uses UK Athletics age group rules (age on 31 August of the season) to classify enquiries:

1. Fetch all active age groups from `age_groups` table
2. Calculate athletics age: `ageForAthletics(dob, today)` from [src/lib/business/age.ts76-83](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/business/age.ts#L76-L83)
3. Match to age group where `age_min_aug31 <= age <= age_max_aug31`
4. Route based on `booking_type` field:

- `"taster"` → create invite with available Tuesday dates, send booking invite email
- `"waitlist"` → add to academy_waitlist table, send waitlist confirmation email

**Sources:**[src/api/enquiry.ts158-166](https://github.com/egac-web/egac/blob/0ba542fc/src/api/enquiry.ts#L158-L166)[src/lib/business/age.ts119-135](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/business/age.ts#L119-L135)[src/lib/business/enquiry.ts69-72](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/business/enquiry.ts#L69-L72)

### Side Effects

Each successful enquiry triggers the following database and email operations:
OperationTable/SystemConditionInsert enquiry record`enquiries`AlwaysUpdate `age_group_id``enquiries`If age group resolvedCreate invite`invites`If `booking_type = 'taster'`Send booking invite emailResend APIIf `booking_type = 'taster'`Add to waitlist`academy_waitlist`If `booking_type = 'waitlist'`Send waitlist emailResend APIIf `booking_type = 'waitlist'`Append event`enquiries.events` JSONOn email successUpdate `processed = 1``enquiries`Always (even if email fails)
**Important:** Email failures are logged but do not prevent the enquiry from being saved. The cron worker will retry failed invites via [handleRetryInvites](/egac-web/egac/9.2-invite-management-jobs).

**Sources:**[src/api/enquiry.ts168-220](https://github.com/egac-web/egac/blob/0ba542fc/src/api/enquiry.ts#L168-L220)[tests/unit/api.test.ts380-394](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/api.test.ts#L380-L394)

### Idempotency and Duplicate Handling

The endpoint allows multiple submissions for the same email address. Each creates a new enquiry record with a unique ID. This is intentional—the system does not prevent re-registration, as an athlete may enquire multiple times across seasons.

**Sources:**[tests/unit/api.test.ts283-302](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/api.test.ts#L283-L302)[src/api/enquiry.ts131-155](https://github.com/egac-web/egac/blob/0ba542fc/src/api/enquiry.ts#L131-L155)

### CORS Support

The endpoint responds to `OPTIONS` preflight requests with a 204 status and appropriate CORS headers, enabling cross-origin form submissions if needed in future.

**Sources:**[tests/unit/api.test.ts396-400](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/api.test.ts#L396-L400)[src/api/enquiry.ts87](https://github.com/egac-web/egac/blob/0ba542fc/src/api/enquiry.ts#L87-L87)

---

## POST /api/booking

### Purpose

Creates a taster session booking using an invite token received via email. This endpoint:

1. Validates the invite token and checks expiry/usage status
2. Re-resolves the athlete's age group for the selected session date
3. Validates the session date falls on a valid session day for the age group
4. Checks capacity constraints before creating the booking
5. Marks the invite as accepted and sends booking confirmation email

### Request Format

**Method:**`POST`**Path:**`/api/booking`**Content-Type:**`application/json`
FieldTypeRequiredDescription`token``string`YesInvite token from booking email link`date``string`YesSession date in `YYYY-MM-DD` format, must be a valid session day (e.g., Tuesday)`age_group_id``string`NoOptional override for age group selection (admin use)
**Example:**

```
{
  "token": "abc123def456ghi789",
  "date": "2026-06-10"
}
```

**Sources:**[src/api/booking.ts28-34](https://github.com/egac-web/egac/blob/0ba542fc/src/api/booking.ts#L28-L34)[tests/unit/api.test.ts431-441](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/api.test.ts#L431-L441)

### Response Format

**Success (200 OK):**

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

**Sources:**[src/api/booking.ts138-145](https://github.com/egac-web/egac/blob/0ba542fc/src/api/booking.ts#L138-L145)[tests/unit/api.test.ts431-441](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/api.test.ts#L431-L441)

### Status Codes and Error Responses
StatusCodeMeaningResponse Example200-Booking created successfullySee above400`INVALID_JSON`Request body malformed`{"ok": false, "error": "Request body is required"}`404`INVITE_NOT_FOUND`Token does not exist in database`{"ok": false, "error": "Invite not found"}`404`ENQUIRY_NOT_FOUND`Enquiry record missing (data integrity issue)`{"ok": false, "error": "Enquiry not found"}`405-Method not allowed`{"ok": false, "error": "Method not allowed"}`409`INVITE_USED`Invite already accepted (one booking per invite)`{"ok": false, "error": "This invite has already been used"}`409`DUPLICATE_BOOKING`Booking already exists for this invite + date`{"ok": false, "error": "You already have a booking for this date"}`409`SLOT_FULL`Session at capacity`{"ok": false, "error": "This session is full. Please choose a different date."}`410`INVITE_EXPIRED`Invite older than 14 days`{"ok": false, "error": "This invite has expired"}`422`MISSING_TOKEN`Token field missing from request`{"ok": false, "error": "Invite token is required"}`422`MISSING_DATE`Date field missing from request`{"ok": false, "error": "Session date is required"}`422`NO_AGE_GROUP`No matching age group for DOB + date`{"ok": false, "error": "No matching age group found for this date. Please contact the club."}`422`SLOT_INVALID`Date validation failed (past date, wrong day of week, etc.)`{"ok": false, "error": "Invalid booking"}`
**Sources:**[tests/unit/api.test.ts443-507](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/api.test.ts#L443-L507)[src/api/booking.ts36-74](https://github.com/egac-web/egac/blob/0ba542fc/src/api/booking.ts#L36-L74)

### Booking Validation Flow

```
null

found

'accepted'

'expired'

'pending' or 'sent'

true

false

null

found

null

group

invalid

valid

exists

null

>= capacity

< capacity

POST /api/booking

getInviteByToken(token)

404 INVITE_NOT_FOUND

Check invite.status

409 INVITE_USED

410 INVITE_EXPIRED

isInviteExpired(created_at)

410 INVITE_EXPIRED

getEnquiryById(enquiry_id)

404 ENQUIRY_NOT_FOUND

resolveAgeGroup(dob, date, tasterGroups)

422 NO_AGE_GROUP

validateBookingRequest(date, dob, ageGroup)

422 SLOT_INVALID

getBookingByInviteAndDate(invite_id, date)

409 DUPLICATE_BOOKING

countBookingsForDateAndGroup(date, age_group_id)

409 SLOT_FULL

createBooking(enquiry_id, invite_id, age_group_id, date)

markInviteAccepted(invite_id)

appendEnquiryEvent('booking_created')

sendBookingConfirmation (fire-and-forget)

200 OK with booking data
```

**Sources:**[src/api/booking.ts36-137](https://github.com/egac-web/egac/blob/0ba542fc/src/api/booking.ts#L36-L137)[tests/unit/api.test.ts407-533](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/api.test.ts#L407-L533)

### Age Group Re-resolution

The endpoint re-resolves the age group at booking time (not at enquiry time) because:

- The athlete's athletics age depends on the session date, not the enquiry date
- Season boundaries may have been crossed between enquiry and booking
- Different session dates in the same enquiry could theoretically fall in different age groups

The resolution logic filters to `booking_type = 'taster'` groups only, preventing accidentally booking waitlist-only groups.

**Sources:**[src/api/booking.ts50-70](https://github.com/egac-web/egac/blob/0ba542fc/src/api/booking.ts#L50-L70)[src/lib/business/age.ts119-135](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/business/age.ts#L119-L135)

### Capacity Checking

Capacity is enforced per age group per session date:

1. Fetch capacity from `age_groups.capacity_per_session` (falls back to system config `capacity_per_slot`)
2. Query `bookings` table for count where `session_date = :date AND age_group_id = :id AND status != 'cancelled'`
3. Reject if `currentCount >= capacity`

This means a Tuesday evening might have 2 U13 slots and 2 U15 slots independently.

**Sources:**[src/api/booking.ts81-87](https://github.com/egac-web/egac/blob/0ba542fc/src/api/booking.ts#L81-L87)[tests/unit/api.test.ts460-468](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/api.test.ts#L460-L468)

### Side Effects
OperationTable/SystemTimingInsert booking record`bookings`SynchronousUpdate `invite.status = 'accepted'``invites`SynchronousAppend `booking_created` event`enquiries.events`SynchronousSend confirmation emailResend APIAsynchronous (fire-and-forget)
**Important:** The confirmation email is sent asynchronously. Failures are logged but do not affect the HTTP response. Booking is always created successfully if validation passes.

**Sources:**[src/api/booking.ts89-136](https://github.com/egac-web/egac/blob/0ba542fc/src/api/booking.ts#L89-L136)[tests/unit/api.test.ts503-533](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/api.test.ts#L503-L533)

---

## POST /api/academy/respond

### Purpose

Records an athlete's response to an academy waitlist invitation (accept or decline). This endpoint:

1. Validates the waitlist token
2. Records the response (`yes` or `no`)
3. Updates waitlist entry status to `accepted` or `declined`
4. Appends event to enquiry trail for audit

### Request Format

**Method:**`POST`**Path:**`/api/academy/respond`**Content-Type:**`application/json`
FieldTypeRequiredDescription`token``string`YesWaitlist token from academy invitation email`response``"yes"` | `"no"`YesAthlete's response to waitlist invitation
**Example:**

```
{
  "token": "xyz789abc123",
  "response": "yes"
}
```

**Sources:**[tests/unit/api.test.ts571-598](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/api.test.ts#L571-L598)

### Response Format

**Success (200 OK):**

```
{
  "ok": true,
  "data": {
    "response": "yes"
  }
}
```

**Already Responded (200 OK, idempotent):**

```
{
  "ok": true,
  "data": {
    "response": "yes",
    "already_responded": true
  }
}
```

**Sources:**[tests/unit/api.test.ts571-643](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/api.test.ts#L571-L643)

### Status Codes and Error Responses
StatusCodeMeaning200-Response recorded successfully (or already recorded)404-Waitlist token not found422-Invalid response value (must be `"yes"` or `"no"`)
**Sources:**[tests/unit/api.test.ts600-620](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/api.test.ts#L600-L620)

### Idempotency

The endpoint is fully idempotent. If the token has already been responded to, it returns the previously recorded response without modifying the database. This prevents duplicate event logging and allows users to refresh the confirmation page without side effects.

**Sources:**[tests/unit/api.test.ts622-643](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/api.test.ts#L622-L643)

### Side Effects
OperationTableConditionUpdate `status`, `response`, `responded_at``academy_waitlist`On first response onlyAppend `academy_response_received` event`enquiries.events`On first response only
**Sources:**[tests/unit/api.test.ts645-658](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/api.test.ts#L645-L658)

---

## GET /api/health

### Purpose

Provides a liveness and readiness check for deployment verification and uptime monitoring. Used by GitHub Actions smoke tests after each deployment to Cloudflare Pages.

### Request Format

**Method:**`GET`**Path:**`/api/health`

No request body or query parameters required.

**Sources:**[src/api/health.ts17](https://github.com/egac-web/egac/blob/0ba542fc/src/api/health.ts#L17-L17)

### Response Format

**Healthy (200 OK):**

```
{
  "ok": true,
  "data": {
    "status": "ok",
    "environment": "production",
    "timestamp": "2026-06-10T12:30:45.123Z"
  }
}
```

**Unhealthy (503 Service Unavailable):**

```
{
  "ok": false,
  "error": "Database unavailable",
  "code": "DB_UNAVAILABLE"
}
```

**Sources:**[src/api/health.ts35-40](https://github.com/egac-web/egac/blob/0ba542fc/src/api/health.ts#L35-L40)[tests/unit/api.test.ts666-690](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/api.test.ts#L666-L690)

### Database Probe

The health check executes `SELECT 1` against the D1 database binding to verify connectivity. If the query fails, the endpoint returns 503, signaling to monitoring systems that the application is not ready to serve traffic.

**Sources:**[src/api/health.ts20-33](https://github.com/egac-web/egac/blob/0ba542fc/src/api/health.ts#L20-L33)

### Status Codes
StatusMeaning200Database reachable, system healthy405Method not allowed (only GET accepted)503Database connection failed
**Sources:**[tests/unit/api.test.ts665-697](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/api.test.ts#L665-L697)

---

## Common Response Structure

All public API endpoints follow a consistent response structure to simplify client-side error handling:

### Success Response Schema

```
{
  "ok": true,
  "data"?: object,      // Optional response payload
  "message"?: string    // Optional human-readable message
}
```

### Error Response Schema

```
{
  "ok": false,
  "error": string,      // Human-readable error message
  "code"?: string       // Machine-readable error code (e.g., "SLOT_FULL")
}
```

All error responses include the `ok: false` flag for consistent boolean checking in client code.

**Sources:**[src/api/helpers.ts](https://github.com/egac-web/egac/blob/0ba542fc/src/api/helpers.ts) (inferred from test patterns in [tests/unit/api.test.ts224-658](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/api.test.ts#L224-L658))

---

## Content-Type Support

Public endpoints accept multiple content types to support both JavaScript fetch calls and native HTML form submission:
Content-TypeEndpointsNotes`application/json`AllPrimary format, parsed via `request.json()``application/x-www-form-urlencoded``/api/enquiry` onlySupports native form POST without JavaScript`multipart/form-data``/api/enquiry` onlySupports file uploads if needed in future
The enquiry endpoint explicitly checks `Content-Type` header and delegates to `parsePublicEnquiryBody()` which handles both JSON and FormData parsing.

**Sources:**[src/api/enquiry.ts39-84](https://github.com/egac-web/egac/blob/0ba542fc/src/api/enquiry.ts#L39-L84)[tests/unit/api.test.ts232-252](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/api.test.ts#L232-L252)

---

## CORS Configuration

All public endpoints include CORS headers to enable cross-origin requests from external domains (e.g., if the enquiry form is embedded on a partner site):

```
Access-Control-Allow-Origin: *
Access-Control-Allow-Methods: GET, POST, OPTIONS
Access-Control-Allow-Headers: Content-Type

```

The `OPTIONS` preflight handler is implemented inline in each endpoint and returns 204 with no body.

**Sources:**[src/api/enquiry.ts87](https://github.com/egac-web/egac/blob/0ba542fc/src/api/enquiry.ts#L87-L87)[tests/unit/api.test.ts396-400](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/api.test.ts#L396-L400)

---

## Authentication

Public APIs are intentionally **unauthenticated**—no admin token, session cookie, or API key is required. Access control is implemented via:

- **Invite tokens:** Cryptographically secure 48-character hex strings (24 random bytes) prevent unauthorized booking access
- **Time-based expiry:** Invites expire after 14 days, enforced in code via `isInviteExpired()`
- **One-time use:** Invites are marked `accepted` after first booking, preventing reuse
- **Rate limiting:** Delegated to Cloudflare's edge network (not implemented in application code)

For admin operations, see [Admin APIs](/egac-web/egac/5.2-admin-apis) which require the `ADMIN_TOKEN` header.

**Sources:**[src/lib/business/tokens.ts45-49](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/business/tokens.ts#L45-L49)[src/api/booking.ts40-44](https://github.com/egac-web/egac/blob/0ba542fc/src/api/booking.ts#L40-L44)

---

## Error Logging

All endpoints log structured JSON to stdout for ingestion by Cloudflare's logging pipeline:

```
console.error(JSON.stringify({
  level: 'error',
  event: 'booking_confirmation_email_failed',
  booking_id: 'bkg_123',
  error: 'Resend timeout after 3 attempts'
}));
```

Logs include:

- `level`: `"info"`, `"error"`
- `event`: Machine-readable event type
- Contextual IDs (`enquiry_id`, `booking_id`, `invite_id`)
- Error details

**Sources:**[src/api/enquiry.ts177-184](https://github.com/egac-web/egac/blob/0ba542fc/src/api/enquiry.ts#L177-L184)[src/api/booking.ts117-135](https://github.com/egac-web/egac/blob/0ba542fc/src/api/booking.ts#L117-L135)[src/api/health.ts24-30](https://github.com/egac-web/egac/blob/0ba542fc/src/api/health.ts#L24-L30)

---

## Testing Strategy

All public endpoints have comprehensive unit test coverage in [tests/unit/api.test.ts](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/api.test.ts) Tests use Vitest with mocked repositories and email functions to isolate handler logic from I/O:
Test CategoryCoverageHappy pathAll endpoints, all valid input variationsValidation errorsMissing fields, invalid formats, constraint violationsBusiness rule violationsExpired tokens, duplicate bookings, capacity limitsEdge casesRepeat submissions, email failures, race conditionsMethod validationOPTIONS preflight, unsupported HTTP methods
**Sources:**[tests/unit/api.test.ts1-699](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/api.test.ts#L1-L699)