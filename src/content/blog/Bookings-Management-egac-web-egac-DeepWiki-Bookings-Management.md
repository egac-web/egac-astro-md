# Bookings Management
Relevant source files
- [docs/snagging.md](https://github.com/egac-web/egac/blob/0ba542fc/docs/snagging.md)
- [src/lib/repositories/age-groups.ts](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/repositories/age-groups.ts)
- [src/lib/repositories/bookings.ts](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/repositories/bookings.ts)
- [src/pages/admin/bookings.astro](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/bookings.astro)
- [src/pages/admin/reports.astro](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/reports.astro)
- [tests/unit/admin.test.ts](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/admin.test.ts)
- [tests/unit/cron.test.ts](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/cron.test.ts)

## Purpose and Scope

This document covers the **Bookings Management** functionality in the admin interface, which provides a calendar view of upcoming taster sessions, capacity tracking, and attendance management. Administrators use this interface to monitor bookings, mark athlete attendance, and optionally trigger membership invites after session completion.

For information about the public-facing booking flow, see [Taster Session Booking](/egac-web/egac/3.2-taster-session-booking). For how enquiries are created and routed, see [Enquiries Management](/egac-web/egac/4.2-enquiries-management). For the API implementation details, see [Admin APIs](/egac-web/egac/5.2-admin-apis).

---

## Overview

The Bookings Management system presents upcoming taster sessions in a calendar format, organized by date and age group. Each session displays:

- Current booking count vs. capacity
- List of booked athletes
- Attendance tracking controls for confirmed bookings

The interface auto-refreshes booking status via AJAX when administrators mark attendance, eliminating the need for full page reloads.

**Sources:**[src/pages/admin/bookings.astro1-53](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/bookings.astro#L1-L53)

---

## Calendar Structure

### Display Layout

The bookings calendar shows the next 4 Tuesday sessions, retrieved using `getNextNTuesdayDates(4)`. Each date is displayed as a card containing slots for all active taster age groups.

```
bookings.astro

checkAdminToken

Load Data Layer

Build Calendar Structure

Render Calendar Grid

listAgeGroups
(taster only)

getNextNTuesdayDates(4)

listUpcomingBookings(4)

getEnquiryById
(for each booking)

DayData[]
{date, displayDate, slots[]}

SlotData[]
{ageGroup, bookings[]}

Grid: 1-4 columns
(responsive)

Day Card
(formatDateForDisplay)

Slot Card
(age group + capacity bar)

Booking List
(with attendance controls)
```

**Data Structure:**
TypeFieldsPurpose`EnrichedBooking``Booking` + `enquiry_name`, `enquiry_email`Combines booking data with athlete information for display`SlotData``ageGroup: AgeGroup`, `bookings: EnrichedBooking[]`Groups bookings by age group for a specific date`DayData``date: string`, `displayDate: string`, `slots: SlotData[]`Represents one Tuesday with all age group slots
**Sources:**[src/pages/admin/bookings.astro19-52](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/bookings.astro#L19-L52)

---

## Capacity Visualization

Each slot card displays a progress bar showing booking capacity utilization:

```
Yes

No

Slot Card

Calculate Metrics

booked = bookings.length

capacity = ageGroup.capacity_per_session

pct = (booked / capacity) * 100

Progress Bar

pct >= 100?

bg-red-600

bg-blue-800

Display: 'X / Y booked'
```

The capacity value comes from `age_groups.capacity_per_session`, which defaults to 2 per session if not specified. The progress bar turns red when capacity is reached or exceeded.

**Sources:**[src/pages/admin/bookings.astro83-110](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/bookings.astro#L83-L110)

---

## Attendance Tracking

### Status Lifecycle

Bookings progress through the following status values:

```
createBooking

Mark attended

Mark no show

(not used in UI)

confirmed

attended

no_show

cancelled
```

Only bookings with `status = 'confirmed'` display attendance control buttons in the UI.

**Sources:**[src/pages/admin/bookings.astro144-163](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/bookings.astro#L144-L163)[src/lib/repositories/bookings.ts125-143](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/repositories/bookings.ts#L125-L143)

---

## Attendance Update Flow

When an administrator clicks "Mark attended" or "No show", the following sequence executes:

```
"Email System"
"bookingRepo.updateBookingStatus"
"/api/admin/bookings/:id/attendance"
"bookings.astro
(client script)"
Admin
"Email System"
"bookingRepo.updateBookingStatus"
"/api/admin/bookings/:id/attendance"
"bookings.astro
(client script)"
Admin
alt
[status === 'attended']
alt
[send_membership_link === true]
Click attend-btn or noshow-btn
confirm("Send membership invite?")
true/false
Disable button
POST {status, send_membership_link}
updateBookingStatus(id, status, note)
UPDATE bookings SET status, updated_at
sendMembershipInvite(enquiry)
200 OK
Update badge text & color
Remove attendance-controls
showToast("Marked attended")
```

### Implementation Details

The attendance update is handled by inline JavaScript in the Astro page:

**Button Handlers:**

- Both `.attend-btn` and `.noshow-btn` have event listeners attached (line 192)
- Button elements store `data-booking-id` and `data-status` attributes
- When `status === 'attended'`, a confirmation dialog asks about sending membership invite

**API Request:**

```
POST /api/admin/bookings/${bookingId}/attendance
Content-Type: application/json
{
  "status": "attended" | "no_show",
  "send_membership_link": boolean
}
```

**UI Updates on Success:**

- Badge text changes to "Attended" or "No show"
- Badge styling updates (green-100/green-700 for attended, red-100/red-700 for no show)
- Attendance control buttons are removed from the DOM
- Toast notification confirms action

**Sources:**[src/pages/admin/bookings.astro192-251](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/bookings.astro#L192-L251)

---

## Membership Invite Option

When marking a booking as "attended", administrators can optionally trigger a membership invite email:

```

```

This feature allows coaches to selectively send membership forms only to athletes who show genuine interest, rather than automatically sending to all attendees.

**Sources:**[src/pages/admin/bookings.astro203-206](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/bookings.astro#L203-L206)

---

## Data Model

### Booking Entity

The `bookings` table stores all taster session bookings:
ColumnTypeDescription`id`string (PK)Generated with `bkg_` prefix`enquiry_id`string (FK)References `enquiries.id``invite_id`string (FK, nullable)References `invites.id` if booked via invite`age_group_id`string (FK)References `age_groups.id``session_date`dateSession date (YYYY-MM-DD format)`session_time`string (nullable)Session time (e.g., "18:30")`status`enum`confirmed`, `attended`, `no_show`, `cancelled``attendance_note`text (nullable)Optional admin note`created_at`timestampBooking creation time`updated_at`timestamp (nullable)Last status change time`environment`string`development`, `staging`, or `production`
**Relationships:**

- Each booking belongs to one enquiry (athlete)
- Each booking may be associated with one invite (if booked via invite token)
- Each booking is assigned to one age group (determines session time/day)

**Sources:**[src/lib/repositories/bookings.ts9-36](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/repositories/bookings.ts#L9-L36)

---

## Repository Functions

### Core Booking Operations

The `bookings.ts` repository provides the following data access functions:

```
Update Operations

Read Operations

Create Operations

Analytics Operations

countBookingsForDateAndGroup(date, ageGroupId)

countBookingsByStatus(status, sinceDate?)

createBooking(enquiryId, inviteId,
ageGroupId, sessionDate, sessionTime)

getBookingById(id)

getBookingByInviteAndDate(inviteId, date)

listBookingsForDate(date)

listBookingsByEnquiry(enquiryId)

listUpcomingBookings(weeksAhead)

updateBookingStatus(id, status, note)

INSERT INTO bookings

UPDATE bookings SET status

SELECT * FROM bookings
WHERE session_date BETWEEN ? AND ?
AND status IN ('confirmed','attended','no_show')
```

### Key Function: `listUpcomingBookings`

This function powers the bookings calendar by retrieving all bookings within a specified time window:

**Logic:**

1. Calculate date range: today + (weeksAhead * 7 days)
2. Query bookings with `session_date` in range
3. Include only `confirmed`, `attended`, and `no_show` statuses
4. Filter by current environment
5. Order by `session_date ASC`

**Usage in Calendar:**

```
const upcomingBookings = await listUpcomingBookings(4, env);
// Returns all bookings for next 4 weeks
```

**Sources:**[src/lib/repositories/bookings.ts104-123](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/repositories/bookings.ts#L104-L123)

---

## Query Patterns

### Capacity Checking

Before creating a booking, the system checks available capacity:

```
countBookingsForDateAndGroup(sessionDate, ageGroupId, env)
// Returns: number of confirmed bookings
// Compare against: ageGroup.capacity_per_session
```

**Sources:**[src/lib/repositories/bookings.ts62-78](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/repositories/bookings.ts#L62-L78)

### Status-Based Reporting

The reports dashboard uses `countBookingsByStatus` to generate metrics:

```
// Total confirmed bookings
await countBookingsByStatus('confirmed', env)
 
// Attended this month
await countBookingsByStatus('attended', env, thisMonthStart)
 
// No-shows this month
await countBookingsByStatus('no_show', env, thisMonthStart)
```

This enables calculation of attendance rates and conversion funnels.

**Sources:**[src/lib/repositories/bookings.ts145-164](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/repositories/bookings.ts#L145-L164)[src/pages/admin/reports.astro48-70](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/reports.astro#L48-L70)

---

## Toast Notifications

The bookings calendar uses a toast notification system for user feedback:

```
200 OK

Error

Attendance Action

API Response

showToast(message, true)
bg-green-700

showToast(message, false)
bg-red-600

Display toast 3 seconds

Auto-hide with setTimeout
```

**Toast Messages:**

- `"Marked attended & membership invite sent."` - When attended with email
- `"Marked attended."` - When attended without email
- `"Marked as no show."` - When marking no show
- `"Failed to record attendance."` - On API error
- `"Network error — please try again."` - On fetch error

**Sources:**[src/pages/admin/bookings.astro180-190](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/bookings.astro#L180-L190)[src/pages/admin/bookings.astro235-248](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/bookings.astro#L235-L248)

---

## Integration Points

### With Enquiries System

Each booking is linked to an enquiry via `enquiry_id`. The calendar enriches booking data by fetching enquiry details:

```
const enriched = await Promise.all(
  upcomingBookings.map(async (b) => {
    const enquiry = await getEnquiryById(b.enquiry_id, env);
    return { 
      ...b, 
      enquiry_name: enquiry?.name ?? null, 
      enquiry_email: enquiry?.email ?? '' 
    };
  })
);
```

This allows displaying athlete names instead of just IDs in the calendar.

**Sources:**[src/pages/admin/bookings.astro26-32](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/bookings.astro#L26-L32)

### With Age Groups System

Age groups determine:

- Which slots appear in the calendar (only `booking_type = 'taster'` groups)
- Session capacity (`capacity_per_session` field)
- Session timing (`session_time` field)
- Which days are available (`session_days` JSON field)

The calendar filters to active taster groups:

```
const tasterGroups = ageGroups.filter(
  (ag) => ag.booking_type === 'taster' && ag.active
);
```

**Sources:**[src/pages/admin/bookings.astro20-21](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/bookings.astro#L20-L21)

### With Email System

When marking attended with membership invite enabled, the system:

1. Creates a record in `membership_otps` table with secure token
2. Sends email via `sendMembershipInvite(enquiry, token, config)`
3. Email contains link to membership form with pre-filled data

This is optional and controlled by administrator choice at time of marking attendance.

**Sources:**[src/pages/admin/bookings.astro203-206](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/bookings.astro#L203-L206)

---

## Authentication

All bookings management operations require admin authentication:

```
const token = Astro.cookies.get('egac_admin_token')?.value ?? '';
if (!checkAdminToken(token, env)) {
  return Astro.redirect('/admin/login');
}
```

The `checkAdminToken` function validates against `env.ADMIN_TOKEN` or accepts `"dev"` token in non-production environments.

**Sources:**[src/pages/admin/bookings.astro14-17](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/bookings.astro#L14-L17)

---

## Responsive Design

The calendar uses a responsive grid layout:

```
<div class="grid grid-cols-1 gap-4 md:grid-cols-2 xl:grid-cols-4">
```

- Mobile: 1 column (single day per row)
- Tablet: 2 columns (2 days side-by-side)
- Desktop: 4 columns (all 4 Tuesdays visible)

**Sources:**[src/pages/admin/bookings.astro73](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/bookings.astro#L73-L73)

---

## Testing Coverage

The bookings system is tested through multiple test suites:

**Admin Handler Tests:**

- Authentication enforcement
- Attendance marking with membership invite option
- Status updates and UI response
- Error handling for missing bookings

**Cron Job Tests:**

- Booking reminder emails (7 days before session)
- Idempotency via event log checking

**Sources:**[tests/unit/admin.test.ts1-583](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/admin.test.ts#L1-L583)[tests/unit/cron.test.ts355-447](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/cron.test.ts#L355-L447)