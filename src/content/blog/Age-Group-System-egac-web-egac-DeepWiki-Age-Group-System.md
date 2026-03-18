# Age Group System
Relevant source files
- [db/migrations/0001_initial_schema.sql](https://github.com/egac-web/egac/blob/0ba542fc/db/migrations/0001_initial_schema.sql)
- [db/seed.sql](https://github.com/egac-web/egac/blob/0ba542fc/db/seed.sql)
- [docs/snagging.md](https://github.com/egac-web/egac/blob/0ba542fc/docs/snagging.md)
- [src/lib/business/age.ts](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/business/age.ts)
- [src/lib/business/enquiry.ts](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/business/enquiry.ts)
- [src/lib/business/templates.ts](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/business/templates.ts)
- [src/lib/business/tokens.ts](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/business/tokens.ts)
- [src/lib/repositories/age-groups.ts](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/repositories/age-groups.ts)
- [src/lib/repositories/bookings.ts](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/repositories/bookings.ts)
- [tests/unit/business.test.ts](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/business.test.ts)
- [tests/unit/db.test.ts](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/db.test.ts)
- [vitest.config.ts](https://github.com/egac-web/egac/blob/0ba542fc/vitest.config.ts)
- [wrangler.toml](https://github.com/egac-web/egac/blob/0ba542fc/wrangler.toml)

The age group system determines athlete eligibility for different training programs (taster sessions vs academy waitlist) based on UK Athletics age classification rules. This system calculates an athlete's age on a specific reference date (31 August of the athletics season) and matches them to the appropriate age group configuration stored in the `age_groups` table.

For information about how enquiries are routed to taster sessions or the academy waitlist based on age group classification, see [Enquiry Routing](/egac-web/egac/6.2-enquiry-routing). For details on generating available session dates for each age group, see [Session Scheduling](/egac-web/egac/6.3-session-scheduling).

---

## UK Athletics Age Group Rules

UK Athletics defines age groups based on the athletics season, which runs from 1 September to 31 August. An athlete's age group is determined by their age on **31 August at the end of the athletics season** the session falls within, not their age on the session date itself.

### Athletics Season Boundary Logic

```
Season 2026/27

Season 2025/26

Age on this date
determines group

Age on this date
determines group

1 Sep 2025

31 Aug 2026
(reference date)

1 Sep 2026

31 Aug 2027
(reference date)

Session: 14 Oct 2026

Session: 14 Apr 2026

U13 if age 11-12

U13 if age 11-12
```

**Key Rule**: If a session date falls in September or later (months 8+), the reference date is 31 August of the **following** calendar year. If the session falls in August or earlier, the reference date is 31 August of the **same** calendar year.

**Example**:

- Session on 14 October 2026 → Season ends 31 August 2027 → Age calculated on 31 Aug 2027
- Athlete born 15 September 2014 → Age on 31 Aug 2027 = 12 → Classified as U13
- Athlete born 1 September 2012 → Age on 31 Aug 2027 = 14 → Classified as U15

Sources: [src/lib/business/age.ts1-44](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/business/age.ts#L1-L44)[db/migrations/0001_initial_schema.sql70-86](https://github.com/egac-web/egac/blob/0ba542fc/db/migrations/0001_initial_schema.sql#L70-L86)

---

## Age Calculation Functions

### Core Calculation Flow

```
Yes (8-11)

No (0-7)

No

Yes

Input:
DOB + Session Date

athleticsSeasonEndYear()
Determine which Aug 31

Session month
≥ September?

Return year + 1

Return year

Build reference date:
YYYY-08-31

ageOnDate()
Calculate whole years

Birthday
before Aug 31?

age = year_diff - 1

age = year_diff

Return age
(used for matching)
```

### Function Hierarchy
FunctionPurposeReturnsLocation`athleticsSeasonEndYear(dateIso)`Determines which calendar year's 31 August ends the season`number` (year)[src/lib/business/age.ts38-44](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/business/age.ts#L38-L44)`ageOnDate(dobIso, referenceDateIso)`Calculates age in whole years on a reference date`number | null`[src/lib/business/age.ts51-68](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/business/age.ts#L51-L68)`ageForAthletics(dobIso, sessionDateIso)`Calculates age on 31 Aug of the relevant season`number | null`[src/lib/business/age.ts76-83](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/business/age.ts#L76-L83)`resolveAgeGroup(dobIso, sessionDateIso, ageGroups)`Matches athlete to an age group config`AgeGroupConfig | null`[src/lib/business/age.ts119-135](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/business/age.ts#L119-L135)`isAgeGroupEligible(dobIso, sessionDateIso, ageGroup)`Validates if DOB falls within a group's boundaries`boolean`[src/lib/business/age.ts143-151](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/business/age.ts#L143-L151)
### Implementation Details

The `ageForAthletics` function is the primary entry point used throughout the system. It delegates to `athleticsSeasonEndYear` to determine the reference date, then calls `ageOnDate` to calculate the age:

```
ageForAthletics('2014-09-15', '2026-10-14')
  ↓
athleticsSeasonEndYear('2026-10-14') → 2027
  ↓
ageOnDate('2014-09-15', '2027-08-31') → 12

```

The `ageOnDate` function handles birthday logic correctly by comparing both month and day to determine if the birthday has occurred on the reference date.

Sources: [src/lib/business/age.ts38-83](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/business/age.ts#L38-L83)[tests/unit/business.test.ts105-162](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/business.test.ts#L105-L162)

---

## Age Group Configuration Schema

### Database Table Structure

Age groups are stored in the `age_groups` table in D1 with the following schema:
ColumnTypeDescription`id`TEXTPrimary key (e.g. `ag_u13`)`code`TEXTUnique short code (e.g. `u13`)`label`TEXTDisplay name (e.g. `U13`)`booking_type`TEXT`'taster'` or `'waitlist'` — determines routing`age_min_aug31`INTEGERMinimum age on 31 August (inclusive)`age_max_aug31`INTEGERMaximum age on 31 August (inclusive)`session_days`TEXTJSON array: `["Tuesday"]` or `["Saturday","Tuesday"]``session_time`TEXTStart time (e.g. `18:30`), nullable for academy`capacity_per_session`INTEGERMax bookings per session (taster groups only)`season_capacity`INTEGERTotal season capacity (academy groups only)`season_start_month`INTEGER1-12, NULL for year-round groups`season_end_month`INTEGER1-12, NULL for year-round groups`active`INTEGER0 or 1 — inactive groups hidden from selection`sort_order`INTEGERDisplay order (lower = higher priority)
### AgeGroupConfig Interface

The business logic layer defines a TypeScript interface matching this schema:

[src/lib/business/age.ts90-105](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/business/age.ts#L90-L105)

```
export interface AgeGroupConfig {
  id: string;
  code: string;
  label: string;
  booking_type: 'taster' | 'waitlist';
  age_min_aug31: number;
  age_max_aug31: number;
  session_days: string; // JSON array string
  session_time: string | null;
  capacity_per_session: number | null;
  season_capacity: number | null;
  season_start_month: number | null;
  season_end_month: number | null;
  active: number;
  sort_order: number;
}
```

This interface is used by all pure business logic functions to avoid circular dependencies with the database schema types.

Sources: [db/migrations/0001_initial_schema.sql88-111](https://github.com/egac-web/egac/blob/0ba542fc/db/migrations/0001_initial_schema.sql#L88-L111)[src/lib/business/age.ts86-105](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/business/age.ts#L86-L105)

---

## Standard Age Group Configurations

The system is seeded with the following age group configurations:
CodeLabelAgesBooking TypeSession DaysCapacityActive`u11`U11 Academy9-10`waitlist`Sat, Tue40 seasonNo (0)`u13`U1311-12`taster`Tuesday2 per sessionYes (1)`u15`U15 and older13-120`taster`Tuesday2 per sessionYes (1)`u17`U1715-16`taster`Tuesday2 per sessionNo (0)`u20`U2017-19`taster`Tuesday2 per sessionNo (0)
**Note**: The U15 group uses `age_max_aug31: 120` to capture all athletes aged 13 and above who don't match more specific groups. This is an intentional catch-all configuration.

**Inactive Groups**: U17 and U20 are currently inactive but remain in the database for future activation or historical reporting.

Sources: [db/seed.sql12-53](https://github.com/egac-web/egac/blob/0ba542fc/db/seed.sql#L12-L53)

---

## Age Group Resolution Process

### Resolution Algorithm

```
null

number

Yes

No

resolveAgeGroup()
Input: DOB, sessionDate, ageGroups[]

ageForAthletics()
Calculate age on Aug 31

Age
calculable?

Return null

Filter to active groups
(active = 1)

Sort by sort_order ASC

Find first group where:
age ≥ age_min_aug31
AND
age ≤ age_max_aug31

Match
found?

Return AgeGroupConfig
```

### Resolution Rules

1. **Age Validation**: If DOB is null, empty, or unparseable, resolution returns `null`
2. **Active Groups Only**: Only age groups with `active = 1` are considered
3. **Sort Priority**: Groups are evaluated in `sort_order` ascending order
4. **First Match Wins**: The first group whose age range contains the athlete's age is returned
5. **Boundary Inclusive**: Both `age_min_aug31` and `age_max_aug31` are inclusive boundaries
6. **Type Coercion**: The function handles D1 returning integers as strings (e.g. `active: "1"`)

### Usage Example

The resolution function is called in two primary contexts:

1. **POST /api/enquiry**: When a public user submits an enquiry form
2. **POST /api/booking**: When a user creates a booking from an invite

The resolved age group's `booking_type` field determines the routing path (see [Enquiry Routing](/egac-web/egac/6.2-enquiry-routing)).

Sources: [src/lib/business/age.ts119-135](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/business/age.ts#L119-L135)[tests/unit/business.test.ts194-252](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/business.test.ts#L194-L252)

---

## Session Day Configuration

### Session Days JSON Format

The `session_days` field stores a JSON array of day names:

```
["Tuesday"]
```

or for academy groups:

```
["Saturday","Tuesday"]
```

### Session Day Functions

```
Inputs

Pure Functions

string[]

boolean

string[]

boolean

getSessionDays()
Parse JSON

isValidSessionDay()
Check date matches

getNextNSessionDates()
Generate future dates

validateSessionDate()
Validate for booking

AgeGroupConfig

Date (ISO string)

Count (N)

['Tuesday']

true/false

['2026-10-13',
'2026-10-20', ...]

true/false
```

### Implementation Details
FunctionPurposeReturns`getSessionDays(ageGroup)`Parses `session_days` JSON, defaults to `["Tuesday"]` on error`string[]``isValidSessionDay(dateIso, ageGroup)`Checks if date falls on a valid session day`boolean``getNextNSessionDates(ageGroup, n, fromDate)`Returns next N valid session dates (starting from tomorrow)`string[]``validateSessionDate(dateIso, ageGroup, today)`Validates date is valid, on correct day, and in future`boolean`
### Date Generation Algorithm

The `getNextNSessionDates` function generates available booking dates:

1. Starts from **tomorrow** (never returns today)
2. Iterates through calendar days
3. Checks if each day matches a valid session day name
4. Collects N matching dates
5. Returns as ISO date strings (YYYY-MM-DD)

This is used to populate the booking form date picker when a user accepts an invite.

Sources: [src/lib/business/age.ts153-210](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/business/age.ts#L153-L210)[tests/unit/business.test.ts275-341](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/business.test.ts#L275-L341)

---

## Session Time Handling

Session times are stored in `session_time` as strings (e.g. `18:30`) and are associated with specific age groups. Different age groups can have different start times:

- **U13**: 18:30
- **U15+**: 18:30
- **U17**: 19:30 (when active)
- **U20**: 19:30 (when active)
- **U11 Academy**: NULL (multiple session times across Saturday and Tuesday)

The `session_time` field is:

- Required for taster groups (displayed in booking confirmations)
- Optional for academy groups (waitlist doesn't need specific times)
- Stored in the bookings table when a booking is created
- Used in email templates for confirmation and reminder emails

Sources: [db/seed.sql28-53](https://github.com/egac-web/egac/blob/0ba542fc/db/seed.sql#L28-L53)[src/lib/repositories/bookings.ts9-36](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/repositories/bookings.ts#L9-L36)

---

## Repository Integration

### Age Groups Repository

The `age-groups.ts` repository provides data access functions:
FunctionPurposeSQL`listAgeGroups(env, activeOnly)`Fetch all or active age groups`WHERE active = 1` (optional)`getAgeGroupById(id, env)`Fetch single age group by ID`WHERE id = ?``getAgeGroupByCode(code, env)`Fetch single age group by code`WHERE code = ? AND active = 1``getTasterAgeGroups(env)`Fetch taster groups`WHERE booking_type = 'taster' AND active = 1``getWaitlistAgeGroups(env)`Fetch academy groups`WHERE booking_type = 'waitlist' AND active = 1``updateAgeGroup(id, updates, env)`Update age group fieldsDynamic SET clause
### Integration Points

```
D1 Database

Business Logic

Repository Layer

API Layer

Pass configs

Pass config

Pass config

POST /api/enquiry

POST /api/booking

GET /api/admin/settings

listAgeGroups()

getAgeGroupById()

updateAgeGroup()

resolveAgeGroup()

validateSessionDate()

getNextNSessionDates()

age_groups table
```

**Pattern**: Repository functions fetch age group configurations from D1, then pass them to pure business logic functions which perform calculations without I/O.

Sources: [src/lib/repositories/age-groups.ts1-117](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/repositories/age-groups.ts#L1-L117)

---

## Data Flow: Enquiry to Age Group

### Complete Resolution Flow

```
routeEnquiry()
resolveAgeGroup()
D1 age_groups
listAgeGroups()
POST /api/enquiry
Enquiry Form
routeEnquiry()
resolveAgeGroup()
D1 age_groups
listAgeGroups()
POST /api/enquiry
Enquiry Form
alt
[booking_type = 'taster']
[booking_type = 'waitlist']
Submit with DOB
Fetch active age groups
SELECT * WHERE active=1
AgeGroupConfig[]
Return configs
resolveAgeGroup(DOB, today, configs)
ageForAthletics(DOB, today)
Find first matching range
Return AgeGroupConfig
routeEnquiry(ageGroup)
'taster'
Create invite record
'waitlist'
Add to academy_waitlist
```

### Error Handling
ScenarioBehaviorMissing DOB`resolveAgeGroup()` returns `null`, defaults to `'taster'` routingInvalid DOB format`ageForAthletics()` returns `null`, resolution returns `null`Age out of rangeNo matching group found, resolution returns `null`, defaults to `'taster'`No active groupsResolution returns `null`, defaults to `'taster'`D1 query failureAPI returns 500 error, transaction rolls back
Sources: [src/lib/business/age.ts119-135](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/business/age.ts#L119-L135)[src/lib/business/enquiry.ts62-72](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/business/enquiry.ts#L62-L72)

---

## Testing Strategy

The age group system has comprehensive unit test coverage:

### Test Coverage
Test SuiteFileCoverageSeason boundary logic[tests/unit/business.test.ts105-125](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/business.test.ts#L105-L125)5 casesAge calculation[tests/unit/business.test.ts131-162](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/business.test.ts#L131-L162)7 casesAge on date edge cases[tests/unit/business.test.ts168-188](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/business.test.ts#L168-L188)5 casesAge group resolution[tests/unit/business.test.ts194-252](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/business.test.ts#L194-L252)9 casesSession date validation[tests/unit/business.test.ts278-309](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/business.test.ts#L278-L309)5 casesDate generation[tests/unit/business.test.ts315-341](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/business.test.ts#L315-L341)3 casesEligibility checks[tests/unit/business.test.ts397-418](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/business.test.ts#L397-L418)4 cases
### Key Test Scenarios

**Athletics Season Boundaries**:

- Session in April → Reference date same August
- Session on 31 August → Reference date same August
- Session in September → Reference date next August
- Session in October → Reference date next August

**Age Calculation Edge Cases**:

- Birthday on 31 August (cutoff date)
- Birthday 1 day before cutoff
- Birthday 1 day after cutoff
- Null/missing DOB handling

**Resolution Edge Cases**:

- Multiple overlapping groups (sort_order precedence)
- D1 returning `active` as string `"1"` instead of integer
- No matching group (out of range ages)
- All groups inactive

Sources: [tests/unit/business.test.ts1-499](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/business.test.ts#L1-L499)

---

## Configuration Management

Age groups can be modified through the admin interface at `/admin/settings`:

### Editable Fields

Administrators can update:

- `label` — Display name
- `age_min_aug31` / `age_max_aug31` — Age boundaries
- `session_days` — JSON array of valid days
- `session_time` — Start time
- `capacity_per_session` — Booking capacity
- `active` — Enable/disable group
- `sort_order` — Display and matching priority

### Non-Editable Fields

The following fields are immutable after creation to maintain data integrity:

- `id` — Primary key
- `code` — Used in URLs and reporting
- `booking_type` — Changing would break enquiry routing

### Update Flow

1. Admin modifies fields in Settings UI
2. POST to `/api/admin/settings/age-groups/:id`
3. `updateAgeGroup()` validates and applies changes
4. Updates `updated_at` timestamp
5. Returns updated `AgeGroup` entity

Changes to age groups take effect immediately for new enquiries. Existing enquiries retain their originally assigned `age_group_id` for historical accuracy.

Sources: [src/lib/repositories/age-groups.ts47-117](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/repositories/age-groups.ts#L47-L117)[docs/snagging.md28-30](https://github.com/egac-web/egac/blob/0ba542fc/docs/snagging.md#L28-L30)

---

## Constants and Configuration

### Global Constants

The following constants define system-wide age group behavior:

[src/lib/business/age.ts20-22](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/business/age.ts#L20-L22)

```
export const SESSION_DAY_OF_WEEK = 2; // Tuesday (0=Sun)
export const ATHLETICS_SEASON_CUTOFF_MONTH = 8; // August (0-indexed)
export const ATHLETICS_SEASON_CUTOFF_DAY = 31;
```

These constants are used by:

- `athleticsSeasonEndYear()` — Determines season boundaries
- Date validation functions — Ensures consistency across the system

**Note**: `ATHLETICS_SEASON_CUTOFF_MONTH` uses JavaScript's 0-indexed months where January = 0, so August = 8 (though this appears as month 9 in some contexts).

Sources: [src/lib/business/age.ts20-22](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/business/age.ts#L20-L22)

---

## Summary

The age group system is a pure business logic layer built on UK Athletics rules. It:

1. **Calculates ages** using the 31 August cutoff date for the relevant athletics season
2. **Resolves athletes** to age groups by matching their age against configurable ranges
3. **Determines routing** via the `booking_type` field (taster vs waitlist)
4. **Generates session dates** respecting each group's valid session days
5. **Validates bookings** ensuring dates fall on correct days and in the future

All age group configuration is stored in the `age_groups` table and can be modified through the admin interface without code changes. The system handles edge cases like birthday boundaries, inactive groups, and overlapping configurations through well-defined precedence rules.

Sources: [src/lib/business/age.ts1-271](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/business/age.ts#L1-L271)[db/migrations/0001_initial_schema.sql88-111](https://github.com/egac-web/egac/blob/0ba542fc/db/migrations/0001_initial_schema.sql#L88-L111)[src/lib/repositories/age-groups.ts1-117](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/repositories/age-groups.ts#L1-L117)