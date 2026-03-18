# Taster Session Booking
Relevant source files
- [docs/snagging.md](https://github.com/egac-web/egac/blob/0ba542fc/docs/snagging.md)
- [src/api/booking.ts](https://github.com/egac-web/egac/blob/0ba542fc/src/api/booking.ts)
- [src/api/enquiry.ts](https://github.com/egac-web/egac/blob/0ba542fc/src/api/enquiry.ts)
- [src/api/health.ts](https://github.com/egac-web/egac/blob/0ba542fc/src/api/health.ts)
- [src/lib/business/age.ts](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/business/age.ts)
- [src/lib/business/enquiry.ts](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/business/enquiry.ts)
- [src/lib/business/templates.ts](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/business/templates.ts)
- [src/lib/business/tokens.ts](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/business/tokens.ts)
- [src/lib/repositories/age-groups.ts](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/repositories/age-groups.ts)
- [src/lib/repositories/bookings.ts](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/repositories/bookings.ts)
- [tests/unit/api.test.ts](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/api.test.ts)
- [tests/unit/business.test.ts](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/business.test.ts)

This document describes how public users book taster training sessions after receiving a booking invite. Taster sessions are available to athletes aged 10+ who have submitted an enquiry through the system. The booking process validates invite tokens, checks session capacity, resolves age group eligibility, and creates confirmed bookings in the database.

**Related pages:**

- For the initial enquiry submission that generates booking invites, see [Enquiry Form](/egac-web/egac/3.1-enquiry-form)
- For how younger athletes join the academy waitlist instead, see [Academy Waitlist](/egac-web/egac/3.3-academy-waitlist)
- For the API specification of the booking endpoint, see [Public APIs](/egac-web/egac/5.1-public-apis)
- For age group eligibility rules and calculations, see [Age Group System](/egac-web/egac/6.1-age-group-system)

---

## Overview

The taster session booking flow is a two-stage process:

1. **Invite generation** (covered in [3.1](/egac-web/egac/3.1-enquiry-form)): When an enquiry is submitted for an athlete aged 10+, the system creates an `invite` record with a secure token and sends an email containing a booking URL with that token
2. **Booking selection** (this page): The user clicks the booking URL, selects a session date from available options, and submits their choice

The booking system enforces strict validation rules:

- Invites expire after 14 days
- Each invite can only be used once
- Session dates must fall on valid training days (typically Tuesday)
- Capacity limits are enforced per age group and session date
- The athlete's age must be eligible for the selected date

**Diagram: Taster Session Booking Flow**

```
"D1 Database"
"Email System"
"age.ts"
"bookings.ts"
"invites.ts"
"POST /api/booking"
"Booking Page
(book/[token].astro)"
"Public User"
"D1 Database"
"Email System"
"age.ts"
"bookings.ts"
"invites.ts"
"POST /api/booking"
"Booking Page
(book/[token].astro)"
"Public User"
Click invite link with token
Fetch available dates
getInviteByToken(token)
SELECT invite
invite record
invite
getNextNSessionDates(ageGroup, weeksAhead)
["2026-06-10", "2026-06-17", ...]
Display date picker
Select date & submit
POST {token, date}
getInviteByToken(token)
Check invite.status != 'accepted'
isInviteExpired(invite.created_at)
resolveAgeGroup(dob, date, ageGroups)
ageGroup
validateSessionDate(date, ageGroup)
true
getBookingByInviteAndDate(inviteId, date)
null (no duplicate)
countBookingsForDateAndGroup(date, ageGroupId)
SELECT COUNT(*)
currentCount
currentCount < capacity
createBooking(enquiryId, inviteId, ageGroupId, date, time)
INSERT INTO bookings
booking
booking
markInviteAccepted(inviteId)
UPDATE invites SET status='accepted'
sendBookingConfirmation(enquiry, booking, venue, calendarUrl)
Confirmation email
200 {booking_id, session_date, message}
Confirmation page
```

**Sources:**[src/api/booking.ts1-146](https://github.com/egac-web/egac/blob/0ba542fc/src/api/booking.ts#L1-L146)[src/lib/repositories/invites.ts](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/repositories/invites.ts)[src/lib/repositories/bookings.ts1-165](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/repositories/bookings.ts#L1-L165)[src/lib/business/age.ts188-210](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/business/age.ts#L188-L210)[tests/unit/api.test.ts407-534](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/api.test.ts#L407-L534)

---

## The Booking Endpoint

The booking endpoint is `POST /api/booking` and is implemented by the `handleBooking` function.

### Request Format

```
{
  "token": "abc123def456...",
  "date": "2026-06-10",
  "age_group_id": "ag_u13"  // optional, usually omitted
}
```
FieldTypeRequiredDescription`token`stringYesThe invite token from the booking URL`date`stringYesSession date in YYYY-MM-DD format`age_group_id`stringNoOverride age group (admin use only)
### Response Format (Success)

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

The endpoint always returns HTTP 200 on success, even if the confirmation email send fails. The booking is never lost due to email delivery problems.

**Sources:**[src/api/booking.ts25-146](https://github.com/egac-web/egac/blob/0ba542fc/src/api/booking.ts#L25-L146)[tests/unit/api.test.ts431-441](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/api.test.ts#L431-L441)

---

## Invite Token Validation

The booking process begins by validating the invite token. This is a multi-step check that prevents misuse of expired or duplicate invites.

**Diagram: Invite Validation Logic**

```
No

Yes

'accepted'

'expired'

'sent' or 'pending'

Yes

No

POST /api/booking
{token, date}

getInviteByToken(token)

Token exists?

404 INVITE_NOT_FOUND

Check invite.status

invite.status

409 INVITE_USED

410 INVITE_EXPIRED

isInviteExpired(created_at)

Age > 14 days?

410 INVITE_EXPIRED

Invite valid
Continue to booking

Age group resolution...
```

### Validation Rules

1. **Token must exist** in the `invites` table with matching environment
2. **Status must not be 'accepted'** - each invite is single-use
3. **Status must not be 'expired'** - expired invites are marked by the cron job
4. **Age must be < 14 days** since `created_at` timestamp (even if status is still 'sent')

The invite TTL constant is defined in [src/lib/business/tokens.ts45](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/business/tokens.ts#L45-L45):

```
export const INVITE_TTL_DAYS = 14;
```

**Sources:**[src/api/booking.ts36-44](https://github.com/egac-web/egac/blob/0ba542fc/src/api/booking.ts#L36-L44)[src/lib/business/tokens.ts45-49](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/business/tokens.ts#L45-L49)[tests/unit/api.test.ts443-501](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/api.test.ts#L443-L501)

---

## Age Group Resolution

Once the invite is validated, the system resolves which age group the athlete belongs to for the selected session date. This is crucial because:

- Age groups determine session eligibility (day of week, time)
- Capacity limits are enforced per age group
- The athlete's age on **31 August** of the athletics season determines the group

**Diagram: Age Group Resolution Process**

```
No

Yes

Yes

No

No

Yes

Validated invite + date

listAgeGroups(env)

Filter booking_type='taster'

resolveAgeGroup(dob, date, tasterGroups)

athleticsSeasonEndYear(date)
→ seasonEndYear

ageForAthletics(dob, date)
→ age on 31 Aug seasonEndYear

Find group where
age >= age_min_aug31
AND age <= age_max_aug31

Group found?

422 NO_AGE_GROUP

age_group_id param provided?

getAgeGroupById(age_group_id)
Use override if taster type

Use resolved group

validateSessionDate(date, ageGroup)

Valid session day
and future date?

422 SLOT_INVALID

Proceed to capacity check
```

### Age Group Resolution Example

For an athlete born `1 September 2014` booking a session on `14 October 2026`:

1. **Session date**: 14 October 2026
2. **Athletics season**: 2026-09-01 to 2027-08-31 (season ends 2027)
3. **Reference date**: 31 August 2027
4. **Age on reference date**: 12 years old
5. **Age group match**: U13 (age_min_aug31=11, age_max_aug31=12)

The `resolveAgeGroup` function in [src/lib/business/age.ts119-135](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/business/age.ts#L119-L135) implements this logic:

```
export function resolveAgeGroup(
  dobIso: string | null | undefined,
  sessionDateIso: string,
  ageGroups: AgeGroupConfig[]
): AgeGroupConfig | null {
  const athleticsAge = ageForAthletics(dobIso, sessionDateIso);
  if (athleticsAge === null) return null;
 
  const active = ageGroups
    .filter((g) => Number(g.active) === 1)
    .sort((a, b) => a.sort_order - b.sort_order);
 
  return (
    active.find((g) => athleticsAge >= g.age_min_aug31 && athleticsAge <= g.age_max_aug31) ?? null
  );
}
```

**Sources:**[src/api/booking.ts50-70](https://github.com/egac-web/egac/blob/0ba542fc/src/api/booking.ts#L50-L70)[src/lib/business/age.ts76-135](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/business/age.ts#L76-L135)[src/lib/repositories/age-groups.ts1-46](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/repositories/age-groups.ts#L1-L46)[tests/unit/business.test.ts194-252](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/business.test.ts#L194-L252)

---

## Session Date Validation

After age group resolution, the selected date is validated against the age group's session rules. Taster age groups typically train on Tuesdays only.

### Validation Rules

The `validateSessionDate` function enforces three rules:

1. **Date must parse as valid ISO date** (YYYY-MM-DD)
2. **Date must fall on a valid session day** for the age group
3. **Date must be strictly in the future** (not today, not past)

The age group's `session_days` field stores valid training days as JSON:

```
["Tuesday"]
```

or for academy groups:

```
["Saturday", "Tuesday"]
```

**Implementation:**

[src/lib/business/age.ts240-253](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/business/age.ts#L240-L253)

```
export function validateSessionDate(
  dateIso: string,
  ageGroup: AgeGroupConfig,
  today: Date = new Date()
): boolean {
  const date = new Date(dateIso);
  if (isNaN(date.getTime())) return false;
  if (!isValidSessionDay(dateIso, ageGroup)) return false;
 
  const todayMidnight = new Date(today);
  todayMidnight.setHours(0, 0, 0, 0);
  date.setHours(0, 0, 0, 0);
  return date > todayMidnight;
}
```

The `isValidSessionDay` helper in [src/lib/business/age.ts171-180](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/business/age.ts#L171-L180) parses the JSON array and checks if the date's day of week matches.

**Sources:**[src/api/booking.ts72-74](https://github.com/egac-web/egac/blob/0ba542fc/src/api/booking.ts#L72-L74)[src/lib/business/age.ts158-180](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/business/age.ts#L158-L180)[src/lib/business/age.ts240-253](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/business/age.ts#L240-L253)[tests/unit/business.test.ts278-309](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/business.test.ts#L278-L309)

---

## Duplicate Booking Check

Before checking capacity, the system prevents duplicate bookings for the same invite and date combination.

```
const existing = await getBookingByInviteAndDate(invite.id, date, env);
if (existing) return err('You already have a booking for this date', 409, 'DUPLICATE_BOOKING');
```

This check is enforced in [src/api/booking.ts77-78](https://github.com/egac-web/egac/blob/0ba542fc/src/api/booking.ts#L77-L78) using the repository function `getBookingByInviteAndDate` from [src/lib/repositories/bookings.ts46-60](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/repositories/bookings.ts#L46-L60)

The query:

```
SELECT * FROM bookings 
WHERE invite_id = ? AND session_date = ? AND environment = ?
```

This prevents:

- Accidental double-clicks during form submission
- User refreshing the booking page after successful submission
- Multiple attempts with the same invite token for the same date

**Note:** The same invite **can** book different dates. The duplicate check is invite+date combined, not just invite. This allows users to potentially book multiple sessions if the invite status hasn't been marked as 'accepted' yet (though the system immediately marks it accepted after the first booking).

**Sources:**[src/api/booking.ts77-78](https://github.com/egac-web/egac/blob/0ba542fc/src/api/booking.ts#L77-L78)[src/lib/repositories/bookings.ts46-60](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/repositories/bookings.ts#L46-L60)[tests/unit/api.test.ts470-478](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/api.test.ts#L470-L478)

---

## Capacity Management

Each taster session has a maximum capacity defined per age group. The system prevents overbooking by counting existing confirmed bookings before creating a new one.

**Diagram: Capacity Check Flow**

```
Yes

No

createBooking request

capacity = ageGroup.capacity_per_session

countBookingsForDateAndGroup(date, ageGroupId)

SELECT COUNT(*) FROM bookings
WHERE session_date = ?
AND age_group_id = ?
AND status = 'confirmed'

currentCount >= capacity?

409 SLOT_FULL
'This session is full'

createBooking(...)
INSERT INTO bookings
```

### Capacity Configuration

Capacity is stored in two places:

1. **Per age group**: `age_groups.capacity_per_session` column (e.g., 2 athletes per U13 session)
2. **Global fallback**: `system_config.capacity_per_slot` (used if age group value is NULL)

The capacity check query in [src/lib/repositories/bookings.ts62-78](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/repositories/bookings.ts#L62-L78):

```
SELECT COUNT(*) as count FROM bookings
WHERE session_date = ? 
  AND age_group_id = ? 
  AND status = 'confirmed' 
  AND environment = ?
```

Only `'confirmed'` bookings count toward capacity. Bookings with status `'attended'`, `'no_show'`, or `'cancelled'` do not reduce available slots for future sessions.

### Capacity Check Code

[src/api/booking.ts81-87](https://github.com/egac-web/egac/blob/0ba542fc/src/api/booking.ts#L81-L87)

```
const config = await getConfig(env);
const capacity = ageGroup.capacity_per_session ?? parseInt(config.weeks_ahead_booking ?? '2', 10);
const currentCount = await countBookingsForDateAndGroup(date, ageGroup.id, env);
 
if (currentCount >= capacity) {
  return err('This session is full. Please choose a different date.', 409, 'SLOT_FULL');
}
```

**Sources:**[src/api/booking.ts81-87](https://github.com/egac-web/egac/blob/0ba542fc/src/api/booking.ts#L81-L87)[src/lib/repositories/bookings.ts62-78](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/repositories/bookings.ts#L62-L78)[tests/unit/api.test.ts460-468](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/api.test.ts#L460-L468)

---

## Booking Creation

Once all validations pass, the system creates the booking record, marks the invite as accepted, and appends an event to the enquiry audit trail.

**Diagram: Booking Creation Sequence**

```
"D1 Database"
"enquiries.ts"
"invites.ts"
"bookings.ts"
"handleBooking"
"D1 Database"
"enquiries.ts"
"invites.ts"
"bookings.ts"
"handleBooking"
createBooking(enquiryId, inviteId, ageGroupId, date, time)
generateId('bkg')
INSERT INTO bookings
(id, enquiry_id, invite_id, age_group_id,
session_date, session_time, status='confirmed')
OK
SELECT * FROM bookings WHERE id=?
booking record
booking
markInviteAccepted(inviteId)
UPDATE invites
SET status='accepted', accepted_at=NOW()
OK
appendEnquiryEvent(enquiryId, {type: 'booking_created'})
UPDATE enquiries
SET events = json_insert(events, '$[
OK
```

### Booking Record Structure

The `bookings` table structure ([src/lib/db/schema.ts](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/db/schema.ts)):
ColumnTypeDescription`id`TEXT PKPrefixed with 'bkg_'`enquiry_id`TEXT FKLinks to original enquiry`invite_id`TEXT FKLinks to invite that was used (nullable for direct bookings)`age_group_id`TEXT FKAge group for this booking`session_date`DATEYYYY-MM-DD format`session_time`TEXTHH:MM format, e.g., '18:30'`status`TEXT'confirmed', 'attended', 'no_show', 'cancelled'`attendance_note`TEXTAdmin notes (nullable)`environment`TEXT'development', 'staging', or 'production'`created_at`TEXTISO 8601 timestamp`updated_at`TEXTISO 8601 timestamp (nullable)
The initial status is always `'confirmed'`. Admins later update to `'attended'` or `'no_show'` via the bookings management interface (see [4.3](/egac-web/egac/4.3-bookings-management)).

### Event Sourcing

The enquiry's `events` field is a JSON array that stores the complete audit trail:

```
[
  {"type": "enquiry_submitted", "timestamp": "2026-06-01T10:00:00Z"},
  {"type": "booking_invite_sent", "timestamp": "2026-06-01T10:00:05Z"},
  {"type": "booking_created", "booking_id": "bkg_abc123", "timestamp": "2026-06-05T14:23:10Z"}
]
```

This is implemented via `appendEnquiryEvent` in [src/lib/repositories/enquiries.ts](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/repositories/enquiries.ts)

**Sources:**[src/api/booking.ts90-101](https://github.com/egac-web/egac/blob/0ba542fc/src/api/booking.ts#L90-L101)[src/lib/repositories/bookings.ts9-36](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/repositories/bookings.ts#L9-L36)[src/lib/repositories/invites.ts](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/repositories/invites.ts)[src/lib/repositories/enquiries.ts](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/repositories/enquiries.ts)

---

## Booking Confirmation Email

After the booking is created, the system sends a confirmation email with session details and an "Add to Calendar" link. Email delivery failures **do not block** the booking response.

### Confirmation Email Flow

[src/api/booking.ts104-136](https://github.com/egac-web/egac/blob/0ba542fc/src/api/booking.ts#L104-L136)

```
const venueName = config.venue_name ?? 'East Grinstead Leisure Centre';
const siteUrl = config.site_url ?? 'https://egac.pages.dev';
const confirmation = buildBookingConfirmation(enquiry, booking, ageGroup, venueName, siteUrl);
 
try {
  const emailResult = await sendBookingConfirmation(
    enquiry,
    booking,
    venueName,
    confirmation.calendarUrl,
    env
  );
 
  if (!emailResult.success) {
    console.error(/* log error but don't throw */);
  }
} catch (e) {
  console.error(/* log exception but don't throw */);
}
 
return ok({
  booking_id: booking.id,
  session_date: booking.session_date,
  // ... always returns 200
});
```

### Email Template Variables

The `booking_confirmation` email template receives these variables:
VariableExampleDescription`contact_name`"Jane Smith"Enquirer name`session_date`"Tuesday 10 June 2026"Formatted date`session_time`"18:30"Session time`slot_label`"U13 (Tuesday 18:30)"Age group label`venue`"East Grinstead Leisure Centre"Venue name`add_to_calendar_url`Google Calendar linkPre-filled event URL
### Calendar URL Generation

The `buildBookingConfirmation` function in [src/lib/business/booking.ts](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/business/booking.ts) generates a Google Calendar URL:

```
https://calendar.google.com/calendar/render?action=TEMPLATE
  &text=EGAC+Taster+Session+-+U13
  &dates=20260610T183000Z/20260610T200000Z
  &details=...
  &location=East+Grinstead+Leisure+Centre

```

This allows users to add the session to their calendar with one click.

**Sources:**[src/api/booking.ts104-136](https://github.com/egac-web/egac/blob/0ba542fc/src/api/booking.ts#L104-L136)[src/lib/business/booking.ts](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/business/booking.ts)[src/lib/email/templates.js](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/email/templates.js)[tests/unit/api.test.ts509-533](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/api.test.ts#L509-L533)

---

## Error Scenarios

The booking endpoint returns specific error codes for different failure modes. This enables the frontend to display contextual error messages.

**Error Code Reference Table:**
HTTP StatusCodeConditionUser Message400`INVALID_JSON`Malformed request bodyRequest body is required404`INVITE_NOT_FOUND`Token doesn't existInvite not found404`ENQUIRY_NOT_FOUND`Enquiry deleted/missingEnquiry not found409`INVITE_USED`Invite already acceptedThis invite has already been used409`DUPLICATE_BOOKING`Same invite+date existsYou already have a booking for this date409`SLOT_FULL`Capacity reachedThis session is full. Please choose a different date.410`INVITE_EXPIRED`Older than 14 daysThis invite has expired422`MISSING_TOKEN`Token field emptyInvite token is required422`MISSING_DATE`Date field emptySession date is required422`NO_AGE_GROUP`Age doesn't match any groupNo matching age group found for this date422`SLOT_INVALID`Date in past or wrong dayInvalid booking
### Error Response Format

```
{
  "ok": false,
  "error": "This session is full. Please choose a different date.",
  "code": "SLOT_FULL",
  "cors": true
}
```

The `code` field enables programmatic error handling in the frontend, while `error` provides a human-readable message.

**Sources:**[src/api/booking.ts25-146](https://github.com/egac-web/egac/blob/0ba542fc/src/api/booking.ts#L25-L146)[src/api/helpers.ts](https://github.com/egac-web/egac/blob/0ba542fc/src/api/helpers.ts)[tests/unit/api.test.ts443-492](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/api.test.ts#L443-L492)

---

## Data Model

**Diagram: Taster Booking Entity Relationships**

```
generates

used to create

owns

classified by

ENQUIRIES

string

id

PK

enq_*

string

email

UK

string

name

string

dob

YYYY-MM-DD

string

age_group_id

FK

string

invite_id

FK

string

events

JSON array

int

processed

0 or 1

string

environment

INVITES

string

id

PK

inv_*

string

token

UK

48-char hex

string

enquiry_id

FK

enum

status

pending,sent,accepted,expired,failed

int

send_attempts

datetime

created_at

datetime

sent_at

datetime

accepted_at

string

last_send_error

string

environment

BOOKINGS

string

id

PK

bkg_*

string

enquiry_id

FK

string

invite_id

FK

nullable

string

age_group_id

FK

date

session_date

YYYY-MM-DD

string

session_time

HH:MM

enum

status

confirmed,attended,no_show,cancelled

string

attendance_note

nullable

datetime

created_at

datetime

updated_at

string

environment

AGE_GROUPS

string

id

PK

ag_*

string

code

UK

u11,u13,u15...

string

label

U11,U13,U15...

enum

booking_type

taster,waitlist

int

age_min_aug31

int

age_max_aug31

string

session_days

JSON ['Tuesday']

string

session_time

HH:MM

int

capacity_per_session

int

active

0 or 1

int

sort_order
```

### Booking Lifecycle State Machine

```
createBooking()

Admin marks attended

Admin marks no-show

Admin/user cancels

confirmed

attended

no_show

cancelled
```

The booking status can only move forward; there is no reverting from `'attended'` back to `'confirmed'`. Admins manage status transitions via the bookings management interface (see [4.3](/egac-web/egac/4.3-bookings-management)).

**Sources:**[src/lib/db/schema.ts](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/db/schema.ts)[src/lib/repositories/bookings.ts1-165](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/repositories/bookings.ts#L1-L165)[src/lib/repositories/invites.ts](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/repositories/invites.ts)[src/lib/repositories/enquiries.ts](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/repositories/enquiries.ts)

---

## Available Session Dates

When the booking page loads, it must display a list of available session dates. This is generated server-side using the `getNextNSessionDates` function.

### Date Generation Logic

[src/lib/business/age.ts188-210](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/business/age.ts#L188-L210)

```
export function getNextNSessionDates(
  ageGroup: AgeGroupConfig,
  n: number,
  fromDate: Date = new Date()
): string[] {
  const validDays = getSessionDays(ageGroup);
  const dayNames = ['Sunday', 'Monday', 'Tuesday', 'Wednesday', 'Thursday', 'Friday', 'Saturday'];
  const validDayNumbers = validDays.map((d) => dayNames.indexOf(d)).filter((d) => d >= 0);
 
  const dates: string[] = [];
  const cursor = new Date(fromDate);
  cursor.setDate(cursor.getDate() + 1); // start from tomorrow
 
  while (dates.length < n) {
    if (validDayNumbers.includes(cursor.getDay())) {
      dates.push(toIsoDateString(cursor));
    }
    cursor.setDate(cursor.getDate() + 1);
  }
 
  return dates;
}
```

### Example Output

For a U13 taster group (session_days = `["Tuesday"]`), calling `getNextNSessionDates(ageGroup, 8)` might return:

```
[
  "2026-06-10",
  "2026-06-17",
  "2026-06-24",
  "2026-07-01",
  "2026-07-08",
  "2026-07-15",
  "2026-07-22",
  "2026-07-29"
]
```

The number of weeks ahead is configured via `system_config.weeks_ahead_booking`, defaulting to 8 weeks.

**Sources:**[src/lib/business/age.ts188-210](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/business/age.ts#L188-L210)[src/api/enquiry.ts189-190](https://github.com/egac-web/egac/blob/0ba542fc/src/api/enquiry.ts#L189-L190)[tests/unit/business.test.ts315-341](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/business.test.ts#L315-L341)

---

## Integration Points

### Prerequisites

Before a user can book a taster session:

1. **Enquiry submitted** (see [3.1](/egac-web/egac/3.1-enquiry-form)) - creates enquiry record
2. **Age group resolved** (see [6.1](/egac-web/egac/6.1-age-group-system)) - determines booking_type='taster'
3. **Invite created** - generates secure token
4. **Invite email sent** - user receives booking URL
5. **User clicks link** - arrives at booking page with token in URL

### Downstream Effects

After a successful booking:

1. **Booking record created** - stored in `bookings` table
2. **Invite marked accepted** - prevents reuse
3. **Enquiry event logged** - audit trail updated
4. **Confirmation email sent** - includes calendar link
5. **Booking appears in admin calendar** (see [4.3](/egac-web/egac/4.3-bookings-management))
6. **Capacity slot consumed** - reduces available spots

### Related Scheduled Jobs

- **Expire Invites** (see [9.2](/egac-web/egac/9.2-invite-management-jobs)) - marks invites older than 14 days as expired
- **Booking Reminders** (see [9.3](/egac-web/egac/9.3-booking-reminder-job)) - sends reminder emails 1 day before session

**Sources:**[src/api/enquiry.ts166-210](https://github.com/egac-web/egac/blob/0ba542fc/src/api/enquiry.ts#L166-L210)[src/api/booking.ts1-146](https://github.com/egac-web/egac/blob/0ba542fc/src/api/booking.ts#L1-L146)[src/lib/repositories/bookings.ts104-123](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/repositories/bookings.ts#L104-L123)