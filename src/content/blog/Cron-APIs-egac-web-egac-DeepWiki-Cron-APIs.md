# Cron APIs
Relevant source files
- [db/migrations/0001_initial_schema.sql](https://github.com/egac-web/egac/blob/0ba542fc/db/migrations/0001_initial_schema.sql)
- [db/seed.sql](https://github.com/egac-web/egac/blob/0ba542fc/db/seed.sql)
- [tests/unit/admin.test.ts](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/admin.test.ts)
- [tests/unit/cron.test.ts](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/cron.test.ts)
- [tests/unit/db.test.ts](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/db.test.ts)
- [vitest.config.ts](https://github.com/egac-web/egac/blob/0ba542fc/vitest.config.ts)
- [workers/cron/src/index.ts](https://github.com/egac-web/egac/blob/0ba542fc/workers/cron/src/index.ts)
- [workers/cron/wrangler.toml](https://github.com/egac-web/egac/blob/0ba542fc/workers/cron/wrangler.toml)
- [wrangler.toml](https://github.com/egac-web/egac/blob/0ba542fc/wrangler.toml)

## Purpose and Scope

This document describes the scheduled job API endpoints in the EGAC platform. These endpoints are called by a dedicated Cloudflare Worker (`egac-cron`) on configured schedules to execute background tasks such as expiring stale invites, retrying failed email sends, sending booking reminders, and rolling over academy seasons.

For information about the business logic executed by these jobs (e.g., invite lifecycle, email sending), see [Email System](/egac-web/egac/8-email-system). For the repository operations used by these handlers, see [Data Access Layer](/egac-web/egac/7-data-access-layer). For general API authentication patterns used by admin endpoints, see [Admin APIs](/egac-web/egac/5.2-admin-apis).

---

## Architecture Overview

The EGAC platform uses a **two-tier cron architecture** to work around Cloudflare Pages' lack of native cron trigger support:

1. A standalone Cloudflare Worker (`egac-cron`) receives cron triggers from Cloudflare's scheduler
2. The worker maps each cron expression to a corresponding API endpoint in the Pages project
3. The worker POSTs to the endpoint with `CRON_SECRET` authentication
4. The Pages API handler executes the business logic and returns a JSON response

This design isolates all business logic in the Pages project while delegating only trigger management to the worker.

**Diagram: Cron Architecture**

```
Email System
D1 Database
Repositories
Pages API
/api/cron/*
egac-cron Worker
Cloudflare Scheduler
Email System
D1 Database
Repositories
Pages API
/api/cron/*
egac-cron Worker
Cloudflare Scheduler
Trigger cron
(e.g., "0 9 * * *")
Lookup CRON_ROUTES
mapping
POST /api/cron/expire-invites
Authorization: Bearer CRON_SECRET
Validate CRON_SECRET
Query data
(e.g., listSentInvitesOlderThan)
SELECT ...
Rows
Update data
(e.g., markInviteExpired)
UPDATE ...
Send emails (optional)
200 OK
{ ok: true, data: {...} }
Log result
```

**Sources:**[workers/cron/src/index.ts1-55](https://github.com/egac-web/egac/blob/0ba542fc/workers/cron/src/index.ts#L1-L55)[tests/unit/cron.test.ts1-607](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/cron.test.ts#L1-L607)

---

## Authentication

All cron API endpoints require the `CRON_SECRET` environment variable to be provided in the `Authorization` header using the Bearer scheme:

```
Authorization: Bearer <CRON_SECRET>

```

If the header is missing or the secret is incorrect, the endpoint returns `401 Unauthorized`:

```
{
  "ok": false,
  "code": "UNAUTHORIZED",
  "message": "Invalid or missing CRON_SECRET"
}
```

The `CRON_SECRET` is configured as a secret in both the worker (`egac-cron`) and the Pages project.

**Sources:**[tests/unit/cron.test.ts189-203](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/cron.test.ts#L189-L203)[tests/unit/cron.test.ts270-274](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/cron.test.ts#L270-L274)

---

## Cron Endpoints

### Summary Table
EndpointScheduleCron ExpressionPurpose`/api/cron/expire-invites`Daily 09:00 UTC`0 9 * * *`Mark invites older than 14 days as expired`/api/cron/retry-invites`Every 30 minutes`*/30 * * * *`Retry pending invite email sends (max 3 attempts)`/api/cron/send-reminders`Tue 18:00 UTC`0 18 * * 2`Send booking reminders for sessions 7 days ahead`/api/cron/academy-rollover`1st of month 09:00 UTC`0 9 1 * *`Close expired academy seasons and migrate waitlist
**Sources:**[workers/cron/src/index.ts18-23](https://github.com/egac-web/egac/blob/0ba542fc/workers/cron/src/index.ts#L18-L23)[wrangler.toml42-49](https://github.com/egac-web/egac/blob/0ba542fc/wrangler.toml#L42-L49)

---

### POST /api/cron/expire-invites

**Schedule:** Daily at 09:00 UTC (`0 9 * * *`)

**Purpose:** Marks taster session booking invites as `expired` if they were sent more than 14 days ago and have not been accepted.

**Logic:**

1. Query all invites with `status = 'sent'` created more than 14 days ago using `listSentInvitesOlderThan(14, env)`
2. For each stale invite:

- Call `markInviteExpired(invite.id, env)` to set `status = 'expired'`
- Append an `invite_expired` event to the enquiry's event log
3. Return the count of expired invites

**Request:**

```
POST /api/cron/expire-invites
Authorization: Bearer <CRON_SECRET>
```

**Response (200 OK):**

```
{
  "ok": true,
  "data": {
    "expired": 3
  }
}
```

**Idempotency:** This job is idempotent. Running it multiple times in a day has no additional effect because invites already marked `expired` are excluded from the query.

**Diagram: Expire Invites Flow**

```
Yes

No

Done

handleExpireInvites

Validate CRON_SECRET

listSentInvitesOlderThan(14)

Invites
found?

For each invite

markInviteExpired(invite.id)

appendEnquiryEvent
('invite_expired')

Return { expired: count }
```

**Sources:**[tests/unit/cron.test.ts181-251](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/cron.test.ts#L181-L251)[workers/cron/src/index.ts19](https://github.com/egac-web/egac/blob/0ba542fc/workers/cron/src/index.ts#L19-L19)

---

### POST /api/cron/retry-invites

**Schedule:** Every 30 minutes (`*/30 * * * *`)

**Purpose:** Retry sending booking invite emails for invites with `status = 'pending'`. Implements exponential backoff with a maximum of 3 attempts. After 3 failed attempts, marks the invite as `failed`.

**Logic:**

1. Query all invites with `status = 'pending'` using `listPendingInvites(env)`
2. For each pending invite:

- Fetch the associated enquiry
- Generate available session dates using `getNextNTuesdayDates`
- Attempt to send the invite email via `sendBookingInvite`
- **On success:**
- Call `markInviteSent(invite.id, env)` to set `status = 'sent'`
- Append `invite_resent` event to enquiry
- Increment `retried` counter
- **On failure:**
- Call `incrementSendAttempts(invite.id, error, env)`
- If `send_attempts + 1 >= 3` (MAX_ATTEMPTS):

- Call `markInviteFailed(invite.id, env)` to set `status = 'failed'`
- Increment `failed` counter
3. Return counts of retried and failed invites

**Constants:**

- `MAX_ATTEMPTS = 3`

**Request:**

```
POST /api/cron/retry-invites
Authorization: Bearer <CRON_SECRET>
```

**Response (200 OK):**

```
{
  "ok": true,
  "data": {
    "retried": 2,
    "failed": 1
  }
}
```

**Diagram: Retry Invites Logic**

```
No

Yes

Yes

No

Yes

No

Done

handleRetryInvites

Validate CRON_SECRET

listPendingInvites()

For each invite

getEnquiryById(invite.enquiry_id)

Enquiry
exists?

getNextNTuesdayDates()

sendBookingInvite()

Email
sent?

markInviteSent()
appendEnquiryEvent('invite_resent')

incrementSendAttempts()

send_attempts + 1
>= 3?

markInviteFailed()
status = 'failed'

Return { retried, failed }
```

**Sources:**[tests/unit/cron.test.ts257-353](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/cron.test.ts#L257-L353)[workers/cron/src/index.ts21](https://github.com/egac-web/egac/blob/0ba542fc/workers/cron/src/index.ts#L21-L21)

---

### POST /api/cron/send-reminders

**Schedule:** Every Tuesday at 18:00 UTC (`0 18 * * 2`)

**Purpose:** Send reminder emails to athletes with confirmed bookings scheduled for 7 days from now.

**Logic:**

1. Calculate the target date: `today + 7 days`
2. Query all bookings with `session_date = targetDate` and `status = 'confirmed'` using `listBookingsForDate(targetDate, env)`
3. For each booking:

- Fetch the associated enquiry
- Check the enquiry's `events` JSON for an existing `reminder_sent` event with matching `booking_id`
- **If reminder already sent:** Skip (idempotency)
- **If no reminder sent:**
- Send reminder email via `sendBookingReminder(enquiry, booking, venue, env)`
- Append `reminder_sent` event to enquiry with `booking_id` field
- Increment `sent` counter
4. Return counts of sent and skipped reminders

**Idempotency:** The event log ensures reminders are sent exactly once per booking, even if the job runs multiple times.

**Request:**

```
POST /api/cron/send-reminders
Authorization: Bearer <CRON_SECRET>
```

**Response (200 OK):**

```
{
  "ok": true,
  "data": {
    "sent": 5,
    "skipped": 2
  }
}
```

**Diagram: Send Reminders Flow**

```
No

Yes

Yes

No

Done

handleSendReminders

Validate CRON_SECRET

targetDate = today + 7 days

listBookingsForDate(targetDate)

For each booking

status ==
'confirmed'?

getEnquiryById(booking.enquiry_id)

Parse enquiry.events JSON

reminder_sent
event exists?

sendBookingReminder()

appendEnquiryEvent
('reminder_sent', { booking_id })

Return { sent, skipped }
```

**Sources:**[tests/unit/cron.test.ts359-447](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/cron.test.ts#L359-L447)[workers/cron/src/index.ts20](https://github.com/egac-web/egac/blob/0ba542fc/workers/cron/src/index.ts#L20-L20)

---

### POST /api/cron/academy-rollover

**Schedule:** 1st of each month at 09:00 UTC (`0 9 1 * *`)

**Purpose:** Automatically close expired academy seasons and migrate waiting athletes to the next upcoming season.

**Logic:**

1. Query all academy seasons with `status = 'open'` and `end_date < today` using `listAcademySeasons({ status: 'open' }, env)`
2. For each expired season:

- Fetch the next upcoming season for the same age group using `getUpcomingAcademySeason(season.age_group_id, env)`
- Query all waitlist entries for the season with `status IN ('waiting', 'invited')`
- For each waitlist entry:

- Check if the enquiry has an `accepted` entry in any previous season using `hasAcceptedEntryForEnquiry`
- Call `rolloverWaitlistEntry(entry.id, nextSeasonId, season.id, isReturning, env)`:

- Creates a new entry for the next season (or leaves `season_id = NULL` if no upcoming season exists)
- Sets `is_returning = 1` if the athlete previously accepted
- Sets `rolled_over_from_season_id` to the old season ID
- Call `updateSeasonStatus(season.id, 'closed', env)`
- Increment counters
3. Return counts of rolled-over entries and closed seasons

**Idempotency:** Running this job multiple times in a month is safe because:

- Only `open` seasons with `end_date < today` are processed
- Once a season is marked `closed`, it is excluded from future runs
- Waitlist entries with `status = 'accepted'` or `'declined'` are not rolled over

**Request:**

```
POST /api/cron/academy-rollover
Authorization: Bearer <CRON_SECRET>
```

**Response (200 OK):**

```
{
  "ok": true,
  "data": {
    "rolledOver": 12,
    "seasonsClosed": 1
  }
}
```

**Diagram: Academy Rollover Flow**

```
No

Yes

No

Yes

Done

Done

handleAcademyRollover

Validate CRON_SECRET

listAcademySeasons
({ status: 'open' })

Filter: end_date < today

Expired
seasons?

For each expired season

getUpcomingAcademySeason
(age_group_id)

listWaitlistBySeason(season.id)

For each waitlist entry

status IN
('waiting', 'invited')?

hasAcceptedEntryForEnquiry
(enquiry_id)

rolloverWaitlistEntry
(entryId, nextSeasonId, oldSeasonId, isReturning)

updateSeasonStatus
(seasonId, 'closed')

Return { rolledOver, seasonsClosed }
```

**Sources:**[tests/unit/cron.test.ts500-606](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/cron.test.ts#L500-L606)[workers/cron/src/index.ts22](https://github.com/egac-web/egac/blob/0ba542fc/workers/cron/src/index.ts#L22-L22)

---

## Worker Configuration

The `egac-cron` worker is a standalone Cloudflare Worker that triggers the cron API endpoints. Its configuration is defined in `workers/cron/wrangler.toml`.

**Key Configuration:**
FieldValueDescription`name``egac-cron`Worker name`main``src/index.ts`Entry point`PAGES_URL``https://egac-cl9.pages.dev`Target Pages project URL`CRON_SECRET`(secret)Shared secret for authentication
**Cron Triggers:**

The worker defines four cron triggers in `[triggers].crons`:

```
[triggers]
crons = [
  "0 9 * * *",      # Daily 09:00 UTC
  "0 18 * * 2",     # Tuesday 18:00 UTC
  "*/30 * * * *",   # Every 30 minutes
  "0 9 1 * *"       # 1st of month 09:00 UTC
]
```

**Worker Logic:**

The worker maintains a `CRON_ROUTES` mapping from cron expressions to API endpoints:

```
const CRON_ROUTES: Record<string, string> = {
  '0 9 * * *':    '/api/cron/expire-invites',
  '0 18 * * 2':   '/api/cron/send-reminders',
  '*/30 * * * *': '/api/cron/retry-invites',
  '0 9 1 * *':    '/api/cron/academy-rollover',
};
```

On each trigger, the worker:

1. Looks up the route for the cron expression
2. POSTs to `${PAGES_URL}${route}` with `Authorization: Bearer ${CRON_SECRET}`
3. Logs the response status and body

**Sources:**[workers/cron/wrangler.toml1-28](https://github.com/egac-web/egac/blob/0ba542fc/workers/cron/wrangler.toml#L1-L28)[workers/cron/src/index.ts1-55](https://github.com/egac-web/egac/blob/0ba542fc/workers/cron/src/index.ts#L1-L55)

---

## Request/Response Patterns

All cron endpoints follow a consistent request/response structure.

### Common Request Headers

```
POST /api/cron/{endpoint}
Authorization: Bearer <CRON_SECRET>
Content-Type: application/json
```

### Success Response (200 OK)

```
{
  "ok": true,
  "data": {
    // Endpoint-specific data
    // Always includes metrics/counts
  }
}
```

### Error Responses
StatusCodeDescription`401``UNAUTHORIZED`Missing or invalid `CRON_SECRET``405``METHOD_NOT_ALLOWED`Non-POST request`500``INTERNAL_ERROR`Unexpected server error
**Example Error Response:**

```
{
  "ok": false,
  "code": "UNAUTHORIZED",
  "message": "Invalid or missing CRON_SECRET"
}
```

**Sources:**[tests/unit/cron.test.ts189-213](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/cron.test.ts#L189-L213)

---

## Error Handling

### Transient Failures

Cron jobs are designed to be **resilient to transient failures**:

- **Email failures** do not block job completion. If `sendBookingInvite` or `sendBookingReminder` fails, the error is logged and the job continues processing other items.
- **Database failures** bubble up as 500 errors and are logged by the worker. The job will retry on the next scheduled run.

### Retry Logic

The `retry-invites` job implements **exponential backoff** internally:

- Each failed send increments `send_attempts`
- After 3 attempts, the invite is permanently marked `failed`
- The 30-minute schedule provides natural spacing between retries

Other jobs rely on **schedule-based retries**:

- If `expire-invites` fails, it retries the next day
- If `send-reminders` fails, it retries the next Tuesday

### Logging

The worker logs all trigger events and API responses:

```
{
  "level": "info",
  "event": "cron_triggered",
  "cron": "0 9 * * *",
  "route": "/api/cron/expire-invites",
  "status": 200,
  "body": "{\"ok\":true,\"data\":{\"expired\":3}}"
}
```

**Sources:**[workers/cron/src/index.ts41-51](https://github.com/egac-web/egac/blob/0ba542fc/workers/cron/src/index.ts#L41-L51)

---

## Deployment

### Worker Deployment

Deploy the cron worker from `workers/cron`:

```
cd workers/cron
npx wrangler deploy
```

Set the `CRON_SECRET` (one-time):

```
npx wrangler secret put CRON_SECRET
```

### Pages Configuration

The Pages project must have the following secret configured:

```
wrangler secret put CRON_SECRET --env production
```

The secret must match the value configured in the worker.

**Sources:**[workers/cron/wrangler.toml10-13](https://github.com/egac-web/egac/blob/0ba542fc/workers/cron/wrangler.toml#L10-L13)[wrangler.toml16-21](https://github.com/egac-web/egac/blob/0ba542fc/wrangler.toml#L16-L21)

---

## Testing

All cron handlers have comprehensive unit tests in `tests/unit/cron.test.ts`:

### Test Coverage
EndpointTest Cases`expire-invites`Auth enforcement, 14-day threshold, idempotency, event logging`retry-invites`Max attempts enforcement, success/failure paths, enquiry lookup`send-reminders`Idempotency via events, status filtering, date calculation`academy-rollover`Season closure, waitlist migration, returning athlete logic
### Running Tests

```
npm test tests/unit/cron.test.ts
```

**Key Mocking Strategy:**

- All repository functions are mocked using Vitest's `vi.mock()`
- Email functions are mocked to return success/failure
- The `CRON_SECRET` is injected via `makeEnv()` helper
- Each test verifies repository calls, event appends, and response structure

**Sources:**[tests/unit/cron.test.ts1-607](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/cron.test.ts#L1-L607)[vitest.config.ts1-25](https://github.com/egac-web/egac/blob/0ba542fc/vitest.config.ts#L1-L25)