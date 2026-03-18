# Academy Waitlist
Relevant source files
- [src/api/booking.ts](https://github.com/egac-web/egac/blob/0ba542fc/src/api/booking.ts)
- [src/api/enquiry.ts](https://github.com/egac-web/egac/blob/0ba542fc/src/api/enquiry.ts)
- [src/api/health.ts](https://github.com/egac-web/egac/blob/0ba542fc/src/api/health.ts)
- [src/lib/business/age.ts](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/business/age.ts)
- [src/lib/business/enquiry.ts](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/business/enquiry.ts)
- [src/lib/business/templates.ts](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/business/templates.ts)
- [src/lib/business/tokens.ts](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/business/tokens.ts)
- [src/pages/admin/academy.astro](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/academy.astro)
- [src/pages/admin/settings.astro](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/settings.astro)
- [tests/unit/api.test.ts](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/api.test.ts)
- [tests/unit/business.test.ts](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/business.test.ts)

## Purpose and Scope

The academy waitlist manages enquiries from younger athletes who are routed to seasonal academy programs instead of immediate taster session bookings. When an enquiry is submitted, the system resolves the athlete's age group from D1 and examines the `booking_type` field. If the age group has `booking_type='waitlist'`, the enquiry bypasses the taster session invite flow and instead creates an entry in the `academy_waitlist` table.

This document covers the waitlist data model, entry creation process, acceptance/decline response flow, and admin management interfaces. For details on how age groups are resolved and the athletics season calculation rules, see [Age Group System](/egac-web/egac/6.1-age-group-system). For scheduled automation of season transitions and waitlist migration, see [Academy Rollover Job](/egac-web/egac/9.4-academy-rollover-job).

---

## Routing Decision: Taster vs Waitlist

The system determines whether to route an enquiry to taster sessions or the academy waitlist based on the `booking_type` field of the resolved age group. This decision happens in [src/lib/business/enquiry.ts69-72](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/business/enquiry.ts#L69-L72) immediately after age group resolution.

```
'taster'

'waitlist'

null (no match)

Enquiry submitted
with DOB

resolveAgeGroup()
queries D1 age_groups

ageGroup.booking_type

createInviteForEnquiry()
sendBookingInvite()

addToWaitlist()
sendAcademyWaitlist()

Default to taster
(backwards compatibility)

invites table

academy_waitlist table
```

**Sources:**[src/lib/business/enquiry.ts69-72](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/business/enquiry.ts#L69-L72)[src/api/enquiry.ts166-210](https://github.com/egac-web/egac/blob/0ba542fc/src/api/enquiry.ts#L166-L210)

The routing function is pure and stateless:
InputOutput`ageGroup` with `booking_type='taster'``'taster'``ageGroup` with `booking_type='waitlist'``'waitlist'``null` (no age group match)`'taster'` (default)
Age groups with `booking_type='waitlist'` typically have `age_min_aug31` and `age_max_aug31` values that cover younger athletes (e.g., 9-10 years old on August 31). Administrators configure these boundaries through the settings page (see [System Settings](/egac-web/egac/4.5-system-settings)).

---

## Data Model

The academy waitlist system uses two primary tables: `academy_waitlist` for individual athlete entries and `academy_seasons` for defining time-bounded capacity allocations per age group.

### Schema: academy_waitlist Table

```
belongs to

assigned to (nullable)

rolled from (nullable)

defines capacity for

academy_waitlist

string

id

PK

awl_...

string

enquiry_id

FK

references enquiries.id

string

season_id

FK

nullable, references academy_seasons.id

string

token

UK

48-char hex, for response URL

int

position

nullable, queue position (future use)

enum

status

waiting|invited|accepted|declined|ineligible|withdrawn

int

is_returning

0|1, flagged during rollover

string

rolled_over_from_season_id

FK

nullable, tracks migration

datetime

invited_at

when status changed to invited

datetime

responded_at

when user accepted/declined

string

response

yes|no|null

int

send_attempts

email retry counter

string

last_send_error

nullable, error message from Resend

datetime

created_at

string

environment

development|staging|production

academy_seasons

string

id

PK

ase_...

string

age_group_id

FK

references age_groups.id

string

label

e.g. U11 Academy 2026/27

date

start_date

season start

date

end_date

season end

int

capacity

max accepted athletes

enum

status

upcoming|open|closed|archived

datetime

created_at

datetime

updated_at

string

environment

enquiries

string

id

PK

string

email

UK

string

name

string

dob

string

age_group_id

FK

age_groups

string

id

PK

string

code

UK

enum

booking_type

taster|waitlist

int

age_min_aug31

int

age_max_aug31
```

**Sources:**[src/lib/db/schema.ts](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/db/schema.ts) (inferred from repository operations), [src/lib/repositories/academy.js](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/repositories/academy.js)

### Field Definitions
FieldTypePurpose`token`string (unique)Secure 48-char hex token included in response URLs`season_id`string (nullable)Links entry to specific academy season; null for unassigned entries`status`enumCurrent lifecycle state of the waitlist entry`is_returning`int (0|1)Flagged during rollover if athlete was in previous season`rolled_over_from_season_id`string (nullable)Tracks the origin season during migration`response`string (nullable)User's response: `'yes'`, `'no'`, or null if not yet responded`send_attempts`intIncremented on each email send attempt; used by retry cron job
The `position` field is reserved for future queue management but currently nullable and unused.

**Sources:**[src/lib/repositories/academy.js](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/repositories/academy.js#LNaN-LNaN)[tests/unit/api.test.ts331-378](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/api.test.ts#L331-L378)

---

## Waitlist Entry Creation

When an enquiry is routed to the waitlist, the handler creates an entry with status `'waiting'` and immediately sends a confirmation email containing the response token.

### Creation Flow

```
D1 Database
email templates
academy repository
enquiries repository
handleEnquiry
D1 Database
email templates
academy repository
enquiries repository
handleEnquiry
returns 'waitlist'
insertEnquiry(data)
INSERT INTO enquiries
enquiry record
routeEnquiry(ageGroup)
addToWaitlist(enquiry_id)
generateId('awl')
generateSecureToken(24)
INSERT INTO academy_waitlist
waitlist entry
sendAcademyWaitlist(enquiry, invitation)
buildVars: contact_name, response_url
{success: true, id: "email_002"}
markWaitlistInviteSent(invitation_id)
UPDATE academy_waitlist SET send_attempts=1
appendEnquiryEvent('academy_waitlist_added')
updateEnquiry(processed=1)
```

**Sources:**[src/api/enquiry.ts169-186](https://github.com/egac-web/egac/blob/0ba542fc/src/api/enquiry.ts#L169-L186)[src/lib/repositories/academy.js](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/repositories/academy.js#LNaN-LNaN)

The entry is created with:

- `id`: Generated using `generateId('awl')`
- `token`: 48-character hex string via `generateSecureToken(24)`
- `status`: `'waiting'`
- `season_id`: `null` (assigned later by admins or rollover job)
- `send_attempts`: `0` initially, incremented to `1` after email send

If the email send fails, the entry is still persisted with `send_attempts=0` and `last_send_error` populated. The retry cron job (see [Retry Invites Job](/egac-web/egac/9.2-invite-management-jobs)) will attempt to resend.

**Sources:**[src/lib/repositories/academy.js22-53](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/repositories/academy.js#L22-L53)[src/email/templates.ts](https://github.com/egac-web/egac/blob/0ba542fc/src/email/templates.ts#LNaN-LNaN)

---

## Response Flow: Accept or Decline

Athletes (or their parents) receive an email with a unique response URL containing the waitlist entry token. Clicking "Yes, I'm interested" or "No thanks" navigates to a page that submits a POST request to `/api/academy/respond`.

### POST /api/academy/respond

**Endpoint:**`POST /api/academy/respond`

**Request Body:**

```
{
  "token": "abc123...xyz",
  "response": "yes" | "no"
}
```

**Response Codes:**
StatusConditionBody200Response recorded successfully`{ok: true, data: {response: "yes"}}`200Already responded (idempotent)`{ok: true, data: {already_responded: true}}`404Token not found`{ok: false, error: "Waitlist entry not found"}`422Invalid response value`{ok: false, error: "Response must be yes or no"}`
**Handler Logic:**

```
No

Yes

null

found

accepted/declined

waiting/invited

POST /api/academy/respond

Parse body: token, response

response === 'yes'
OR 'no'?

422 VALIDATION_ERROR

getWaitlistEntryByToken(token)

404 ENTRY_NOT_FOUND

entry.status
already responded?

200 {already_responded: true}

recordWaitlistResponse(id, response)

UPDATE academy_waitlist
SET status, response, responded_at

appendEnquiryEvent('academy_response_received')

200 {ok: true, data: {response}}
```

**Sources:**[src/api/academy.ts](https://github.com/egac-web/egac/blob/0ba542fc/src/api/academy.ts#LNaN-LNaN)[tests/unit/api.test.ts540-658](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/api.test.ts#L540-L658)

The handler is **idempotent**: if a user clicks the link twice, subsequent requests return 200 with `already_responded: true` without modifying the database.

Status transitions:

- `'waiting'` → `'accepted'` (if response='yes')
- `'waiting'` → `'declined'` (if response='no')
- `'invited'` → `'accepted'` (if response='yes')
- `'invited'` → `'declined'` (if response='no')

The `response` field is set to `'yes'` or `'no'`, and `responded_at` is set to the current timestamp.

**Sources:**[src/lib/repositories/academy.js](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/repositories/academy.js#LNaN-LNaN)[src/api/academy.ts24-89](https://github.com/egac-web/egac/blob/0ba542fc/src/api/academy.ts#L24-L89)

---

## Admin Management Interface

Administrators access the academy management dashboard at `/admin/academy` to view all waitlist entries, assign them to seasons, and update their status.

### Page Structure: /admin/academy

```
admin/academy.astro

Header KPIs

Unassigned Waitlist Section

Season Allocation Section

Total Seasons count

Open Seasons count

Unassigned entries count

Total Accepted count

Table: enquiry_name, email,
created_at, status

Status update form
POST action='waitlist-status'

Grid of season cards

Season label, dates, capacity
Fill rate progress bar

Status dropdown
POST action='season-status'

First 8 waitlist entries
with status badges
```

**Sources:**[src/pages/admin/academy.astro1-298](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/academy.astro#L1-L298)

### Unassigned Waitlist Table

Entries with `season_id=null` are displayed in a dedicated table. Administrators can manually change their status via a dropdown:
Status OptionMeaning`waiting`Default state after creation`invited`Manually marked as invited by admin`accepted`Athlete has accepted (or admin override)`declined`Athlete has declined (or admin override)`ineligible`Does not meet criteria (admin decision)`withdrawn`Athlete has withdrawn from waitlist
Changing the status submits a POST request with `action=waitlist-status`, which calls [src/lib/repositories/academy.js](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/repositories/academy.js#LNaN-LNaN)

**Sources:**[src/pages/admin/academy.astro171-228](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/academy.astro#L171-L228)[src/lib/repositories/academy.js](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/repositories/academy.js#LNaN-LNaN)

### Season Allocation

Each academy season is displayed as a card showing:

- **Label**: e.g., "U11 Academy 2026/27"
- **Dates**: Start date to end date
- **Capacity**: `X accepted / Y capacity`
- **Fill Rate**: Progress bar calculated as `(acceptedCount / capacity) * 100`
- **Status Dropdown**: Update season status (upcoming, open, closed, archived)
- **Entry Preview**: First 8 waitlist entries assigned to this season

Admins can update a season's status via POST with `action=season-status`, which calls [src/lib/repositories/academy.js](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/repositories/academy.js#LNaN-LNaN)

**Sources:**[src/pages/admin/academy.astro230-296](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/academy.astro#L230-L296)[src/lib/repositories/academy.js](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/repositories/academy.js#LNaN-LNaN)

### Enrichment Pattern

The page enriches waitlist entries with enquiry details (name, email) by fetching from the `enquiries` table and caching in a `Map` to avoid duplicate lookups:

```
const enquiryCache = new Map<string, { name: string | null; email: string }>();
 
async function getEnquiryDetails(enquiryId: string) {
  const cached = enquiryCache.get(enquiryId);
  if (cached) return cached;
  
  const enquiry = await getEnquiryById(enquiryId, env);
  const details = { name: enquiry?.name ?? null, email: enquiry?.email ?? '' };
  enquiryCache.set(enquiryId, details);
  return details;
}
```

This prevents N+1 query problems when rendering large waitlists.

**Sources:**[src/pages/admin/academy.astro68-87](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/academy.astro#L68-L87)

---

## Season Lifecycle and Status Management

Academy seasons have a four-stage lifecycle controlled by the `status` field:

```
Season created

Admin opens registration

Season ends or capacity reached

After season completes

Soft delete

upcoming

open

closed

archived

Future season, not yet accepting

Currently accepting waitlist responses

No longer accepting, in progress

Historical record only
```

**Sources:**[src/lib/db/schema.ts](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/db/schema.ts) (inferred from status enum), [src/pages/admin/academy.astro26-34](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/academy.astro#L26-L34)

### Status Transitions
FromToTrigger`upcoming``open`Admin manual update or rollover job`open``closed`Admin manual update or season end date passes`closed``archived`Admin manual update after season completes
The rollover job (see [Academy Rollover Job](/egac-web/egac/9.4-academy-rollover-job)) automatically transitions seasons with `end_date < today` from `open` to `closed`, then creates entries in the next season for accepted athletes.

**Sources:**[src/api/cron/academy-rollover.ts](https://github.com/egac-web/egac/blob/0ba542fc/src/api/cron/academy-rollover.ts) (referenced for context)

---

## Integration Points

### Email Template: academy_waitlist

When a waitlist entry is created, the system sends an email using the `academy_waitlist` template key. The template receives these variables:
VariableSourceExample`contact_name``enquiry.name`"Jane Smith"`parent_name``enquiry.name`"Jane Smith"`child_name``enquiry.name`"Jane Smith"`response_url``{siteUrl}/academy/respond/{token}``https://egac.pages.dev/academy/respond/abc123...`
The response URL includes the waitlist entry's unique token, which is validated server-side when the user submits their choice.

**Sources:**[src/lib/email/templates.ts](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/email/templates.ts#LNaN-LNaN)[src/lib/business/templates.ts](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/business/templates.ts#LNaN-LNaN)

### Enquiry Event Trail

All waitlist-related actions are recorded in the `enquiries.events` JSON array:
Event TypeTriggerPayload`academy_waitlist_added`Entry created`{}``academy_response_received`User responds`{response: "yes" | "no"}`
This event sourcing enables full audit trails and troubleshooting.

**Sources:**[src/api/enquiry.ts175](https://github.com/egac-web/egac/blob/0ba542fc/src/api/enquiry.ts#L175-L175)[src/api/academy.ts69-77](https://github.com/egac-web/egac/blob/0ba542fc/src/api/academy.ts#L69-L77)

---

## Testing

The academy waitlist response handler has comprehensive test coverage:

**Test Cases (from [tests/unit/api.test.ts540-658](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/api.test.ts#L540-L658)]):**
TestValidatesValid yes responseReturns 200 with `{response: "yes"}`Valid no responseReturns 200 with `{response: "no"}`Token not foundReturns 404Invalid response valueReturns 422 for response != 'yes'|'no'Idempotent re-submissionReturns 200 with `already_responded: true`Event appendingCalls `appendEnquiryEvent` with correct payload
The tests mock all repository calls and verify the handler's routing logic without hitting real D1 or Resend APIs.

**Sources:**[tests/unit/api.test.ts540-658](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/api.test.ts#L540-L658)

---

## Summary

The academy waitlist system provides a structured pathway for younger athletes to register interest in seasonal academy programs. Key characteristics:

1. **Automatic Routing**: Age group `booking_type` field determines taster vs waitlist path
2. **Token-Based Responses**: Secure 48-char hex tokens enable simple yes/no response flow
3. **Flexible Status Management**: Six status values support various admin workflows
4. **Season Allocation**: Waitlist entries can be unassigned or linked to specific seasons
5. **Admin Control**: Full CRUD operations via `/admin/academy` interface
6. **Event Sourcing**: All actions logged to `enquiries.events` for auditability

For age group configuration and booking type settings, see [System Settings](/egac-web/egac/4.5-system-settings). For automated season transitions and waitlist migration, see [Academy Rollover Job](/egac-web/egac/9.4-academy-rollover-job).