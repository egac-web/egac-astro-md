# Business Logic
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

This page documents the core business logic layer of the EGAC platform—the pure computation functions that implement domain rules, validation, and calculations without performing I/O operations. Business logic functions are deterministic, testable in isolation, and form the calculation engine that drives enquiry processing, age group resolution, and booking validation.

For details on specific subsystems, see:

- Age group resolution and UK Athletics season rules: [Age Group System](/egac-web/egac/6.1-age-group-system)
- Enquiry routing between taster sessions and academy waitlist: [Enquiry Routing](/egac-web/egac/6.2-enquiry-routing)
- Session date generation and validation: [Session Scheduling](/egac-web/egac/6.3-session-scheduling)
- Template rendering and token generation: [Template & Token Systems](/egac-web/egac/6.4-template-and-token-systems)

For how business logic is consumed by API handlers, see [API Reference](/egac-web/egac/5-api-reference). For data persistence that happens before/after business logic execution, see [Data Access Layer](/egac-web/egac/7-data-access-layer).

---

## Purpose and Scope

The business logic layer exists to:

1. **Enforce domain rules** that are independent of infrastructure (e.g., UK Athletics age group calculations based on 31 August cutoff dates)
2. **Enable unit testing** by keeping computation separate from database queries and HTTP requests
3. **Prevent duplication** by centralizing calculations used across multiple API endpoints
4. **Document rules explicitly** through typed function signatures and pure implementations

All business logic modules live in `src/lib/business/` and export only pure functions—no database clients, no environment variables, no HTTP calls. They accept primitive values or typed interfaces as input and return computed results.

