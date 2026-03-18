---
title: "Enquiries Management"
description: "This page documents the admin enquiries list interface"
pubDate: 2026-03-18
---
# Enquiries Management
Relevant source files
- [src/layouts/AdminBase.astro](https://github.com/egac-web/egac/blob/0ba542fc/src/layouts/AdminBase.astro)
- [src/pages/admin/enquiries.astro](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/enquiries.astro)
- [tests/unit/admin.test.ts](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/admin.test.ts)
- [tests/unit/cron.test.ts](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/cron.test.ts)

This page documents the admin enquiries list interface, which provides administrators with a searchable, filterable view of all enquiry submissions. The interface supports status-based filtering, full-text search, pagination, and quick access to detailed enquiry views.

For information about the enquiry detail view and editing capabilities, see [Admin Interface](/egac-web/egac/4-admin-interface). For the public enquiry submission flow, see [Enquiry Form](/egac-web/egac/3.1-enquiry-form). For API endpoint specifications, see [Admin APIs](/egac-web/egac/5.2-admin-apis).

---

## Overview

The enquiries management interface is located at `/admin/enquiries` and serves as the primary dashboard for viewing and triaging all athlete enquiry submissions. Administrators can filter by processing status, search by name or email, and access individual enquiry records for detailed review.

**Sources:**[src/pages/admin/enquiries.astro1-314](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/enquiries.astro#L1-L314)

---

## Page Structure and Authentication

The enquiries list page is an Astro component that inherits layout and navigation from `AdminBase.astro`. Server-side authentication occurs before any data is loaded.

```
Invalid

Valid

HTTP GET /admin/enquiries

checkAdminToken()

egac_admin_token cookie

Redirect to /admin/login

Load enquiries + age groups

Render table
```

**Authentication Check:**

The page reads the `egac_admin_token` cookie and validates it using `checkAdminToken()`. Invalid tokens trigger a redirect to `/admin/login`.

**Sources:**[src/pages/admin/enquiries.astro14-20](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/enquiries.astro#L14-L20)

---

## Data Loading Architecture

The page performs parallel data loading to efficiently retrieve enquiries, their associated invites, bookings, and age group metadata.

```
D1 Database
listBookingsByEnquiry()
getInviteById()
listAgeGroups()
listEnquiries()
enquiries.astro
D1 Database
listBookingsByEnquiry()
getInviteById()
listAgeGroups()
listEnquiries()
enquiries.astro
par
[Load base data]
alt
[Has invite_id]
par
[Load related data]
loop
[For each enquiry]
Parse URL params (status, search, page)
Build EnquiryFilters object
listEnquiries(filters, env)
SELECT with WHERE + LIMIT + OFFSET
rows[]
Enquiry[]
listAgeGroups(env)
SELECT * FROM age_groups
age_groups[]
AgeGroup[]
Build ageGroupMap
getInviteById(invite_id, env)
SELECT
Invite
Invite or null
listBookingsByEnquiry(enq.id, env)
SELECT WHERE enquiry_id
Booking[]
Booking[]
Extract latest booking status
Render table with all data
```

**Key Functions:**

- `listEnquiries(filters, env)`: Retrieves enquiries matching filter criteria with pagination support
- `listAgeGroups(env)`: Loads all age group definitions for path classification
- `getInviteById(invite_id, env)`: Fetches invite status for enquiries with pending/sent invites
- `listBookingsByEnquiry(enquiry_id, env)`: Retrieves all bookings to determine latest booking status

**Sources:**[src/pages/admin/enquiries.astro32-83](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/enquiries.astro#L32-L83)

---

## URL Parameters and Filtering

The page supports three query parameters that control data retrieval and display:
ParameterTypeDescriptionDefault`status`stringFilter by processing status: `all`, `pending`, `processed`, `academy``all``search`stringFull-text search across name and email fieldsempty`page`numberPagination offset (1-indexed)`1`
**Filter Object Construction:**

```
const filters: EnquiryFilters = {
  limit: PAGE_SIZE + 1,  // Fetch one extra to detect hasNextPage
  offset: (page - 1) * PAGE_SIZE,
  ...(rawStatus === 'pending' || rawStatus === 'processed' || rawStatus === 'academy'
    ? { status: rawStatus }
    : {}),
  ...(search ? { search } : {}),
};
```

The `limit` is set to `PAGE_SIZE + 1` (21 rows) to determine if additional pages exist. Only the first 20 rows are displayed.

**Sources:**[src/pages/admin/enquiries.astro22-46](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/enquiries.astro#L22-L46)

---

## Table Structure and Columns

The enquiries table displays eight columns derived from multiple data sources:

```
Table Columns

Data Sources

enquiries table

invites table

bookings table

age_groups table

Date

Name

Email

DOB

Path
(Taster/Academy)

Invite Status

Booking Status

Actions
(View/Resend)
```

### Column Implementations

**Date Column:**
Displays `enquiry.created_at` formatted as `DD/MM/YYYY` using `formatDate()` helper.

**Path Column:**
Determines if enquiry is routed to "Taster" or "Academy" path by checking:

1. `age_groups.booking_type === 'waitlist'` for the enquiry's age group
2. `enquiry.source` contains "academy" (case-insensitive)

**Invite Status Column:**
Shows badge with one of five states based on `invite.status`:

- `pending`: Yellow badge "Pending"
- `sent`: Blue badge "Sent"
- `accepted`: Green badge "Accepted"
- `expired`: Gray badge "Expired"
- `failed`: Red badge "Failed"
- No invite: Gray badge "None"

**Booking Status Column:**
Displays latest booking status from `listBookingsByEnquiry()` results, sorted by `session_date DESC, created_at DESC`:

- `attended`: Emerald badge
- `no_show`: Rose badge (displays as "No show")
- `cancelled`: Slate badge
- `confirmed`: Blue badge
- No bookings: Gray dash "—"

**Sources:**[src/pages/admin/enquiries.astro86-108](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/enquiries.astro#L86-L108)[src/pages/admin/enquiries.astro180-276](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/enquiries.astro#L180-L276)

---

## Status Filter Tabs

The interface provides four filter tabs above the table:

```
All
No status filter

Pending
processed = 0

Processed
processed = 1

Academy
waitlist-specific

?status=all

?status=pending

?status=processed

?status=academy
```

**Tab URL Generation:**

The `tabUrl()` helper constructs URLs that:

1. Set the `status` query parameter
2. Reset pagination to page 1 (`page` param deleted)
3. Preserve existing `search` parameter if present

Active tab styling applies when `rawStatus === tab.id`.

**Sources:**[src/pages/admin/enquiries.astro117-130](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/enquiries.astro#L117-L130)[src/pages/admin/enquiries.astro153-170](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/enquiries.astro#L153-L170)

---

## Search Functionality

The search form submits a GET request with the `search` query parameter. The backend performs case-insensitive matching against `name` and `email` columns.

**Form Implementation:**

```
<form method="GET">
  <input type="hidden" name="status" value={rawStatus} />
  <input
    type="search"
    name="search"
    value={search}
    placeholder="Search..."
  />
</form>
```

The form preserves the current `status` filter via a hidden input, ensuring search results remain scoped to the active filter tab.

**Sources:**[src/pages/admin/enquiries.astro139-148](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/enquiries.astro#L139-L148)

---

## Pagination Controls

Pagination uses offset-based paging with a fixed `PAGE_SIZE` of 20 records. The interface detects additional pages by fetching `PAGE_SIZE + 1` rows.

```
Yes

No

Yes

No

Fetch 21 rows
(PAGE_SIZE + 1)

rows.length > 20?

Slice to first 20
rows.slice(0, PAGE_SIZE)

hasNextPage = true

hasNextPage = false

page > 1 OR
hasNextPage?

Render prev/next buttons

Hide pagination
```

**Navigation Button Logic:**

- **Previous**: Enabled when `page > 1`, links to `pageUrl(page - 1)`
- **Next**: Enabled when `hasNextPage === true`, links to `pageUrl(page + 1)`

The `pageUrl()` helper preserves all existing query parameters while updating only the `page` parameter.

**Sources:**[src/pages/admin/enquiries.astro27-30](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/enquiries.astro#L27-L30)[src/pages/admin/enquiries.astro45-46](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/enquiries.astro#L45-L46)[src/pages/admin/enquiries.astro111-115](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/enquiries.astro#L111-L115)[src/pages/admin/enquiries.astro281-311](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/enquiries.astro#L281-L311)

---

## Actions Column

Each enquiry row provides two action links:

### View Action

Links to `/admin/enquiries/{enquiry.id}` for detailed enquiry view, which displays:

- Full enquiry record with all fields
- Event timeline (parsed from `events` JSON)
- Associated invite details
- All bookings for the enquiry
- Note editing interface

### Resend Action

Currently links to the same detail page (implementation placeholder). The resend functionality is handled by the `POST /api/admin/invites/resend` endpoint, which:

1. Reuses existing `pending` or `sent` invites
2. Creates new invites if none exist or previous invite expired
3. Sends booking invitation email with available session dates

**Sources:**[src/pages/admin/enquiries.astro259-269](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/enquiries.astro#L259-L269)

---

## Related API Endpoints

The enquiries management interface is supported by several admin API handlers:

### GET /api/admin/enquiries

**Handler:**`handleAdminEnquiries(request, env)`

Lists enquiries with filtering and pagination. Accepts query parameters:

- `status`: Filter by processing status
- `search`: Full-text search term
- `page`: Pagination offset
- `limit`: Records per page (internal use)

Returns:

```
{
  "ok": true,
  "data": {
    "enquiries": Enquiry[],
    "page": 1,
    "hasNextPage": false
  }
}
```

### POST /api/admin/enquiries

**Handler:**`handleAdminEnquiries(request, env)`

Creates manual enquiry submissions. Requires JSON body:

```
{
  "name": "Athlete Name",
  "email": "athlete@example.com",
  "dob": "2015-03-15"
}
```

Returns existing enquiry if email already registered (`already_existed: true`) or creates new record with `source: 'admin'`.

### POST /api/admin/invites/resend

**Handler:**`handleAdminInviteResend(request, env)`

Resends booking invitation for an enquiry. Requires:

```
{
  "enquiry_id": "enq_abc123"
}
```

**Logic Flow:**

```
No

Yes

Yes

No

Yes

No

POST /api/admin/invites/resend

getEnquiryById(enquiry_id)

Enquiry exists?

404 NOT_FOUND

getInviteByEnquiryId(enquiry_id)

Invite exists?

status in
[pending, sent]?

Reuse existing invite
reused: true

createInviteForEnquiry()
reused: false

sendBookingInvite()

markInviteSent()

appendEnquiryEvent()
type: invite_resent

200 OK
```

**Sources:**[tests/unit/admin.test.ts463-582](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/admin.test.ts#L463-L582)

---

## Authentication Requirements

All admin endpoints require authentication via the `Authorization` header or `x-admin-token` header:

```
Authorization: Bearer {ADMIN_TOKEN}

```

Or:

```
x-admin-token: {ADMIN_TOKEN}

```

In development environments (`APP_ENV === 'development'`), the special token `"dev"` is accepted for convenience. Production deployments reject this token.

**Token Validation:**

The `checkAdminToken()` function in `src/lib/auth/admin.js` performs validation:

1. Extract token from `Authorization: Bearer {token}` or `x-admin-token` header
2. Accept `ADMIN_TOKEN` from environment in any environment
3. Accept `"dev"` token only when `APP_ENV === 'development'`
4. Return `401 UNAUTHORIZED` for invalid tokens

**Sources:**[tests/unit/admin.test.ts167-215](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/admin.test.ts#L167-L215)

---

## Data Model References

The enquiries list interface works with the following database entities:

### Enquiry

Primary entity loaded by `listEnquiries()`:

- `id`: Primary key (e.g., `enq_abc123`)
- `created_at`: Submission timestamp
- `email`: Unique contact email
- `name`: Athlete name
- `dob`: Date of birth (YYYY-MM-DD)
- `age_group_id`: Foreign key to `age_groups`
- `invite_id`: Foreign key to `invites` (nullable)
- `source`: Origin of enquiry (`website`, `admin`, etc.)
- `processed`: Integer flag (0 = pending, 1 = processed)
- `events`: JSON array of event objects for audit trail
- `note`: Admin notes (editable)

### Invite

Loaded via `getInviteById()` when `enquiry.invite_id` is present:

- `id`: Primary key (e.g., `inv_abc123`)
- `status`: Enum: `pending`, `sent`, `accepted`, `expired`, `failed`
- `token`: Secure booking URL token
- `enquiry_id`: Foreign key to enquiries
- `send_attempts`: Count of email send attempts
- `created_at`, `sent_at`, `accepted_at`: Lifecycle timestamps

### Booking

Loaded via `listBookingsByEnquiry()` to determine latest status:

- `id`: Primary key (e.g., `bkg_abc123`)
- `enquiry_id`: Foreign key
- `status`: Enum: `confirmed`, `attended`, `no_show`, `cancelled`
- `session_date`: Taster session date (YYYY-MM-DD)
- `age_group_id`: Foreign key to age_groups

### AgeGroup

Loaded via `listAgeGroups()` to build lookup map:

- `id`: Primary key (e.g., `ag_u13`)
- `code`: Human-readable code (e.g., `U13`)
- `booking_type`: Enum: `taster`, `waitlist` (determines path classification)

**Sources:**[src/pages/admin/enquiries.astro12](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/enquiries.astro#L12-L12)[src/lib/db/schema.js](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/db/schema.js)

---

## Empty State Handling

When no enquiries match the current filters, the page displays a centered empty state message:

```
┌─────────────────────────────────────┐
│                                     │
│       No enquiries found.           │
│                                     │
└─────────────────────────────────────┘

```

This occurs when:

- `enquiries.length === 0` after slicing to `PAGE_SIZE`
- Search term produces no matches
- Status filter has no matching records

**Sources:**[src/pages/admin/enquiries.astro172-177](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/enquiries.astro#L172-L177)

---

## Performance Considerations

The page employs several optimization strategies:

### Parallel Data Loading

Age groups are loaded concurrently with enquiries using `Promise.all()`:

```
const [rows, ageGroups] = await Promise.all([
  listEnquiries(filters, env),
  listAgeGroups(env)
]);
```

### Nested Parallel Fetching

For each enquiry, invite and booking data are fetched in parallel:

```
await Promise.all(
  enquiries.map(async (e) => {
    const tasks = [];
    if (e.invite_id) {
      tasks.push(getInviteById(e.invite_id, env));
    }
    tasks.push(listBookingsByEnquiry(e.id, env));
    await Promise.all(tasks);
  })
);
```

### Pagination Limit

Only 20 records are displayed per page, with one extra row fetched to determine page existence. This prevents expensive full table scans.

**Sources:**[src/pages/admin/enquiries.astro43-83](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/enquiries.astro#L43-L83)
