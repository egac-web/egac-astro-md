---
title: "Admin interface"
description: "The admin interface provides authenticated club administrators with tools to manage the club."
pubDate: 2026-03-18
---
# Admin Interface
Relevant source files
- [src/layouts/AdminBase.astro](https://github.com/egac-web/egac/blob/0ba542fc/src/layouts/AdminBase.astro)
- [src/pages/admin/bookings.astro](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/bookings.astro)
- [src/pages/admin/reports.astro](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/reports.astro)
- [tests/unit/admin.test.ts](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/admin.test.ts)
- [tests/unit/cron.test.ts](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/cron.test.ts)

## Purpose and Scope

The admin interface provides authenticated club administrators with tools to manage enquiries, bookings, academy waitlists, and system configuration. This document covers the overall architecture, authentication mechanism, layout structure, and API patterns used throughout the admin interface.

For detailed information about specific features:

- Layout and navigation structure: [Admin Layout & Navigation](/egac-web/egac/4.1-admin-layout-and-navigation)
- Enquiry management workflows: [Enquiries Management](/egac-web/egac/4.2-enquiries-management)
- Booking calendar and attendance tracking: [Bookings Management](/egac-web/egac/4.3-bookings-management)
- Metrics and reporting: [Reports Dashboard](/egac-web/egac/4.4-reports-dashboard)
- Configuration management: [System Settings](/egac-web/egac/4.5-system-settings)
- Academy season management: [Academy Management](/egac-web/egac/4.6-academy-management)

---

## Authentication System

All admin pages and API endpoints require authentication via the `checkAdminToken` function. The system supports two authentication modes:

### Token-Based Authentication

Admin requests must include a token via one of two methods:

1. **Bearer Token** (HTTP Authorization header):

```
Authorization: Bearer <token>

```
2. **Custom Header**:

```
x-admin-token: <token>

```

The token is validated against `env.ADMIN_TOKEN` configured in environment variables. Authentication logic resides in [src/lib/auth/admin.ts](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/auth/admin.ts)

**Sources:**[tests/unit/admin.test.ts167-214](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/admin.test.ts#L167-L214)[src/pages/admin/bookings.astro14-17](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/bookings.astro#L14-L17)[src/pages/admin/reports.astro13-16](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/reports.astro#L13-L16)

### Development Mode Shortcut

In non-production environments (`APP_ENV !== 'production'`), the special token `"dev"` is accepted as valid. This simplifies local development by eliminating the need to configure production credentials.

```
// Development token accepted in dev/staging
if (env.APP_ENV !== 'production' && token === 'dev') {
  return true;
}
```

Production environments reject the `"dev"` token and require the real `ADMIN_TOKEN`.

**Sources:**[tests/unit/admin.test.ts183-198](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/admin.test.ts#L183-L198)

### Cookie-Based Session

Admin pages retrieve the token from the `egac_admin_token` cookie and redirect to `/admin/login` if authentication fails:

```
const token = Astro.cookies.get('egac_admin_token')?.value ?? '';
if (!checkAdminToken(token, env)) return Astro.redirect('/admin/login');
```

**Sources:**[src/pages/admin/bookings.astro16-17](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/bookings.astro#L16-L17)[src/pages/admin/reports.astro15-16](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/reports.astro#L15-L16)

### Authentication Flow

```
Env
Auth
AdminPage
Env["Environment Variables"]
Auth["checkAdminToken()"]
AdminPage["Admin Page (.astro)"]
Browser
Env
Auth
AdminPage
Env["Environment Variables"]
Auth["checkAdminToken()"]
AdminPage["Admin Page (.astro)"]
Browser
alt
["Valid token"]
["Invalid token"]
"API endpoints follow same pattern
but return 401 JSON response"
"GET /admin/bookings"
"Read egac_admin_token cookie"
"checkAdminToken(token, env)"
"Compare with ADMIN_TOKEN"
"true"
"Render admin page"
"false"
"Redirect to /admin/login"
```

**Sources:**[tests/unit/admin.test.ts167-214](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/admin.test.ts#L167-L214)[src/pages/admin/bookings.astro14-17](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/bookings.astro#L14-L17)

---

## Base Layout Architecture

All admin pages inherit from the `AdminBase.astro` layout component, which provides consistent navigation, branding, and styling across the admin interface.

### Layout Structure

```
Renders

AdminBase.astro
(Layout Component)

Sidebar Navigation

Main Content Area
(slot)

Navigation Items Array

EGAC Brand Header

Environment Indicator
& Logout Link

Dashboard
/admin/reports

Enquiries
/admin/enquiries

Bookings
/admin/bookings

Academy
/admin/academy

Templates
/admin/templates

Settings
/admin/settings

Child Page Content
(bookings.astro, reports.astro, etc)
```

**Sources:**[src/layouts/AdminBase.astro1-78](https://github.com/egac-web/egac/blob/0ba542fc/src/layouts/AdminBase.astro#L1-L78)

### Layout Props

The layout accepts two properties:
PropertyTypeRequiredDescription`title``string`YesPage title displayed in browser tab`activeNav``'dashboard' | 'enquiries' | 'bookings' | 'academy' | 'templates' | 'settings'`NoHighlights the active navigation item
**Sources:**[src/layouts/AdminBase.astro5-8](https://github.com/egac-web/egac/blob/0ba542fc/src/layouts/AdminBase.astro#L5-L8)

### Navigation Configuration

The sidebar navigation is defined as an array of navigation items, each mapping a nav ID to a label and route:

```
const navItems: Array<{ id: NonNullable<Props['activeNav']>; label: string; href: string }> = [
  { id: 'dashboard', label: 'Dashboard', href: '/admin/reports' },
  { id: 'enquiries', label: 'Enquiries', href: '/admin/enquiries' },
  { id: 'bookings', label: 'Bookings', href: '/admin/bookings' },
  { id: 'academy', label: 'Academy', href: '/admin/academy' },
  { id: 'templates', label: 'Templates', href: '/admin/templates' },
  { id: 'settings', label: 'Settings', href: '/admin/settings' },
];
```

**Sources:**[src/layouts/AdminBase.astro14-21](https://github.com/egac-web/egac/blob/0ba542fc/src/layouts/AdminBase.astro#L14-L21)

### Visual Styling

The layout uses a dark blue sidebar (`bg-[#214c8f]`) with white text and a light gray main content area (`bg-[#eff2f7]`). Active navigation items are highlighted with a slightly lighter blue background (`bg-[#2d5ea5]`).

**Sources:**[src/layouts/AdminBase.astro32-61](https://github.com/egac-web/egac/blob/0ba542fc/src/layouts/AdminBase.astro#L32-L61)

---

## Admin Sections Overview

The admin interface is divided into six primary sections accessible via the sidebar navigation:

### Dashboard (Reports)

The reports page displays key performance indicators (KPIs), monthly enquiry trends, booking funnel analysis, academy pipeline status, and upcoming session capacity. Implemented in [src/pages/admin/reports.astro](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/reports.astro)

**Key Metrics:**

- Total enquiries (all-time)
- Enquiries this month (with month-over-month trend)
- Attendance rate (this month)
- Academy waitlist size

**Visualizations:**

- 6-month enquiry trend (bar chart)
- Booking funnel (Enquiries → Invites → Bookings → Attended)
- Academy pipeline (accepted/declined/pending)
- Upcoming sessions capacity grid

Detailed documentation: [Reports Dashboard](/egac-web/egac/4.4-reports-dashboard)

**Sources:**[src/pages/admin/reports.astro1-338](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/reports.astro#L1-L338)

### Enquiries

List and search enquiries with filters for status, age group, and date range. View detailed enquiry records including event history, invite status, and associated bookings. Update enquiry notes and manually resend booking invites.

Detailed documentation: [Enquiries Management](/egac-web/egac/4.2-enquiries-management)

**Sources:**[tests/unit/admin.test.ts217-458](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/admin.test.ts#L217-L458)

### Bookings

Calendar view of upcoming taster sessions grouped by date and age group. Track attendance in real-time with inline status updates. Send membership invites after marking athletes as attended.

The bookings page displays the next 4 Tuesdays with capacity indicators and attendance controls.

Detailed documentation: [Bookings Management](/egac-web/egac/4.3-bookings-management)

**Sources:**[src/pages/admin/bookings.astro1-253](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/bookings.astro#L1-L253)

### Academy

Manage academy seasons, waitlist entries, and season rollover. View waitlist positions, send invites, and track acceptance/decline responses.

Detailed documentation: [Academy Management](/egac-web/egac/4.6-academy-management)

### Templates

Preview and edit email templates used throughout the system. Validate template variables and test rendering with sample data.

### Settings

Configure system-wide settings including age groups, session times, capacity limits, venue information, and email addresses.

Detailed documentation: [System Settings](/egac-web/egac/4.5-system-settings)

---

## API Architecture

Admin API endpoints follow a consistent pattern with authentication, standardized request/response formats, and repository-based data access.

### API Endpoint Structure

```
401 if invalid

Authenticated

422 if invalid

Valid

HTTP Request

checkAdminToken()

API Handler Function

Request Validation

Repository Layer

D1 Database

JSON Response
```

**Sources:**[tests/unit/admin.test.ts1-583](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/admin.test.ts#L1-L583)

### Admin API Endpoints
EndpointMethodPurposeHandler`/api/admin/enquiries`GETList enquiries with pagination`handleAdminEnquiries``/api/admin/enquiries`POSTCreate manual enquiry`handleAdminEnquiries``/api/admin/enquiries/:id`GETGet enquiry detail with events`handleAdminEnquiryDetail``/api/admin/enquiries/:id`PATCHUpdate enquiry note`handleAdminEnquiryDetail``/api/admin/invites/resend`POSTResend booking invite`handleAdminInviteResend``/api/admin/bookings/:id/attendance`POSTMark booking attendanceNot shown in files
**Sources:**[tests/unit/admin.test.ts52-56](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/admin.test.ts#L52-L56)

### Request Authentication

API endpoints extract the token from either the `Authorization` header or `x-admin-token` header:

```
// Bearer token format
Authorization: Bearer <token>
 
// Custom header format
x-admin-token: <token>
```

If authentication fails, the endpoint returns a 401 Unauthorized response:

```
{
  "ok": false,
  "code": "UNAUTHORIZED",
  "message": "Invalid or missing admin token"
}
```

**Sources:**[tests/unit/admin.test.ts172-214](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/admin.test.ts#L172-L214)

### Response Format

All admin API responses follow a consistent JSON structure:

**Success Response:**

```
{
  "ok": true,
  "data": {
    // Response payload
  }
}
```

**Error Response:**

```
{
  "ok": false,
  "code": "ERROR_CODE",
  "message": "Human-readable error message"
}
```

Common error codes:

- `UNAUTHORIZED` (401): Invalid or missing admin token
- `NOT_FOUND` (404): Resource not found
- `VALIDATION_ERROR` (422): Invalid request data
- `INTERNAL_ERROR` (500): Server error

**Sources:**[tests/unit/admin.test.ts159-161](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/admin.test.ts#L159-L161)[tests/unit/admin.test.ts290-296](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/admin.test.ts#L290-L296)

### Pagination Pattern

List endpoints support pagination via query parameters:

```
GET /api/admin/enquiries?page=2&status=pending

```

Response includes pagination metadata:

```
{
  "ok": true,
  "data": {
    "enquiries": [...],
    "page": 2,
    "hasNextPage": true
  }
}
```

The system uses `PAGE_SIZE = 20` and fetches one extra row to determine if more pages exist. If 21 rows are returned, `hasNextPage` is set to `true` and only the first 20 are included in the response.

**Sources:**[tests/unit/admin.test.ts243-259](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/admin.test.ts#L243-L259)

---

## Admin Page Components and Patterns

### Calendar Grid Pattern

The bookings page uses a responsive calendar grid to display upcoming sessions:

```
Calendar Grid Container

Tuesday 1
2025-06-10

Tuesday 2
2025-06-17

Tuesday 3
2025-06-24

Tuesday 4
2025-07-01

U11s 18:00
2/2 booked

U13s 18:00
1/2 booked

U15s 19:00
0/2 booked

Alice Smith
Confirmed

Bob Jones
Confirmed

Mark Attended

No Show
```

Each day column contains age group slots, each slot displays:

- Age group label and session time
- Capacity bar (visual indicator)
- Booked count / Total capacity
- List of bookings with attendance controls

**Sources:**[src/pages/admin/bookings.astro34-52](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/bookings.astro#L34-L52)[src/pages/admin/bookings.astro74-176](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/bookings.astro#L74-L176)

### Attendance Update Flow

Attendance marking uses client-side JavaScript to update booking status via AJAX without page reload:

```

```

**Sources:**[src/pages/admin/bookings.astro179-252](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/bookings.astro#L179-L252)

### Metric Card Pattern

The reports dashboard uses a consistent card layout for displaying KPIs:

```
<div class="rounded-xl border border-slate-200 bg-white p-5 shadow-sm">
  <p class="text-xs font-medium uppercase tracking-wide text-slate-500">
    Metric Label
  </p>
  <p class="mt-2 text-3xl font-semibold text-slate-900">
    Value
  </p>
  <p class="mt-1 text-xs text-slate-400">
    Additional context
  </p>
</div>
```

This pattern is used for:

- Total enquiries
- Monthly enquiries (with trend indicator)
- Attendance rate
- Academy waitlist size

**Sources:**[src/pages/admin/reports.astro152-189](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/reports.astro#L152-L189)

### Data Enrichment Pattern

Admin pages often enrich data from multiple sources before rendering:

```
// Load bookings
const upcomingBookings: Booking[] = await listUpcomingBookings(4, env);
 
// Enrich with enquiry names
type EnrichedBooking = Booking & { enquiry_name: string | null; enquiry_email: string };
const enriched: EnrichedBooking[] = await Promise.all(
  upcomingBookings.map(async (b) => {
    const enquiry = await getEnquiryById(b.enquiry_id, env);
    return { ...b, enquiry_name: enquiry?.name ?? null, enquiry_email: enquiry?.email ?? '' };
  })
);
```

This pattern is used to:

- Add enquiry details to bookings
- Attach invite status to enquiries
- Include event history in enquiry details

**Sources:**[src/pages/admin/bookings.astro25-32](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/bookings.astro#L25-L32)

---

## Toast Notification System

Admin pages that perform actions (like marking attendance) use a toast notification system to provide feedback without page reload:

```
function showToast(message: string, success = true) {
  if (!toast) return;
  toast.textContent = message;
  toast.className = `fixed top-4 right-4 z-50 px-4 py-3 rounded-lg shadow-lg text-sm font-medium text-white ${
    success ? 'bg-green-700' : 'bg-red-600'
  }`;
  toast.classList.remove('hidden');
  setTimeout(() => toast.classList.add('hidden'), 3000);
}
```

The toast element is positioned fixed in the top-right corner and automatically dismisses after 3 seconds. Success messages use green background, error messages use red.

**Sources:**[src/pages/admin/bookings.astro180-190](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/bookings.astro#L180-L190)

---

## Security Considerations

### Token Storage

Admin tokens are stored in HTTP-only cookies to prevent XSS attacks. The cookie is set upon successful login and validated on every admin page and API request.

### CSRF Protection

API endpoints that modify data (POST, PATCH, DELETE) require the admin token in the request headers, providing protection against cross-site request forgery.

### Environment Isolation

The `environment` field on all database tables ensures that development, staging, and production data remain isolated. Admin queries filter by `environment = ?` to prevent cross-environment data access.

**Sources:**[src/pages/admin/reports.astro19-20](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/reports.astro#L19-L20)

### Production Token Enforcement

The development shortcut token `"dev"` is explicitly rejected in production environments, ensuring that production systems require properly configured authentication credentials.

**Sources:**[tests/unit/admin.test.ts183-190](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/admin.test.ts#L183-L190)