**Sources:**[src/lib/business/age.ts1-19](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/business/age.ts#L1-L19)[src/lib/business/enquiry.ts1-4](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/business/enquiry.ts#L1-L4)[src/lib/business/templates.ts1-7](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/business/templates.ts#L1-L7)

---

## Architecture Overview

The EGAC platform follows a strict layered architecture where business logic sits between the API layer and the repository layer:

```
Data Layer

Repository Layer

Business Logic Layer

API Layer

1.validateEnquiryData()

2.resolveAgeGroup()

3.routeEnquiry()

4.insertEnquiry()

1.validateSessionDate()

2.resolveAgeGroup()

3.createBooking()

validation

listAgeGroups()

handleEnquiry
(src/api/enquiry.ts)

handleBooking
(src/api/booking.ts)

handleAcademyRespond
(src/api/academy.ts)

src/lib/business/age.ts
• resolveAgeGroup()
• ageForAthletics()
• validateSessionDate()
• getNextNSessionDates()

src/lib/business/enquiry.ts
• validateEnquiryData()
• routeEnquiry()
• formatEnquiryForEmail()

src/lib/business/templates.ts
• renderTemplate()
• validateTemplateVariables()

src/lib/business/tokens.ts
• generateSecureToken()
• isTokenExpired()
• isInviteExpired()

enquiries.ts
insertEnquiry()
getEnquiryByEmail()

invites.ts
createInviteForEnquiry()
markInviteSent()

bookings.ts
createBooking()
countBookingsForDateAndGroup()

age-groups.ts
listAgeGroups()
getAgeGroupById()

D1 Database
```

**Key architectural rules:**

1. **Business logic never imports repositories** — it receives data as function arguments
2. **Business logic never uses `Env`** — infrastructure concerns stay in API/repository layers
3. **Repositories never contain business rules** — they only execute queries
4. **API handlers orchestrate** — they call business logic, then repositories, then more business logic

**Sources:**[src/api/enquiry.ts1-223](https://github.com/egac-web/egac/blob/0ba542fc/src/api/enquiry.ts#L1-L223)[src/api/booking.ts1-147](https://github.com/egac-web/egac/blob/0ba542fc/src/api/booking.ts#L1-L147)[tests/unit/business.test.ts1-499](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/business.test.ts#L1-L499)

---

## Business Logic Modules

### Module: `src/lib/business/age.ts`

The age module implements UK Athletics age group rules and session date calculations. It is the most complex business logic module because athletics age groups do not follow calendar years—they are determined by an athlete's age on **31 August at the end of the athletics season**, not the session date.
FunctionPurposeInputsOutput`ageForAthletics()`Calculate an athlete's athletics age for a session`dobIso`, `sessionDateIso``number | null``athleticsSeasonEndYear()`Determine which August 31 ends the season`dateIso``number``resolveAgeGroup()`Find matching age group from D1 list`dobIso`, `sessionDateIso`, `ageGroups[]``AgeGroupConfig | null``validateSessionDate()`Check if date is valid for age group`dateIso`, `ageGroup`, `today?``boolean``getNextNSessionDates()`Generate N future session dates for age group`ageGroup`, `n`, `fromDate?``string[]``isAgeGroupEligible()`Check if athlete fits in specific age group`dobIso`, `sessionDateIso`, `ageGroup``boolean`
**Example: Athletics Season Logic**

```
// Session on 14 October 2026 falls in the 2026/27 season
// The season ends on 31 August 2027
athleticsSeasonEndYear('2026-10-14') // → 2027
 
// Athlete born 15 September 2014:
// Age on 31 August 2027 = 12 years old → U13
ageForAthletics('2014-09-15', '2026-10-14') // → 12
```

The `resolveAgeGroup()` function is called by both enquiry submission and booking confirmation to ensure athletes are always placed in the correct group based on their DOB and the session date.

**Sources:**[src/lib/business/age.ts1-271](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/business/age.ts#L1-L271)[tests/unit/business.test.ts105-252](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/business.test.ts#L105-L252)

### Module: `src/lib/business/enquiry.ts`

Handles enquiry data validation and routing decisions.
FunctionPurpose`validateEnquiryData()`Throws on invalid contact details, email format, or implausible ages`routeEnquiry()`Returns `'taster'` or `'waitlist'` based on `booking_type` of resolved age group`formatEnquiryForEmail()`Sanitizes enquiry fields for safe template rendering
**Routing Logic:**

```
booking_type='taster'

booking_type='waitlist'

null (no DOB)

Enquiry Submitted

resolveAgeGroup()
(business/age.ts)

routeEnquiry()
(business/enquiry.ts)

Create Invite
Send booking email

Add to Waitlist
Send academy email

Default to taster
```

The routing decision is based entirely on the `booking_type` field of the resolved age group. This field is stored in the `age_groups` table and managed via the admin interface.

**Sources:**[src/lib/business/enquiry.ts1-87](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/business/enquiry.ts#L1-L87)[src/api/enquiry.ts166-210](https://github.com/egac-web/egac/blob/0ba542fc/src/api/enquiry.ts#L166-L210)[tests/unit/business.test.ts257-272](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/business.test.ts#L257-L272)

### Module: `src/lib/business/templates.ts`

Template rendering with strict variable validation.
FunctionPurpose`renderTemplate()`Replace `{{variable}}` placeholders; throws if any variable is missing`extractTemplateVariables()`Return all `{{variable}}` names found in a template`validateTemplateVariables()`Return list of undeclared variables (admin template editor validation)`buildSampleVars()`Merge custom overrides with `SAMPLE_TEMPLATE_VARS` for previews
**Rendering guarantees:**

1. **No unfilled placeholders** — `renderTemplate()` throws if `{{missing_var}}` has no value provided
2. **Case-sensitive matching** — `{{name}}` ≠ `{{Name}}`
3. **Extra variables ignored** — passing `{ name, extra }` when template only uses `{{name}}` is safe

This strictness prevents emails from being sent with placeholder text visible to users. The admin template editor uses `validateTemplateVariables()` to warn about typos before saving.

**Sources:**[src/lib/business/templates.ts1-112](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/business/templates.ts#L1-L112)[tests/unit/business.test.ts455-486](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/business.test.ts#L455-L486)

### Module: `src/lib/business/tokens.ts`

Secure token generation and expiry checking.
FunctionPurposeTTL`generateSecureToken()`Generate cryptographically random hex tokenN/A`isInviteExpired()`Check if invite token is older than 14 days14 days`isMembershipOtpExpired()`Check if membership OTP is older than 7 days7 days`isTokenExpired()`Generic expiry check with custom TTLCustom
All token generation uses `crypto.getRandomValues()` under the hood (via `src/lib/db/client.ts:generateToken`). Tokens are URL-safe hex strings with configurable byte length (default 24 bytes = 48 hex chars).

**Sources:**[src/lib/business/tokens.ts1-71](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/business/tokens.ts#L1-L71)[tests/unit/business.test.ts424-449](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/business.test.ts#L424-L449)

---

## Business Logic Usage Patterns

### Pattern 1: Validation Before Persistence

API handlers validate input using business logic before touching the database:

```
// src/api/enquiry.ts:113-125
try {
  validateEnquiryData({
    contact_name: enquirerName,
    contact_email: enquirerEmail,
    dob: athleteDob,
    interest: body.interest,
    training_days: body.training_days,
    phone: enquirerPhone,
  });
} catch (validationErr) {
  return err(message, 422, 'VALIDATION_ERROR', true);
}
 
// Only after validation passes:
const enquiry = await insertEnquiry({ ... }, env);
```

This prevents invalid data from entering the database and provides immediate feedback to users.

**Sources:**[src/api/enquiry.ts113-125](https://github.com/egac-web/egac/blob/0ba542fc/src/api/enquiry.ts#L113-L125)[tests/unit/api.test.ts254-263](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/api.test.ts#L254-L263)

### Pattern 2: Computation → Repository → Computation

Business logic is called before and after database operations to compute derived values:

```
// src/api/enquiry.ts:158-166
// 1. Fetch age groups from D1 (repository)
const allGroups = await listAgeGroups(env);
 
// 2. Compute age group (business logic)
const ageGroup = resolveAgeGroup(enquiry.dob, todayIso, allGroups);
 
// 3. Store computed value (repository)
if (ageGroup) {
  await updateEnquiry(enquiry.id, { age_group_id: ageGroup.id }, env);
}
 
// 4. Route based on computation (business logic)
const path = routeEnquiry(ageGroup);
```

The API handler orchestrates the sequence but delegates all computation to pure functions.

**Sources:**[src/api/enquiry.ts158-167](https://github.com/egac-web/egac/blob/0ba542fc/src/api/enquiry.ts#L158-L167)[src/api/booking.ts50-74](https://github.com/egac-web/egac/blob/0ba542fc/src/api/booking.ts#L50-L74)

### Pattern 3: Capacity Checking

Business logic validates constraints; repositories count existing data:

```
// src/api/booking.ts:73-87
// 1. Validate session date is valid for age group (business logic)
const validation = validateBookingRequest(date, enquiry.dob, ageGroup);
if (!validation.valid) return err(validation.reason, 422);
 
// 2. Count current bookings (repository)
const currentCount = await countBookingsForDateAndGroup(date, ageGroup.id, env);
 
// 3. Check against capacity (business logic comparison)
if (currentCount >= capacity) {
  return err('This session is full', 409, 'SLOT_FULL');
}
```

The business logic module defines what "full" means; the repository provides the current state.

**Sources:**[src/api/booking.ts73-87](https://github.com/egac-web/egac/blob/0ba542fc/src/api/booking.ts#L73-L87)[src/lib/repositories/bookings.ts62-78](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/repositories/bookings.ts#L62-L78)

---

## Testing Strategy

All business logic modules have comprehensive unit tests in `tests/unit/business.test.ts`. Tests use no mocks because business logic has no dependencies—only pure functions with deterministic outputs.

### Test Coverage by Module
ModuleTest CasesLines Tested`age.ts`48 test cases`athleticsSeasonEndYear`, `ageForAthletics`, `resolveAgeGroup`, `validateSessionDate`, `getNextNSessionDates`, `isAgeGroupEligible``enquiry.ts`12 test cases`validateEnquiryData`, `routeEnquiry``templates.ts`8 test cases`renderTemplate`, `extractTemplateVariables`, `validateTemplateVariables``tokens.ts`6 test cases`generateSecureToken`, `isTokenExpired`, `isInviteExpired`
### Example Test Structure

```
// tests/unit/business.test.ts:131-140
it('athlete born 15 Sep 2014 at session Oct 2026 → age 11 (U13)', () => {
  // Session on 14 Oct 2026 falls in 2026/27 season
  // Season ends Aug 2027
  // Age on 31 Aug 2027 = 12 → U13
  expect(ageForAthletics('2014-09-15', '2026-10-14')).toBe(12);
});
```

Tests document expected behavior and serve as executable specifications for domain rules.

**Sources:**[tests/unit/business.test.ts1-499](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/business.test.ts#L1-L499)

---

## Relationship to Other Layers

```
Email System

Repositories

Business Logic Functions

Public Interface

1.validate

2.fetch groups

3.compute

4.route

5.persist

6.generate token

7.send email

1.fetch groups

2.compute

3.validate

4.persist

5.send email

uses

uses

uses

/api/enquiry

/api/booking

validateEnquiryData()
(enquiry.ts)

resolveAgeGroup()
(age.ts)

routeEnquiry()
(enquiry.ts)

validateSessionDate()
(age.ts)

renderTemplate()
(templates.ts)

generateSecureToken()
(tokens.ts)

insertEnquiry()

createInviteForEnquiry()

createBooking()

listAgeGroups()

sendBookingInvite()

sendBookingConfirmation()
```

Business logic functions are called by:

- **API handlers** for request validation and routing decisions
- **Email templates** for rendering variable placeholders
- **Cron jobs** for expiry checks and date calculations
- **Repositories** (rarely) when they need to compute IDs or timestamps

Business logic never calls:

- Database queries (no `env.DB`)
- External APIs (no `fetch()`)
- Email services (no `sendEmail()`)
- Other business logic modules (minimal coupling—only `age.ts` imports nothing)

**Sources:**[src/api/enquiry.ts1-223](https://github.com/egac-web/egac/blob/0ba542fc/src/api/enquiry.ts#L1-L223)[src/api/booking.ts1-147](https://github.com/egac-web/egac/blob/0ba542fc/src/api/booking.ts#L1-L147)[src/lib/email/templates.ts1-300](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/email/templates.ts#L1-L300)

---

## Key Design Principles

### 1. Pure Functions Only

Every exported function in `src/lib/business/` is deterministic:

```
// ✅ Pure — same inputs always produce same output
export function ageForAthletics(dobIso: string, sessionDateIso: string): number | null {
  const seasonEndYear = athleticsSeasonEndYear(sessionDateIso);
  const aug31 = `${seasonEndYear}-08-31`;
  return ageOnDate(dobIso, aug31);
}
 
// ❌ Impure — would violate business logic layer contract
export async function ageForAthletics(dobIso: string, env: Env): Promise<number> {
  const config = await getConfig(env); // I/O operation
  // ...
}
```

**Sources:**[src/lib/business/age.ts76-83](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/business/age.ts#L76-L83)

### 2. Fail Fast with Descriptive Errors

Business logic throws typed errors that API handlers convert to appropriate HTTP status codes:

```
// src/lib/business/enquiry.ts:27-29
if (!d.contact_name || typeof d.contact_name !== 'string' || !d.contact_name.trim()) {
  throw new Error('contact_name is required');
}
 
// API handler catches and returns 422
// src/api/enquiry.ts:122-125
} catch (validationErr) {
  const message = validationErr instanceof Error ? validationErr.message : 'Validation error';
  return err(message, 422, 'VALIDATION_ERROR', true);
}
```

**Sources:**[src/lib/business/enquiry.ts20-60](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/business/enquiry.ts#L20-L60)[src/api/enquiry.ts122-125](https://github.com/egac-web/egac/blob/0ba542fc/src/api/enquiry.ts#L122-L125)

### 3. Explicit Type Interfaces

Business logic defines its own interfaces rather than importing database schemas:

```
// src/lib/business/age.ts:90-105
export interface AgeGroupConfig {
  id: string;
  code: string;
  booking_type: 'taster' | 'waitlist';
  age_min_aug31: number;
  age_max_aug31: number;
  session_days: string; // JSON array
  // ...
}
```

This prevents circular dependencies and allows business logic to be tested independently of database schema changes.

**Sources:**[src/lib/business/age.ts90-105](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/business/age.ts#L90-L105)

### 4. Date Calculations Use ISO Strings

All date functions accept and return `YYYY-MM-DD` strings, not `Date` objects:

```
// ✅ Recommended — portable, testable, database-friendly
export function athleticsSeasonEndYear(dateIso: string): number
 
// ❌ Avoided — timezone-dependent, harder to test
export function athleticsSeasonEndYear(date: Date): number
```

ISO date strings are unambiguous, serialize cleanly to JSON, and match D1's date column format.

**Sources:**[src/lib/business/age.ts38-44](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/business/age.ts#L38-L44)[src/lib/business/age.ts258-260](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/business/age.ts#L258-L260)

---

## Summary

The business logic layer provides:
CapabilityModuleKey Exports**Age calculations**`age.ts``ageForAthletics()`, `resolveAgeGroup()`, `validateSessionDate()`**Enquiry routing**`enquiry.ts``validateEnquiryData()`, `routeEnquiry()`**Template rendering**`templates.ts``renderTemplate()`, `validateTemplateVariables()`**Token security**`tokens.ts``generateSecureToken()`, `isInviteExpired()`**Booking validation**`booking.ts``validateBookingRequest()`, `buildBookingConfirmation()`
All functions are:

- **Pure** — no side effects
- **Testable** — 74 unit tests with 100% coverage of core rules
- **Type-safe** — explicit TypeScript interfaces
- **Documented** — inline comments explain UK Athletics rules and edge cases

For implementation details of each subsystem, see the child pages listed at the top of this document.

**Sources:**[src/lib/business/age.ts1-271](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/business/age.ts#L1-L271)[src/lib/business/enquiry.ts1-87](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/business/enquiry.ts#L1-L87)[src/lib/business/templates.ts1-112](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/business/templates.ts#L1-L112)[src/lib/business/tokens.ts1-71](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/business/tokens.ts#L1-L71)[tests/unit/business.test.ts1-499](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/business.test.ts#L1-L499)