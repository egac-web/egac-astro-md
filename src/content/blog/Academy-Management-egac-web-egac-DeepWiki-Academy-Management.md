---
title: "Academy Management"
description: "This page documents the Academy Management administrative interface"
pubDate: 2026-03-18
---
# Academy Management
Relevant source files
- [src/lib/business/age.ts](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/business/age.ts)
- [src/lib/business/enquiry.ts](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/business/enquiry.ts)
- [src/lib/business/templates.ts](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/business/templates.ts)
- [src/lib/business/tokens.ts](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/business/tokens.ts)
- [src/pages/admin/academy.astro](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/academy.astro)
- [src/pages/admin/settings.astro](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/settings.astro)
- [tests/unit/admin.test.ts](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/admin.test.ts)
- [tests/unit/business.test.ts](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/business.test.ts)
- [tests/unit/cron.test.ts](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/cron.test.ts)

## Purpose and Scope

This page documents the **Academy Management** administrative interface, which allows club administrators to manage academy seasons, waitlist allocations, and athlete progression for the younger age groups. The academy system handles athletes below the `academy_max_age` threshold (typically age 10 and under) who are routed to a structured waitlist rather than immediate taster session booking.

For information on how enquiries are initially routed to the academy waitlist, see [Enquiry Routing](/egac-web/egac/6.2-enquiry-routing). For the public-facing academy waitlist acceptance/decline flow, see [Academy Waitlist](/egac-web/egac/3.3-academy-waitlist). For the automated season rollover job, see [Academy Rollover Job](/egac-web/egac/9.4-academy-rollover-job).

---

## Academy Waitlist Concept

The academy waitlist system supports multi-season capacity planning for younger athletes. Instead of immediate booking, athletes are:

1. Added to the academy waitlist when they submit an enquiry with DOB indicating age ≤ `academy_max_age`
2. Assigned to a specific **academy season** (or left unassigned if no season is open for enrollment)
3. Moved through status transitions: `waiting` → `invited` → `accepted` or `declined`
4. Automatically rolled over to the next season if they remain in `waiting` status when a season closes

Each season has:

- Start and end dates
- Capacity limit (total accepted athletes)
- Status: `upcoming`, `open`, `closed`, or `archived`

