# Session Scheduling
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

## Purpose and Scope

This document describes how the EGAC platform generates, validates, and manages session dates for taster sessions and academy training. Session scheduling determines which dates are available for booking, enforces day-of-week constraints, and ensures sessions do not exceed capacity limits.

For information about how age groups are resolved from athlete date of birth, see [Age Group System](/egac-web/egac/6.1-age-group-system). For the enquiry routing logic that determines whether an athlete gets taster sessions or academy waitlist placement, see [Enquiry Routing](/egac-web/egac/6.2-enquiry-routing).

---

## Overview

Session scheduling operates on two core principles:

1. **Age group configuration** defines valid session days, times, and capacity constraints
2. **Pure business logic functions** generate and validate session dates without database I/O

Each age group in the `age_groups` table specifies:

- `session_days`: JSON array of valid weekday names (e.g., `["Tuesday"]` or `["Saturday","Tuesday"]`)
- `session_time`: Start time for the session (e.g., `"18:30"`)
- `capacity_per_session`: Maximum number of confirmed bookings per session

The scheduling system ensures athletes can only book sessions that:

- Fall on a valid day for their age group
- Occur in the future (never today or past dates)
- Have not reached capacity

**Sources:**[src/lib/business/age.ts1-271](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/business/age.ts#L1-L271)[src/lib/repositories/age-groups.ts1-118](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/repositories/age-groups.ts#L1-L118)

---

## Session Date Generation

### Core Function: `getNextNSessionDates`

The primary date generation function returns the next N upcoming valid session dates for an age group, starting from tomorrow.

```
Yes

Yes

No

No

getNextNSessionDates(ageGroup, n, fromDate)

Parse session_days JSON
Extract valid day names

Convert day names to numbers
[Sunday=0, Tuesday=2, etc.]

Initialize cursor = fromDate + 1 day
(Never return today)

dates.length < n?

cursor.getDay()
in validDayNumbers?

Add date to results array
toIsoDateString(cursor)

cursor.setDate(cursor.getDate() + 1)

Return array of ISO date strings
```

**Sources:**[src/lib/business/age.ts188-210](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/business/age.ts#L188-L210)

### Implementation Details
AspectBehavior**Input**`AgeGroupConfig`, count `n`, optional `fromDate`**Output**Array of ISO date strings (`YYYY-MM-DD`)**Starting point**Tomorrow (always excludes today)**Day filtering**Only dates matching `session_days` configuration**Determinism**Pure function — same inputs always yield same outputs
**Example usage in enquiry flow:**

When a taster age group is resolved, the enquiry handler generates available dates:

```
// From src/api/enquiry.ts:189-191
const weeksAhead = parseInt(config.weeks_ahead_booking ?? '8', 10);
const availableDates = getNextNTuesdayDates(weeksAhead);
const invite = await createInviteForEnquiry(enquiry.id, env);
```

**Sources:**[src/lib/business/age.ts188-210](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/business/age.ts#L188-L210)[src/api/enquiry.ts188-191](https://github.com/egac-web/egac/blob/0ba542fc/src/api/enquiry.ts#L188-L191)

---

## Backwards Compatibility Wrapper

The `getNextNTuesdayDates` function exists for backwards compatibility with the original hardcoded Tuesday-only system. It creates a synthetic age group configuration and delegates to `getNextNSessionDates`:

```

```

**Sources:**[src/lib/business/age.ts213-231](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/business/age.ts#L213-L231)

---

## Session Date Validation

### Validation Rules

The `validateSessionDate` function enforces three requirements before a booking can proceed:

```
Invalid

Valid

No

Yes

No

Yes

validateSessionDate(dateIso, ageGroup, today)

Parse date string
Valid ISO format?

Return false

isValidSessionDay
(date, ageGroup)?

Return false

date > today
(midnight comparison)?

Return false

Return true
```

**Sources:**[src/lib/business/age.ts240-253](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/business/age.ts#L240-L253)

### Day-of-Week Validation

The `isValidSessionDay` function checks if a date falls on an allowed weekday:
StepLogic1Parse `session_days` JSON from age group configuration2Extract day-of-week from the provided date3Check if day name exists in the valid days array
**Example configurations:**

```
// Taster groups (Tuesday only)
{ "session_days": "[\"Tuesday\"]" }
 
// Academy groups (Saturday and Tuesday)
{ "session_days": "[\"Saturday\",\"Tuesday\"]" }
```

**Sources:**[src/lib/business/age.ts171-180](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/business/age.ts#L171-L180)

---

## Capacity Management

### Booking Count Query

Before creating a booking, the system queries current capacity for the requested session slot:

```
Client
System Config
D1 Database
bookingRepo
handleBooking
Client
System Config
D1 Database
bookingRepo
handleBooking
alt
[currentCount >= capacity]
[slot available]
getConfig(env)
weeks_ahead_booking, capacity defaults
countBookingsForDateAndGroup(date, ageGroupId, env)
SELECT COUNT(*) FROM bookings
WHERE session_date = ?
AND age_group_id = ?
AND status = 'confirmed'
count result
currentCount
Compare currentCount vs capacity_per_session
409 SLOT_FULL
createBooking(...)
```

**Sources:**[src/lib/repositories/bookings.ts62-78](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/repositories/bookings.ts#L62-L78)[src/api/booking.ts81-87](https://github.com/egac-web/egac/blob/0ba542fc/src/api/booking.ts#L81-L87)

### Capacity Resolution Logic

The booking handler determines capacity using this precedence:

```
Yes

No

Determine capacity limit

ageGroup.capacity_per_session
!== null?

Use ageGroup.capacity_per_session

Use config.weeks_ahead_booking
(fallback default: 2)

Apply capacity limit
```

**Sources:**[src/api/booking.ts81-87](https://github.com/egac-web/egac/blob/0ba542fc/src/api/booking.ts#L81-L87)

### Capacity Check Query

The repository function counts only `confirmed` bookings to determine current utilization:

```
SELECT COUNT(*) as count FROM bookings
WHERE session_date = ?
  AND age_group_id = ?
  AND status = 'confirmed'
  AND environment = ?
```

**Excluded statuses:**`cancelled`, `attended`, `no_show` do not count against capacity.

**Sources:**[src/lib/repositories/bookings.ts62-78](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/repositories/bookings.ts#L62-L78)

---

## Integration with Age Groups

### Age Group Configuration Schema

Session scheduling relies on these `age_groups` table fields:
FieldTypePurpose`session_days``TEXT` (JSON)Valid weekdays for this age group`session_time``TEXT`Start time (e.g., `"18:30"`)`capacity_per_session``INTEGER`Max bookings per session slot`booking_type``TEXT``"taster"` or `"waitlist"` — determines if scheduling applies
**Sources:**[src/lib/business/age.ts90-105](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/business/age.ts#L90-L105)

### Configuration Flow

```
API Layer

Business Logic

Repository Layer

D1 Database

age_groups table

listAgeGroups(env)

getAgeGroupById(id, env)

getSessionDays(ageGroup)
Parse session_days JSON

getNextNSessionDates(ageGroup, n)
Generate future dates

validateSessionDate(date, ageGroup)
Check day + future

POST /api/enquiry
Generate invite dates

POST /api/booking
Validate requested date
```

**Sources:**[src/lib/business/age.ts158-210](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/business/age.ts#L158-L210)[src/lib/repositories/age-groups.ts9-27](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/repositories/age-groups.ts#L9-L27)

---

## Practical Examples

### Example 1: Taster Session Date Generation

For a U13 age group with Tuesday sessions, generating 8 weeks of available dates:

```
// Age group configuration from D1:
const u13Group = {
  id: 'ag_u13',
  code: 'u13',
  label: 'U13',
  booking_type: 'taster',
  session_days: '["Tuesday"]',
  session_time: '18:30',
  capacity_per_session: 2,
  // ...other fields
};
 
// Generate next 8 Tuesdays
const dates = getNextNSessionDates(u13Group, 8);
// Returns: ['2026-06-10', '2026-06-17', '2026-06-24', ...]
```

**Sources:**[tests/unit/business.test.ts315-323](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/business.test.ts#L315-L323)

### Example 2: Academy Multi-Day Scheduling

For an academy age group with Saturday and Tuesday sessions:

```
// Age group configuration:
const u11Academy = {
  id: 'ag_u11',
  code: 'u11',
  label: 'U11 Academy',
  booking_type: 'waitlist',
  session_days: '["Saturday","Tuesday"]',
  capacity_per_session: null, // Uses season-based capacity instead
  // ...other fields
};
 
// Generate next 6 session dates (mix of Saturdays and Tuesdays)
const dates = getNextNSessionDates(u11Academy, 6);
// Returns: ['2026-06-07', '2026-06-10', '2026-06-14', '2026-06-17', ...]
//           (Sat)        (Tue)        (Sat)        (Tue)
```

**Sources:**[tests/unit/business.test.ts325-333](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/business.test.ts#L325-L333)

### Example 3: Booking Validation Flow

Complete validation sequence when a user attempts to book a session:

```
D1
bookings.ts
age.ts functions
POST /api/booking
User
D1
bookings.ts
age.ts functions
POST /api/booking
User
POST {token, date: "2026-06-17"}
resolveAgeGroup(dob, "2026-06-17", allGroups)
ageGroup {session_days: '["Tuesday"]', ...}
validateSessionDate("2026-06-17", ageGroup)
Check 1: Valid ISO date? ✓
Check 2: Falls on Tuesday? ✓
Check 3: Future date? ✓
true
countBookingsForDateAndGroup("2026-06-17", "ag_u13", env)
SELECT COUNT(*) WHERE session_date = "2026-06-17"
AND age_group_id = "ag_u13" AND status = "confirmed"
count: 1
1
Check capacity: 1 < 2 ✓
createBooking(enquiryId, inviteId, "ag_u13", "2026-06-17", "18:30", env)
200 OK {booking_id, session_date, ...}
```

**Sources:**[src/api/booking.ts25-146](https://github.com/egac-web/egac/blob/0ba542fc/src/api/booking.ts#L25-L146)[src/lib/business/age.ts240-253](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/business/age.ts#L240-L253)

---

## Error Handling

### Validation Error Codes
Validation FailureHTTP StatusError CodeReasonInvalid date format422`SLOT_INVALID`Date string unparseableWrong day of week422`SLOT_INVALID`Date not in `session_days`Past date422`SLOT_INVALID`Date is today or earlierNo matching age group422`NO_AGE_GROUP`Athlete's age doesn't match any active groupSession at capacity409`SLOT_FULL``currentCount >= capacity_per_session`Duplicate booking409`DUPLICATE_BOOKING`Same invite + date already booked
**Sources:**[src/api/booking.ts64-87](https://github.com/egac-web/egac/blob/0ba542fc/src/api/booking.ts#L64-L87)[tests/unit/api.test.ts460-477](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/api.test.ts#L460-L477)

### Graceful Degradation

When `session_days` JSON parsing fails, the system falls back to Tuesday-only:

```
// From getSessionDays function
export function getSessionDays(ageGroup: AgeGroupConfig): string[] {
  try {
    return JSON.parse(ageGroup.session_days) as string[];
  } catch {
    return ['Tuesday']; // Safe default
  }
}
```

**Sources:**[src/lib/business/age.ts158-164](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/business/age.ts#L158-L164)

---

## Testing Strategy

The business logic functions are extensively tested with pure unit tests (no database mocking required):

```
Pure Functions Under Test

Test Coverage

getNextNSessionDates
Returns N future dates

validateSessionDate
Rejects past dates

validateSessionDate
Rejects wrong weekday

getSessionDays
Parses JSON correctly

Multi-day scheduling
Saturday + Tuesday mix

Edge case: Never returns today

getNextNSessionDates

validateSessionDate

getSessionDays

isValidSessionDay
```

**Sources:**[tests/unit/business.test.ts276-341](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/business.test.ts#L276-L341)

---

## Summary

Session scheduling in the EGAC platform is driven by database-configured age groups and implemented through pure business logic functions. The system ensures:

1. **Configurability**: Administrators can adjust session days, times, and capacity per age group
2. **Type safety**: All functions operate on typed `AgeGroupConfig` interfaces
3. **Predictability**: Pure functions with no I/O enable comprehensive testing
4. **Flexibility**: Supports single-day (Tuesday) and multi-day (Saturday + Tuesday) scheduling
5. **Capacity enforcement**: Prevents overbooking by counting confirmed bookings per session slot

**Key code entities:**

- `getNextNSessionDates(ageGroup, n, fromDate)` — generates future session dates
- `validateSessionDate(dateIso, ageGroup, today)` — validates booking requests
- `countBookingsForDateAndGroup(date, ageGroupId, env)` — capacity check query
- `session_days`, `session_time`, `capacity_per_session` — age group configuration fields

**Sources:**[src/lib/business/age.ts1-271](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/business/age.ts#L1-L271)[src/lib/repositories/bookings.ts1-165](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/repositories/bookings.ts#L1-L165)[src/api/booking.ts1-147](https://github.com/egac-web/egac/blob/0ba542fc/src/api/booking.ts#L1-L147)