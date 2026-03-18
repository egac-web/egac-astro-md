# Reports Dashboard
Relevant source files
- [docs/snagging.md](https://github.com/egac-web/egac/blob/0ba542fc/docs/snagging.md)
- [src/lib/repositories/age-groups.ts](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/repositories/age-groups.ts)
- [src/lib/repositories/bookings.ts](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/repositories/bookings.ts)
- [src/pages/admin/bookings.astro](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/bookings.astro)
- [src/pages/admin/enquiries.astro](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/enquiries.astro)
- [src/pages/admin/reports.astro](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/reports.astro)

The Reports Dashboard provides club administrators with real-time visibility into key performance indicators (KPIs), trends, and operational metrics across the EGAC platform. This dashboard consolidates data from enquiries, bookings, invites, and academy waitlist to present a comprehensive view of system activity.

For managing individual enquiries, see [Enquiries Management](/egac-web/egac/4.2-enquiries-management). For calendar-based booking management, see [Bookings Management](/egac-web/egac/4.3-bookings-management). For academy-specific operations, see [Academy Management](/egac-web/egac/4.6-academy-management).

---

## Overview

The Reports Dashboard is implemented as a server-rendered Astro page at [src/pages/admin/reports.astro1-338](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/reports.astro#L1-L338) It performs all data aggregation on the server and renders static visualizations using pure CSS and inline styles. The page displays six primary components:
ComponentPurposeData Source**Metric Cards**High-level KPIs (4 cards)Direct SQL queries + repository functions**Monthly Trend**Last 6 months of enquiry volume`countEnquiries` with date ranges**Booking Funnel**Conversion rates through stages`enquiries`, `invites`, `bookings` tables**Academy Pipeline**Waitlist response breakdown`academy_waitlist` table queries**Upcoming Sessions**Next 4 weeks capacity status`listUpcomingBookings` + `listAgeGroups`
All visualizations use horizontal bar charts rendered with CSS `width` percentages and Tailwind utility classes. No client-side JavaScript is required for chart rendering.

**Sources:**[src/pages/admin/reports.astro1-338](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/reports.astro#L1-L338)

---

## Page Structure and Data Flow

```
UI Rendering

Data Structures

Business Logic

Data Aggregation Layer

Request Handler

checkAdminToken
[admin.js]

reports.astro
frontmatter execution

countEnquiries()
local helper function

bookings.ts
repository functions

age-groups.ts
listAgeGroups()

Direct D1 queries
academy_waitlist table

age.js
getNextNTuesdayDates
formatDateForDisplay

Scalar variables
enquiriesTotal
attendanceRate
etc

monthlyTrend[]
6 month history

funnelSteps[]
4 stage pipeline

upcomingSessions[]
4 week forecast

Metric Cards
4 stat boxes

Monthly Trend
horizontal bars

Booking Funnel
% conversion bars

Academy Pipeline
stacked bar

Upcoming Sessions
capacity bars
```

**Sources:**[src/pages/admin/reports.astro13-145](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/reports.astro#L13-L145)

---

## Authentication and Environment Setup

The dashboard requires admin authentication via the `egac_admin_token` cookie. If authentication fails, the request redirects to `/admin/login`.

```
// Authentication check
const env = Astro.locals.runtime.env;
const token = Astro.cookies.get('egac_admin_token')?.value ?? '';
if (!checkAdminToken(token, env)) return Astro.redirect('/admin/login');
```

The page establishes database and environment context using the `Env` binding from Cloudflare runtime:

- `getDB(env)` returns a D1 database instance
- `getEnv(env)` returns the environment string (`development`, `staging`, or `production`)

All queries include `environment = ?` conditions to ensure data isolation between environments.

**Sources:**[src/pages/admin/reports.astro13-21](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/reports.astro#L13-L21)

---

## Metric Cards

Four metric cards display top-level KPIs at the page header:

### Card 1: Total Enquiries

Displays the lifetime count of enquiries in the current environment.

**Data Source:**`countEnquiries()` with no date filters**Implementation:**[src/pages/admin/reports.astro29-59](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/reports.astro#L29-L59)

### Card 2: This Month Enquiries

Shows current month enquiry count with month-over-month (MoM) comparison:

```
const thisMonthStart = new Date(now.getFullYear(), now.getMonth(), 1)
  .toISOString().split('T')[0];
 
// Calculate percentage change
const trend = enquiriesLastMonth > 0
  ? Math.round(((enquiriesThisMonth - enquiriesLastMonth) / enquiriesLastMonth) * 100)
  : null;
```

Displays a green "↑" for positive growth or red "↓" for decline.

**Sources:**[src/pages/admin/reports.astro23-170](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/reports.astro#L23-L170)

### Card 3: Attendance Rate

Calculates the percentage of athletes who attended their taster session (this month only):

```
const totalDecided = attendedThisMonth + noShowThisMonth;
const attendanceRate = totalDecided > 0 
  ? Math.round((attendedThisMonth / totalDecided) * 100) 
  : null;
```

- Numerator: `COUNT(*) FROM bookings WHERE status = 'attended' AND session_date >= thisMonthStart`
- Denominator: `attendedThisMonth + noShowThisMonth`
- Displays "—" if no sessions have been decided yet

**Sources:**[src/pages/admin/reports.astro48-179](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/reports.astro#L48-L179)

### Card 4: Academy Waitlist

Shows total academy waitlist entries with a breakdown of responses:

```
-- Total entries
SELECT COUNT(*) FROM academy_waitlist WHERE environment = ?
 
-- Yes responses
SELECT COUNT(*) FROM academy_waitlist WHERE response = 'yes' AND environment = ?
 
-- No responses
SELECT COUNT(*) FROM academy_waitlist WHERE response = 'no' AND environment = ?
 
-- Pending = Total - Yes - No
```

Displays as "42 total" with "18 yes · 15 pending" subtext.

**Sources:**[src/pages/admin/reports.astro72-189](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/reports.astro#L72-L189)

---

## Monthly Enquiry Trend

Displays a 6-month horizontal bar chart showing enquiry volume over time:

```
Visualization

Data Collection Loop

for i = 5 to 0
(6 iterations)

Calculate month start/end dates
using Date arithmetic

countEnquiries(start, end)
for each month

monthlyTrend.push({
label: 'Jan 2024',
count: 42
})

maxMonthly = max(counts)
for percentage scale

Each month renders:
- Label (right-aligned)
- Blue bar (width % of max)
- Count value
```

The chart iterates backward through time (`i = 5` down to `i = 0`) to build chronological data:

```
for (let i = 5; i >= 0; i--) {
  const d = new Date(now.getFullYear(), now.getMonth() - i, 1);
  const start = d.toISOString().split('T')[0];
  const end = new Date(d.getFullYear(), d.getMonth() + 1, 0).toISOString().split('T')[0];
  const count = await countEnquiries(start, end);
  monthlyTrend.push({
    label: d.toLocaleDateString('en-GB', { month: 'short', year: 'numeric' }),
    count,
  });
}
```

Each bar's width is calculated as `(count / maxMonthly) * 100%`.

**Sources:**[src/pages/admin/reports.astro91-211](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/reports.astro#L91-L211)

---

## Booking Funnel

The booking funnel visualizes conversion rates through four stages:
StageDescriptionData Source**Enquiries**Total lifetime enquiries`COUNT(*) FROM enquiries`**Invites sent**Invites with `status IN ('sent', 'accepted')``COUNT(*) FROM invites`**Booked**Bookings with `status = 'confirmed'``countBookingsByStatus('confirmed')`**Attended**Bookings with `status = 'attended'``countBookingsByStatus('attended')`
```
Visualization Logic

Stage Definitions

Stage 1: Enquiries
funnelSteps[0]
value from countEnquiries()

Stage 2: Invites sent
funnelSteps[1]
SELECT COUNT(*) FROM invites
WHERE status IN ('sent','accepted')

Stage 3: Booked
funnelSteps[2]
countBookingsByStatus('confirmed')

Stage 4: Attended
funnelSteps[3]
countBookingsByStatus('attended')

funnelMax = funnelSteps[0].value
(enquiries as 100% baseline)

For each stage:
pct = (value / funnelMax) * 100

Conversion rate:
(value / enquiriesTotal) * 100

Color gradient:
blue-600 → blue-400 → green-500 → teal-500
```

Each stage displays:

- The stage name and absolute count
- A horizontal bar (width = percentage of total enquiries)
- Conversion percentage in parentheses (except for stage 1)

**Sources:**[src/pages/admin/reports.astro106-242](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/reports.astro#L106-L242)

---

## Academy Pipeline Visualization

The academy pipeline shows waitlist response distribution as a stacked horizontal bar:

```
// Query responses
const academyYes = await db.prepare(
  `SELECT COUNT(*) as n FROM academy_waitlist WHERE response = 'yes' AND environment = ?`
).bind(environment).first<{ n: number }>();
 
const academyNo = await db.prepare(
  `SELECT COUNT(*) as n FROM academy_waitlist WHERE response = 'no' AND environment = ?`
).bind(environment).first<{ n: number }>();
 
const academyPending = (academyTotal?.n ?? 0) - (academyYes?.n ?? 0) - (academyNo?.n ?? 0);
```

The stacked bar renders three segments:

- **Green segment**: "Yes" responses (width = `(yesCount / total) * 100%`)
- **Red segment**: "No" responses (width = `(noCount / total) * 100%`)
- **Gray segment**: Pending responses (width = `(pendingCount / total) * 100%`)

Each segment displays its count as white text if the segment is wide enough.

**Sources:**[src/pages/admin/reports.astro72-295](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/reports.astro#L72-L295)

---

## Upcoming Sessions Capacity View

The upcoming sessions component forecasts booking capacity for the next 4 Tuesdays:

```
Rendering

Structure Assembly

Data Preparation

getNextNTuesdayDates(4)
[age.js:227-241]
Returns ['2024-01-09', '2024-01-16', ...]

listAgeGroups(env)
Filter to active taster groups

listUpcomingBookings(4, env)
All confirmed/attended/no_show
for next 4 weeks

For each date:
- format: formatDateForDisplay(date)
- slots: []

For each age group:
- label: ag.label
- capacity: ag.capacity_per_session
- booked: count matching bookings

upcomingSessions[] = {
date, display, slots[]
}

Display formatted date

For each slot:
- Show label + booked/capacity
- Render progress bar
- Red if >= 100%, blue otherwise
```

The structure created is:

```
type UpcomingSession = {
  date: string;                    // YYYY-MM-DD
  display: string;                 // e.g. "Tuesday 9 Jan"
  slots: Array<{
    label: string;                 // Age group label
    booked: number;                // Count of bookings
    capacity: number;              // From age_groups.capacity_per_session
  }>;
};
```

Each slot renders a capacity bar where:

- Width = `Math.min(100, (booked / capacity) * 100)%`
- Color = `red-500` if at/over capacity, otherwise `blue-500`

**Sources:**[src/pages/admin/reports.astro125-335](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/reports.astro#L125-L335)

---

## Data Query Patterns

### The `countEnquiries` Helper

The page defines a local helper function for flexible enquiry counting:

```
async function countEnquiries(since?: string, until?: string): Promise<number> {
  let sql = 'SELECT COUNT(*) as n FROM enquiries WHERE environment = ?';
  const params: unknown[] = [environment];
  
  if (since) {
    sql += ' AND created_at >= ?';
    params.push(since);
  }
  if (until) {
    sql += ' AND created_at <= ?';
    params.push(until + 'T23:59:59Z');  // End of day
  }
  
  const row = await db.prepare(sql).bind(...params).first<{ n: number }>();
  return row?.n ?? 0;
}
```

This function is called multiple times with different date ranges:

- `countEnquiries()` → all time total
- `countEnquiries(thisMonthStart)` → this month only
- `countEnquiries(lastMonthStart, lastMonthEnd)` → last month only
- `countEnquiries(monthStart, monthEnd)` → specific month (in loop)

**Sources:**[src/pages/admin/reports.astro29-45](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/reports.astro#L29-L45)

### Repository Function Usage

The page imports and uses several repository functions:
FunctionSourcePurpose`countBookingsByStatus`[src/lib/repositories/bookings.ts145-164](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/repositories/bookings.ts#L145-L164)Count bookings by status, optionally since a date`listUpcomingBookings`[src/lib/repositories/bookings.ts104-123](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/repositories/bookings.ts#L104-L123)Get all bookings for next N weeks`listAgeGroups`[src/lib/repositories/age-groups.ts9-15](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/repositories/age-groups.ts#L9-L15)Fetch all age groups (filters to active)
**Sources:**[src/pages/admin/reports.astro8-89](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/reports.astro#L8-L89)

---

## Layout and Responsive Design

The dashboard uses a responsive grid layout:

```
<!-- Top-level container -->
<div class="space-y-6">
  
  <!-- 4 metric cards -->
  <div class="grid grid-cols-2 gap-4 lg:grid-cols-4">
    <!-- Cards render here -->
  </div>
  
  <!-- Visualization grid -->
  <div class="grid grid-cols-1 gap-6 lg:grid-cols-2">
    <!-- Monthly trend (left) -->
    <!-- Booking funnel (right) -->
    <!-- Academy pipeline (left) -->
    <!-- Upcoming sessions (right) -->
  </div>
  
</div>
```

**Breakpoints:**

- Mobile: Metric cards 2-column grid, visualizations stack vertically
- Large screens (`lg:`): Metric cards 4-column grid, visualizations 2-column grid

All cards use the same base styling:

- `rounded-xl border border-slate-200 bg-white p-5 shadow-sm`

**Sources:**[src/pages/admin/reports.astro148-337](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/reports.astro#L148-L337)

---

## Performance Characteristics

The Reports Dashboard executes 15+ database queries during server-side rendering:

**Query Breakdown:**

- 1 authentication check
- 7 enquiry count queries (total, this month, last month, 6 months historical)
- 4 booking status counts (confirmed, attended, attended this month, no-show this month)
- 3 academy waitlist counts (total, yes, no)
- 1 invite count query
- 1 age groups query
- 1 upcoming bookings query

All queries execute in parallel where possible using `Promise.all()`. The page performs no client-side data fetching—all metrics are computed at request time.

**Cache Considerations:**

- No built-in caching is implemented
- Each page load re-queries all data
- Admin users see real-time metrics on every refresh

For high-traffic deployments, consider adding caching at the `getSystemConfig` level or implementing stale-while-revalidate patterns.

**Sources:**[src/pages/admin/reports.astro18-138](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/reports.astro#L18-L138)

---

## Styling and Visual Design

All visualizations use pure CSS for rendering—no charting libraries are included. The visual design follows these patterns:

### Horizontal Bar Charts

```
<div class="flex items-center gap-3">
  <span class="w-20 text-right text-xs text-gray-500">Label</span>
  <div class="flex-1 bg-gray-100 rounded-full h-4">
    <div class="h-4 rounded-full bg-blue-600" style="width: 75%"></div>
  </div>
  <span class="w-6 text-xs text-gray-700">42</span>
</div>
```

### Stacked Bars

```
<div class="flex rounded-full overflow-hidden h-6">
  <div class="bg-green-500" style="width: 60%">18</div>
  <div class="bg-red-400" style="width: 20%">6</div>
  <div class="bg-gray-300" style="width: 20%">6</div>
</div>
```

### Status Badges

Metric cards and labels use Tailwind color scales:

- Blue: primary metrics, confirmed bookings
- Green: positive trends, attendance, "yes" responses
- Red: negative trends, "no" responses, over-capacity
- Gray: neutral/pending states

**Sources:**[src/pages/admin/reports.astro152-335](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/reports.astro#L152-L335)

---

## Integration with Other Admin Pages

The Reports Dashboard serves as the landing page for the admin interface (activated by `activeNav="dashboard"`). It provides high-level visibility that links to detail pages:

```
View enquiry details

Manage session capacity

Academy pipeline drill-down

Reports Dashboard
/admin/reports

Enquiries Management
/admin/enquiries

Bookings Management
/admin/bookings

Academy Management
/admin/academy
```

Administrators typically use the dashboard to:

1. Monitor overall system health (enquiry volume, attendance rates)
2. Identify capacity constraints (upcoming sessions near/at capacity)
3. Track conversion effectiveness (booking funnel drop-offs)
4. Manage academy waitlist responses

**Sources:**[src/pages/admin/reports.astro147](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/reports.astro#L147-L147)[src/layouts/AdminBase.astro](https://github.com/egac-web/egac/blob/0ba542fc/src/layouts/AdminBase.astro)