**Sources:**[src/pages/admin/academy.astro1-298](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/academy.astro#L1-L298)

---

## Data Model

### Academy Seasons

The `academy_seasons` table defines capacity-managed periods for academy enrollment:
FieldTypeDescription`id`string (PK)Season identifier`code`stringMachine-readable code (e.g. `academy-2026`)`label`stringDisplay label (e.g. "Academy 2026")`age_group_id`string (FK)Links to `age_groups` table`start_date`dateSeason start date`end_date`dateSeason end date`capacity`integerMaximum accepted athletes`status`enum`upcoming`, `open`, `closed`, `archived`
### Academy Waitlist Entries

The `academy_waitlist` table tracks individual athlete enrollment attempts:
FieldTypeDescription`id`string (PK)Waitlist entry identifier`enquiry_id`string (FK)References `enquiries.id``season_id`string (FK, nullable)References `academy_seasons.id` (null = unassigned)`token`stringSecure response token for accept/decline URL`position`integer (nullable)Queue position (currently unused)`status`enum`waiting`, `invited`, `accepted`, `declined`, `ineligible`, `withdrawn``is_returning`integer (boolean)1 if athlete previously accepted in any season`rolled_over_from_season_id`string (nullable)Source season ID if created by rollover`invited_at`datetime (nullable)Timestamp of invitation email`responded_at`datetime (nullable)Timestamp of athlete response`response`string (nullable)`accept` or `decline`
**Sources:**[src/lib/db/schema.ts1-400](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/db/schema.ts#L1-L400) (inferred from usage)

---

## Relationship Diagram

```
generates

defines capacity for

enrolls

rolled_over_from

ENQUIRIES

string

id

PK

string

email

string

name

string

dob

string

age_group_id

FK

ACADEMY_WAITLIST

string

id

PK

string

enquiry_id

FK

string

season_id

FK

string

token

enum

status

waiting invited accepted declined ineligible withdrawn

integer

is_returning

string

rolled_over_from_season_id

FK

AGE_GROUPS

string

id

PK

string

code

enum

booking_type

taster or waitlist

ACADEMY_SEASONS

string

id

PK

string

code

string

label

string

age_group_id

FK

date

start_date

date

end_date

integer

capacity

enum

status

upcoming open closed archived
```

**Sources:**[src/pages/admin/academy.astro11-12](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/academy.astro#L11-L12)[tests/unit/cron.test.ts456-499](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/cron.test.ts#L456-L499)

---

## Admin Interface Layout

The academy management page is located at `/admin/academy` and rendered by `src/pages/admin/academy.astro`. It displays:

### Summary Cards

Four metric cards at the top:

- **Seasons**: Total count of all seasons
- **Open seasons**: Count of seasons with `status = 'open'`
- **Unassigned waitlist**: Count of entries with `season_id = null`
- **Accepted total**: Sum of entries with `status = 'accepted'` across all seasons

**Code:**[src/pages/admin/academy.astro136-153](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/academy.astro#L136-L153)

### Unassigned Waitlist Section

Lists waitlist entries that have not been allocated to a season (`season_id = null`). Each row displays:

- Athlete name (from `enquiries.name`)
- Email (from `enquiries.email`)
- Created date
- Current status
- Status dropdown + Save button for quick updates

**Code:**[src/pages/admin/academy.astro171-228](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/academy.astro#L171-L228)

### Season Allocation Blocks

Grid layout showing each season as a card with:

- Season label, date range, and capacity fill rate
- Progress bar showing `acceptedCount / capacity`
- Status dropdown (upcoming → open → closed → archived)
- First 8 waitlist entries for that season
- Truncation message if more than 8 entries exist

**Code:**[src/pages/admin/academy.astro230-295](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/academy.astro#L230-L295)

**Sources:**[src/pages/admin/academy.astro1-298](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/academy.astro#L1-L298)

---

## Season Status Workflow

### Status Transitions

```
"Season created"

"Admin opens enrollment"

"Season expires (automated) or manually closed"

"Admin archives old season"

upcoming

open

closed

archived

"Waitlist entries can be invited/accepted"

"Rollover cron migrates waiting entries"
```

### Season Status Values
StatusDescriptionActions Available`upcoming`Season not yet open for enrollmentCan be changed to `open``open`Actively enrolling athletesCan send invites, accept athletes, or close manually`closed`Enrollment complete, season endedNo further modifications; can archive`archived`Historical recordRead-only
### Automated Status Changes

The academy rollover cron job ([handleAcademyRollover](/egac-web/egac/9.4-academy-rollover-job)) runs monthly and:

1. Identifies all `open` seasons where `end_date < today`
2. Migrates `waiting` and `invited` entries to the next `upcoming` season
3. Updates season status from `open` to `closed`

**Code:**[tests/unit/cron.test.ts500-606](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/cron.test.ts#L500-L606)

### Manual Status Updates

Administrators can change season status via the inline dropdown on each season card:

```
POST /admin/academy
  action=season-status
  season_id=acs_2026
  status=open

```

**Handler:**[src/pages/admin/academy.astro26-35](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/academy.astro#L26-L35)

**Sources:**[src/pages/admin/academy.astro21-57](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/academy.astro#L21-L57)[tests/unit/cron.test.ts500-606](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/cron.test.ts#L500-L606)

---

## Waitlist Status Workflow

### Status Transitions

```
"Enquiry submitted with age ≤ academy_max_age"

"Admin manually invites athlete"

"Admin marks ineligible (age/location)"

"Parent withdraws interest"

"Parent accepts via email link"

"Parent declines via email link"

"Parent withdraws after invite"

"Athlete enrolled"

"Final status"

"Final status"

"Final status"

waiting

invited

ineligible

withdrawn

accepted

declined

"Can be rolled over to next season"

"Can be rolled over if season expires"
```

### Waitlist Status Values
StatusDescriptionRollover Behavior`waiting`Athlete on waitlist, not yet invited**Rolled over** to next season`invited`Invitation sent, awaiting response**Rolled over** to next season`accepted`Parent accepted, athlete enrolled**Not rolled over** (finalized)`declined`Parent declined enrollment**Not rolled over** (finalized)`ineligible`Admin marked athlete as ineligible**Not rolled over** (finalized)`withdrawn`Parent withdrew interest**Not rolled over** (finalized)
### Manual Status Updates

Each waitlist entry row includes a dropdown to change status:

```
POST /admin/academy
  action=waitlist-status
  waitlist_id=awl_123
  status=accepted

```

**Handler:**[src/pages/admin/academy.astro37-53](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/academy.astro#L37-L53)

**Sources:**[src/pages/admin/academy.astro21-57](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/academy.astro#L21-L57)[tests/unit/cron.test.ts584-605](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/cron.test.ts#L584-L605)

---

## Repository Functions

### Listing Operations

#### `listAcademySeasons(env: Env): Promise<AcademySeason[]>`

Returns all academy seasons ordered by `start_date DESC`. Used to populate the season cards.

**Implementation:**[src/lib/repositories/academy.js1-50](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/repositories/academy.js#L1-L50) (inferred)

**Usage:**[src/pages/admin/academy.astro59](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/academy.astro#L59-L59)

#### `listWaitlistBySeason(seasonId: string, env: Env): Promise<AcademyWaitlistEntry[]>`

Returns all waitlist entries for a specific season, ordered by `created_at ASC`.

**Usage:**[src/pages/admin/academy.astro64](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/academy.astro#L64-L64)

#### `listUnassignedWaitlist(env: Env): Promise<AcademyWaitlistEntry[]>`

Returns waitlist entries where `season_id IS NULL`, ordered by `created_at ASC`.

**Usage:**[src/pages/admin/academy.astro60](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/academy.astro#L60-L60)

### Update Operations

#### `updateSeasonStatus(seasonId: string, status: AcademySeason['status'], env: Env): Promise<void>`

Updates the status of a season. Used by both admin UI and rollover cron.

**Usage:**[src/pages/admin/academy.astro33](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/academy.astro#L33-L33)[tests/unit/cron.test.ts559-563](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/cron.test.ts#L559-L563)

#### `updateWaitlistStatus(waitlistId: string, status: AcademyWaitlistStatus, env: Env): Promise<void>`

Updates the status of a waitlist entry.

**Usage:**[src/pages/admin/academy.astro51](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/academy.astro#L51-L51)

### Rollover Operations

#### `rolloverWaitlistEntry(waitlistId: string, newSeasonId: string | null, fromSeasonId: string, isReturning: boolean, env: Env): Promise<void>`

Creates a new waitlist entry for the next season, copying data from the original entry but resetting status to `waiting`. Sets `rolled_over_from_season_id` and `is_returning` flags.

**Usage:**[tests/unit/cron.test.ts552-558](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/cron.test.ts#L552-L558)[tests/unit/cron.test.ts575-581](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/cron.test.ts#L575-L581)

#### `hasAcceptedEntryForEnquiry(enquiryId: string, env: Env): Promise<boolean>`

Checks if an enquiry has any `accepted` waitlist entry in any previous season. Used to set the `is_returning` flag during rollover.

**Usage:**[tests/unit/cron.test.ts540](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/cron.test.ts#L540-L540)[tests/unit/cron.test.ts570](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/cron.test.ts#L570-L570)

#### `getUpcomingAcademySeason(ageGroupId: string, env: Env): Promise<AcademySeason | null>`

Returns the next `upcoming` or `open` season for a given age group. Used to determine rollover target.

**Usage:**[tests/unit/cron.test.ts538](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/cron.test.ts#L538-L538)

**Sources:**[src/pages/admin/academy.astro4-10](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/academy.astro#L4-L10)[tests/unit/cron.test.ts36-43](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/cron.test.ts#L36-L43)

---

## Automated Season Rollover

### Cron Job: `handleAcademyRollover`

**Trigger:** Monthly on the 1st at 09:00 UTC via cron worker

**Endpoint:**`POST /api/cron/academy-rollover`

**Authentication:** Requires `Authorization: Bearer <CRON_SECRET>`

### Rollover Logic Flow

```
No

Yes

Yes

No

Yes

No

handleAcademyRollover triggered

listAcademySeasons()

Filter: status='open' AND end_date < today

Any expired?

Return {rolledOver: 0, seasonsClosed: 0}

For each expired season

getUpcomingAcademySeason(age_group_id)

listWaitlistBySeason(season_id)

Filter: status IN ('waiting', 'invited')

For each rollable entry

hasAcceptedEntryForEnquiry(enquiry_id)

rolloverWaitlistEntry(entry, newSeason, oldSeason, isReturning)

More entries?

updateSeasonStatus(season, 'closed')

More expired seasons?

Return {rolledOver: N, seasonsClosed: M}

End
```

### Rollover Behavior Examples

**Scenario 1: Athlete with no prior acceptance**

- Entry: `status = 'waiting'`, `season_id = 'acs_2025'`
- `hasAcceptedEntryForEnquiry('enq_001')` → `false`
- Creates new entry: `season_id = 'acs_2026'`, `is_returning = 0`, `rolled_over_from_season_id = 'acs_2025'`

**Scenario 2: Returning athlete**

- Entry: `status = 'invited'`, `season_id = 'acs_2025'`
- `hasAcceptedEntryForEnquiry('enq_002')` → `true` (accepted in 2024 season)
- Creates new entry: `season_id = 'acs_2026'`, `is_returning = 1`, `rolled_over_from_season_id = 'acs_2025'`

**Scenario 3: No upcoming season**

- Entry: `status = 'waiting'`, `season_id = 'acs_2025'`
- `getUpcomingAcademySeason()` → `null`
- Creates new entry: `season_id = null` (unassigned pool), `is_returning = 0`

**Scenario 4: Finalized entries**

- Entry: `status = 'accepted'` or `status = 'declined'`
- **Not rolled over** — remains linked to original season

**Code:**[tests/unit/cron.test.ts500-606](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/cron.test.ts#L500-L606)

**Sources:**[tests/unit/cron.test.ts450-606](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/cron.test.ts#L450-L606)

---

## Code Entity Reference

### Key Files
FilePurpose`src/pages/admin/academy.astro`Main admin UI page for academy management`src/lib/repositories/academy.js`Data access layer for seasons and waitlist`src/api/cron/academy-rollover.js`Automated season transition handler`src/lib/business/age.ts`Age group resolution logic (determines waitlist routing)
### Key Functions
FunctionFileDescription`listAcademySeasons``academy.js`Fetch all seasons`listWaitlistBySeason``academy.js`Fetch entries for a season`listUnassignedWaitlist``academy.js`Fetch entries with `season_id = null``updateSeasonStatus``academy.js`Change season status (admin or cron)`updateWaitlistStatus``academy.js`Change entry status (admin only)`rolloverWaitlistEntry``academy.js`Migrate entry to next season`hasAcceptedEntryForEnquiry``academy.js`Check if athlete has prior acceptance`getUpcomingAcademySeason``academy.js`Find next season for rollover target`handleAcademyRollover``academy-rollover.js`Cron handler for season transitions`routeEnquiry``enquiry.ts`Determines if enquiry goes to waitlist
### Database Tables
TableKey Fields`academy_seasons``id`, `code`, `label`, `age_group_id`, `start_date`, `end_date`, `capacity`, `status``academy_waitlist``id`, `enquiry_id`, `season_id`, `token`, `status`, `is_returning`, `rolled_over_from_season_id`
**Sources:**[src/pages/admin/academy.astro1-298](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/academy.astro#L1-L298)[src/lib/business/enquiry.ts69-72](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/business/enquiry.ts#L69-L72)[tests/unit/cron.test.ts36-43](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/cron.test.ts#L36-L43)

---

## Enquiry Enrichment Pattern

The admin UI enriches waitlist entries with enquiry details for display:

```
// Cache to avoid duplicate DB queries
const enquiryCache = new Map<string, { name: string | null; email: string }>();
 
async function getEnquiryDetails(enquiryId: string) {
  const cached = enquiryCache.get(enquiryId);
  if (cached) return cached;
  const enquiry = await getEnquiryById(enquiryId, env);
  const details = { name: enquiry?.name ?? null, email: enquiry?.email ?? '' };
  enquiryCache.set(enquiryId, details);
  return details;
}
 
async function enrich(entries: AcademyWaitlistEntry[]): Promise<EnrichedWaitlist[]> {
  return Promise.all(
    entries.map(async (entry) => {
      const details = await getEnquiryDetails(entry.enquiry_id);
      return { ...entry, enquiry_name: details.name, enquiry_email: details.email };
    })
  );
}
```

This pattern batches enrichment via `Promise.all` while caching enquiry lookups to minimize redundant DB queries when the same enquiry appears in multiple seasons.

**Code:**[src/pages/admin/academy.astro68-95](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/academy.astro#L68-L95)

**Sources:**[src/pages/admin/academy.astro68-95](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/academy.astro#L68-L95)

---

## Helper Functions

### Status Badge Styling

The `statusBadge()` function maps status values to Tailwind CSS classes:

```
function statusBadge(status: string): string {
  switch (status) {
    case 'accepted': return 'bg-emerald-100 text-emerald-700';
    case 'invited': return 'bg-blue-100 text-blue-700';
    case 'declined': return 'bg-rose-100 text-rose-700';
    case 'waiting': return 'bg-amber-100 text-amber-700';
    case 'ineligible':
    case 'withdrawn': return 'bg-slate-100 text-slate-600';
    default: return 'bg-slate-100 text-slate-600';
  }
}
```

**Code:**[src/pages/admin/academy.astro113-129](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/academy.astro#L113-L129)

### Date Formatting

The `formatDate()` function renders ISO date strings in UK locale:

```
function formatDate(iso: string | null): string {
  if (!iso) return '—';
  return new Date(iso).toLocaleDateString('en-GB', {
    day: '2-digit',
    month: 'short',
    year: 'numeric',
  });
}
```

**Output example:**`31 Aug 2026`

**Code:**[src/pages/admin/academy.astro104-111](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/academy.astro#L104-L111)

**Sources:**[src/pages/admin/academy.astro104-129](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/academy.astro#L104-L129)
