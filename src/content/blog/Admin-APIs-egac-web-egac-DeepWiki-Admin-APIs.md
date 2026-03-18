# Admin APIs
Relevant source files
- [src/pages/admin/bookings.astro](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/bookings.astro)
- [src/pages/admin/enquiries.astro](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/enquiries.astro)
- [src/pages/admin/reports.astro](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/reports.astro)
- [tests/unit/admin.test.ts](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/admin.test.ts)
- [tests/unit/cron.test.ts](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/cron.test.ts)

## Purpose and Scope

Admin APIs provide authenticated endpoints for club administrators to manage enquiries, invites, and bookings. These endpoints enforce token-based authentication and enable CRUD operations on the core entities that flow through the registration and booking pipeline.

For public-facing endpoints (enquiry submission, booking creation), see [Public APIs](/egac-web/egac/5.1-public-apis). For scheduled job endpoints, see [Cron APIs](/egac-web/egac/5.3-cron-apis).

---

## Authentication System

All admin endpoints require authentication via token validation. The system supports two authentication mechanisms:
Header TypeFormatExampleBearer Token`Authorization: Bearer {token}``Authorization: Bearer secret_token`Direct Token`x-admin-token: {token}``x-admin-token: secret_token`
### Token Validation

The `checkAdminToken` function (referenced in [src/lib/auth/admin.js](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/auth/admin.js)) validates tokens against the `ADMIN_TOKEN` environment variable. A special development token `"dev"` is accepted when `APP_ENV=development` but rejected in production environments.

**Authentication Flow:**

```
Env
checkAdminToken
Handler
Client
Env
checkAdminToken
Handler
Client
alt
["Token matches
ADMIN_TOKEN"]
["Token is 'dev' and
APP_ENV=development"]
["Invalid or missing token"]
"Request with token"
"checkAdminToken(token, env)"
"Check ADMIN_TOKEN"
"Valid"
"true"
"Process request"
"200 OK + data"
"Valid (dev mode)"
"true"
"Process request"
"200 OK + data"
"false"
"401 Unauthorized"
```

**Sources:**[tests/unit/admin.test.ts167-215](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/admin.test.ts#L167-L215)[src/pages/admin/bookings.astro14-17](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/bookings.astro#L14-L17)

---

## Enquiry Management APIs

### GET /api/admin/enquiries

Lists enquiries with filtering, search, and pagination support.

**Handler:**`handleAdminEnquiries` in [src/api/admin/enquiries.js](https://github.com/egac-web/egac/blob/0ba542fc/src/api/admin/enquiries.js)

**Query Parameters:**
ParameterTypeDescriptionValid Values`status`stringFilter by processing status`all`, `pending`, `processed`, `academy``search`stringSearch term for name/emailAny string`page`numberPage number (1-indexed)Positive integer
**Response:**

```
{
  "ok": true,
  "data": {
    "enquiries": [
      {
        "id": "enq_abc123",
        "email": "athlete@example.com",
        "name": "John Smith",
        "created_at": "2025-01-15T10:00:00.000Z",
        "age_group_id": "ag_u13",
        "invite_id": "inv_xyz789",
        "processed": 1,
        ...
      }
    ],
    "page": 1,
    "hasNextPage": true
  }
}
```

**Pagination Logic:**

- `PAGE_SIZE = 20` (defined in handler)
- Query fetches `PAGE_SIZE + 1` rows
- If result count > `PAGE_SIZE`, `hasNextPage = true`
- Only first `PAGE_SIZE` rows returned

**Status Filter Values:**

```
'all'
(no filter)

'pending'
(processed=0)

'processed'
(processed=1)

'academy'
(source contains 'academy'
or booking_type='waitlist')
```

**Sources:**[tests/unit/admin.test.ts221-269](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/admin.test.ts#L221-L269)[src/pages/admin/enquiries.astro22-44](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/enquiries.astro#L22-L44)

---

### POST /api/admin/enquiries

Creates a new enquiry or returns an existing one by email. Used for manual registration by administrators.

**Handler:**`handleAdminEnquiries` in [src/api/admin/enquiries.js](https://github.com/egac-web/egac/blob/0ba542fc/src/api/admin/enquiries.js)

**Request Body:**

```
{
  "name": "Jane Doe",
  "email": "jane@example.com",
  "dob": "2015-06-15",
  "phone": "+441234567890"
}
```

**Required Fields:**

- `email` (must be valid email format)

**Optional Fields:**

- `name`, `dob`, `phone`

**Response (New Enquiry):**

```
{
  "ok": true,
  "data": {
    "enquiry_id": "enq_new123",
    "already_existed": false
  }
}
```

*Status: 201 Created*

**Response (Existing Enquiry):**

```
{
  "ok": true,
  "data": {
    "enquiry_id": "enq_existing456",
    "already_existed": true
  }
}
```

*Status: 200 OK*

**Validation Errors:**
ConditionStatusResponseMissing email422`{ ok: false, code: "VALIDATION_ERROR", error: "..." }`Invalid email format422`{ ok: false, code: "VALIDATION_ERROR", error: "..." }`
**Behavior:**

- Sets `source: 'admin'` on created enquiries
- Checks `getEnquiryByEmail` before creating
- Returns existing enquiry without modification if email already registered

**Sources:**[tests/unit/admin.test.ts275-336](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/admin.test.ts#L275-L336)

---

### GET /api/admin/enquiries/:id

Retrieves detailed information about a specific enquiry, including its event history, associated invite, and bookings.

**Handler:**`handleAdminEnquiryDetail` in [src/api/admin/enquiry-detail.js](https://github.com/egac-web/egac/blob/0ba542fc/src/api/admin/enquiry-detail.js)

**URL Parameters:**

- `:id` - Enquiry ID (e.g., `enq_abc123`)

**Response:**

```
{
  "ok": true,
  "data": {
    "enquiry": {
      "id": "enq_abc123",
      "email": "athlete@example.com",
      "name": "John Smith",
      "events": "[{\"type\":\"booking_invite_sent\",\"timestamp\":\"...\"}]",
      "invite_id": "inv_xyz789",
      ...
    },
    "events": [
      {
        "type": "reminder_sent",
        "timestamp": "2025-01-20T08:00:00.000Z",
        "booking_id": "bkg_001"
      },
      {
        "type": "booking_invite_sent",
        "timestamp": "2025-01-15T08:00:00.000Z"
      }
    ],
    "invite": {
      "id": "inv_xyz789",
      "status": "sent",
      "token": "tok_secure123",
      "created_at": "2025-01-15T08:00:00.000Z",
      "sent_at": "2025-01-15T08:05:00.000Z",
      ...
    },
    "bookings": [
      {
        "id": "bkg_001",
        "session_date": "2025-02-04",
        "status": "confirmed",
        "age_group_id": "ag_u13",
        ...
      }
    ]
  }
}
```

**Event Processing:**

- Events parsed from `enquiry.events` JSON field
- Sorted by `timestamp` descending (newest first)
- Includes event types: `booking_invite_sent`, `reminder_sent`, `invite_expired`, `invite_resent`, etc.

**Enrichment Process:**

```
Not found

getEnquiryById(id)

Parse events JSON

getInviteById(invite_id)

listBookingsByEnquiry(enquiry_id)

Sort by timestamp DESC

Build response object

Return 200 OK

Return 404
```

**Error Responses:**
ConditionStatusResponseEnquiry not found404`{ ok: false, code: "NOT_FOUND" }`
**Sources:**[tests/unit/admin.test.ts342-419](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/admin.test.ts#L342-L419)[src/pages/admin/enquiries.astro54-83](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/enquiries.astro#L54-L83)

---

### PATCH /api/admin/enquiries/:id

Updates the administrative note field on an enquiry.

**Handler:**`handleAdminEnquiryDetail` in [src/api/admin/enquiry-detail.js](https://github.com/egac-web/egac/blob/0ba542fc/src/api/admin/enquiry-detail.js)

**URL Parameters:**

- `:id` - Enquiry ID

**Request Body:**

```
{
  "note": "Called parent - confirmed availability"
}
```

**Response:**

```
{
  "ok": true,
  "data": {
    "enquiry": {
      "id": "enq_abc123",
      "note": "Called parent - confirmed availability",
      ...
    }
  }
}
```

**Validation:**

- Only `note` field is updatable via this endpoint
- Other fields are ignored if present in request body

**Sources:**[tests/unit/admin.test.ts425-457](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/admin.test.ts#L425-L457)

---

## Invite Management APIs

### POST /api/admin/invites/resend

Resends or creates a booking invite for an enquiry. Intelligently reuses existing invites when possible.

**Handler:**`handleAdminInviteResend` in [src/api/admin/invites.js](https://github.com/egac-web/egac/blob/0ba542fc/src/api/admin/invites.js)

**Request Body:**

```
{
  "enquiry_id": "enq_abc123"
}
```

**Response:**

```
{
  "ok": true,
  "data": {
    "invite_id": "inv_xyz789",
    "reused": true
  }
}
```

**Invite Reuse Logic:**

```
Not found

Found

'pending' or 'sent'

'expired' or 'failed'

null (no invite)

POST /api/admin/invites/resend

getEnquiryById(enquiry_id)

404 NOT_FOUND

getInviteByEnquiryId(enquiry_id)

Check invite.status

Reuse existing invite

createInviteForEnquiry()

sendBookingInvite()

markInviteSent()

appendEnquiryEvent('invite_resent')

200 OK
{reused: true/false}

reused: false

reused: true
```

**Reuse Criteria:**
Existing Invite StatusAction`reused` Value`pending`Reuse`true``sent`Reuse`true``expired`Create new`false``failed`Create new`false``accepted`Reuse`true`None existsCreate new`false`
**Email Sending:**

- Fetches available dates via `getNextNTuesdayDates`
- Sends via `sendBookingInvite` with retry logic
- Marks invite as sent regardless of email success (optimistic)
- Appends `invite_resent` event to enquiry

**Error Responses:**
ConditionStatusResponseEnquiry not found404`{ ok: false, code: "NOT_FOUND" }`Missing enquiry_id422`{ ok: false, code: "VALIDATION_ERROR" }`
**Sources:**[tests/unit/admin.test.ts463-582](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/admin.test.ts#L463-L582)

---

## Booking Management APIs

### POST /api/admin/bookings/:id/attendance

Records attendance status for a booking and optionally sends a membership invite.

**Handler:** Referenced in [src/pages/admin/bookings.astro209-214](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/bookings.astro#L209-L214)

**URL Parameters:**

- `:id` - Booking ID (e.g., `bkg_abc123`)

**Request Body:**

```
{
  "status": "attended",
  "send_membership_link": true
}
```

**Status Values:**

- `attended` - Athlete attended the session
- `no_show` - Athlete did not attend

**Response:**

```
{
  "ok": true,
  "data": {
    "booking_id": "bkg_abc123",
    "status": "attended",
    "membership_sent": true
  }
}
```

**Attendance Recording Flow:**

```
"membershipRepo.createOTP"
"sendMembershipInvite"
"bookingRepo.updateBookingStatus"
"POST /api/admin/bookings/:id/attendance"
"Admin UI"
"membershipRepo.createOTP"
"sendMembershipInvite"
"bookingRepo.updateBookingStatus"
"POST /api/admin/bookings/:id/attendance"
"Admin UI"
alt
["send_membership_link = true"]
"User clicks 'Mark attended'"
"Confirm: Send membership link?"
"POST {status: 'attended', send_membership_link: true}"
"updateBookingStatus(id, 'attended')"
"Updated booking"
"createOTP(enquiry_id)"
"otp_token"
"sendMembershipInvite(enquiry, token)"
"Email sent"
"200 OK"
"Update badge to 'Attended'"
"Hide attendance buttons"
```

**Membership Invite Logic:**

- Only triggered when `send_membership_link: true` in request
- Creates a one-time password token in `membership_otps` table
- Sends email with secure link to membership registration form
- Email failure does not block attendance recording (optimistic)

**UI Integration:**

The admin bookings interface ([src/pages/admin/bookings.astro179-252](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/bookings.astro#L179-L252)) implements AJAX updates:

1. User clicks "Mark attended" or "No show" button
2. If marking attended, confirms membership invite preference
3. POSTs to attendance endpoint
4. On success:

- Updates status badge inline
- Removes attendance control buttons
- Shows toast notification

**Sources:**[src/pages/admin/bookings.astro179-252](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/bookings.astro#L179-L252)

---

## API Architecture Overview

```
D1 Database

Email Subsystem

Repository Layer

Authentication

Handler Layer

Client Layer

Valid

Valid

Valid

Admin UI Pages
bookings.astro
enquiries.astro

fetch() calls

handleAdminEnquiries
(enquiries.js)

handleAdminEnquiryDetail
(enquiry-detail.js)

handleAdminInviteResend
(invites.js)

handleBookingAttendance
(referenced in bookings.astro)

checkAdminToken

enquiryRepo
listEnquiries
insertEnquiry
getEnquiryById
updateEnquiry
appendEnquiryEvent

inviteRepo
getInviteById
getInviteByEnquiryId
createInviteForEnquiry
markInviteSent

bookingRepo
listBookingsByEnquiry
updateBookingStatus

sendBookingInvite

sendMembershipInvite

enquiries
invites
bookings
membership_otps
```

**Sources:**[tests/unit/admin.test.ts1-583](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/admin.test.ts#L1-L583)[src/pages/admin/bookings.astro1-253](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/bookings.astro#L1-L253)

---

## Common Response Patterns

### Success Response

All successful responses follow the structure:

```
{
  "ok": true,
  "data": {
    // Endpoint-specific data
  }
}
```

### Error Response

All error responses include:

```
{
  "ok": false,
  "code": "ERROR_CODE",
  "error": "Human-readable error message"
}
```

**Standard Error Codes:**
CodeStatusUsage`UNAUTHORIZED`401Missing or invalid admin token`NOT_FOUND`404Requested resource does not exist`VALIDATION_ERROR`422Invalid request body or parameters`METHOD_NOT_ALLOWED`405HTTP method not supported for endpoint`INTERNAL_ERROR`500Unexpected server error
### HTTP Status Codes
StatusMeaningWhen Used200 OKSuccessGET requests, updates, existing resource returned201 CreatedSuccessPOST requests creating new resource401 UnauthorizedAuth failureInvalid/missing admin token404 Not FoundResource missingInvalid enquiry/invite/booking ID405 Method Not AllowedWrong methodUsing GET on POST-only endpoint422 Unprocessable EntityValidation failureInvalid email, missing required fields500 Internal Server ErrorServer errorDatabase failure, unexpected exception
**Sources:**[tests/unit/admin.test.ts159-161](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/admin.test.ts#L159-L161)

---

## Authentication Testing

The test suite in [tests/unit/admin.test.ts167-215](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/admin.test.ts#L167-L215) validates:
Test CaseExpected BehaviorNo auth headerReturns 401Wrong tokenReturns 401`"dev"` token in productionReturns 401 (rejected)`"dev"` token in developmentReturns 200 (accepted)Correct `ADMIN_TOKEN`Returns 200 in any environment`x-admin-token` headerReturns 200 (alternative header accepted)
**Sources:**[tests/unit/admin.test.ts167-215](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/admin.test.ts#L167-L215)

---

## Handler-to-Repository Mapping

The following table maps each admin API handler to its primary repository dependencies:
HandlerRepositories UsedPurpose`handleAdminEnquiries` (GET)`enquiryRepo.listEnquiries`Fetch paginated enquiry list`handleAdminEnquiries` (POST)`enquiryRepo.getEnquiryByEmail``enquiryRepo.insertEnquiry`Check duplicates, create enquiry`handleAdminEnquiryDetail` (GET)`enquiryRepo.getEnquiryById``inviteRepo.getInviteById``bookingRepo.listBookingsByEnquiry`Enrich enquiry with related data`handleAdminEnquiryDetail` (PATCH)`enquiryRepo.getEnquiryById``enquiryRepo.updateEnquiry`Update note field`handleAdminInviteResend``enquiryRepo.getEnquiryById``inviteRepo.getInviteByEnquiryId``inviteRepo.createInviteForEnquiry``inviteRepo.markInviteSent``enquiryRepo.appendEnquiryEvent`Resend/create invite with event trackingBooking attendance handler`bookingRepo.updateBookingStatus``membershipRepo.createOTP` (conditional)Record attendance, send membership link
**Sources:**[tests/unit/admin.test.ts53-62](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/admin.test.ts#L53-L62)