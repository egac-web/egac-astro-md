# Public User Interface
Relevant source files
- [astro.config.mjs](https://github.com/egac-web/egac/blob/0ba542fc/astro.config.mjs)
- [src/api/booking.ts](https://github.com/egac-web/egac/blob/0ba542fc/src/api/booking.ts)
- [src/api/enquiry.ts](https://github.com/egac-web/egac/blob/0ba542fc/src/api/enquiry.ts)
- [src/api/health.ts](https://github.com/egac-web/egac/blob/0ba542fc/src/api/health.ts)
- [src/layouts/Base.astro](https://github.com/egac-web/egac/blob/0ba542fc/src/layouts/Base.astro)
- [src/pages/index.astro](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/index.astro)
- [src/styles/tailwind.css](https://github.com/egac-web/egac/blob/0ba542fc/src/styles/tailwind.css)
- [tests/unit/api.test.ts](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/api.test.ts)

## Purpose and Scope

This page documents the public-facing components of the EGAC platform that athletes and parents interact with. It covers the enquiry form, booking system, and academy waitlist response mechanism—collectively comprising the entire public user experience.

For administrative interfaces used by club staff, see [Admin Interface](/egac-web/egac/4-admin-interface). For detailed API specifications of the public endpoints, see [Public APIs](/egac-web/egac/5.1-public-apis).

---

## System Overview

The public interface consists of three primary user journeys, all initiated from a single entry point at `src/pages/index.astro`:
JourneyEntry PointAPI EndpointOutcome**Enquiry Submission**Public form`POST /api/enquiry`Creates enquiry record, triggers age-based routing**Taster Booking**Email invite link`POST /api/booking`Confirms session reservation for 10+ athletes**Academy Response**Email waitlist link`POST /api/academy/respond`Records yes/no response for U11 Academy
The system uses **token-based authentication** for booking and academy flows—no user accounts or passwords are required. Each token is a single-use secure string embedded in email links.

**Sources:**

- [src/pages/index.astro1-339](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/index.astro#L1-L339)
- [src/api/enquiry.ts1-224](https://github.com/egac-web/egac/blob/0ba542fc/src/api/enquiry.ts#L1-L224)
- [src/api/booking.ts1-147](https://github.com/egac-web/egac/blob/0ba542fc/src/api/booking.ts#L1-L147)
- [tests/unit/api.test.ts199-401](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/api.test.ts#L199-L401)

---

## Public Page Structure

```
Base Layout

Submit enquiry

Select date

Accept/Decline

10+ years

Under 10

Click booking link

Click response link

uses

src/pages/index.astro
Public enquiry form

Booking page
(token-based URL)

Academy response
(token-based URL)

POST /api/enquiry

POST /api/booking

POST /api/academy/respond

Email: booking_invite

Email: academy_waitlist

Email: booking_confirmation

src/layouts/Base.astro
Tailwind CSS, header, footer
```

**Sources:**

- [src/pages/index.astro1-15](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/index.astro#L1-L15)
- [src/layouts/Base.astro1-44](https://github.com/egac-web/egac/blob/0ba542fc/src/layouts/Base.astro#L1-L44)
- [astro.config.mjs1-14](https://github.com/egac-web/egac/blob/0ba542fc/astro.config.mjs#L1-L14)

---

## Enquiry Form Architecture

The enquiry form at `src/pages/index.astro` implements progressive enhancement: it functions as a standard HTML form (POST with form encoding) but enhances to AJAX submission when JavaScript is available.

### Form Fields and Validation
FieldRequiredClient ValidationServer Validation`enquiry_for`Yes (radio)Always validDefaults to 'self'`name`YesNon-empty`validateEnquiryData``email`YesEmail regex`validateEnquiryData``phone`NoNoneNone`athlete_name`Conditional*Non-empty if otherRequired if `enquiry_for='other'``dob`YesNon-emptyDate format, age eligibility
* Required only when "enquiring on behalf of someone else" is selected

**Client-side validation** is implemented in the inline `<script>` block at [src/pages/index.astro177-338](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/index.astro#L177-L338) It displays errors both inline (per-field) and in a summary panel at the top of the form.

**Server-side validation** occurs in `handleEnquiry` via `validateEnquiryData` from the business logic layer. The server never trusts client input.

```

```

**Sources:**

- [src/pages/index.astro31-149](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/index.astro#L31-L149)
- [src/pages/index.astro177-338](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/index.astro#L177-L338)
- [src/api/enquiry.ts86-223](https://github.com/egac-web/egac/blob/0ba542fc/src/api/enquiry.ts#L86-L223)
- [tests/unit/api.test.ts199-401](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/api.test.ts#L199-L401)

---

## On-Behalf Enquiry Flow

The form supports two modes determined by the `enquiry_for` radio button:

- **Self**: Contact details and athlete details are the same person
- **Other**: Parent/guardian enters their contact info separately from athlete details

The client-side function `syncEnquiryMode()` at [src/pages/index.astro198-210](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/index.astro#L198-L210) toggles the visibility of the `athlete_name` field and updates label text dynamically.

### Field Mapping

When `enquiry_for='other'`, the submission includes:

```
{
  "enquiry_for": "other",
  "enquirer_name": "Parent Name",
  "enquirer_email": "parent@example.com",
  "enquirer_phone": "07700900000",
  "athlete_name": "Child Name",
  "athlete_dob": "2015-03-20"
}
```

The server normalizes this at [src/api/enquiry.ts93-102](https://github.com/egac-web/egac/blob/0ba542fc/src/api/enquiry.ts#L93-L102) by mapping to the database schema:

- `enquiry.name` = `enquirer_name` (contact person)
- `enquiry.email` = `enquirer_email`
- `enquiry.phone` = `enquirer_phone`
- `enquiry.dob` = `athlete_dob` (athlete's DOB, used for age group)
- `enquiry.note` = `"Enquiring about: {athlete_name}"`

**Sources:**

- [src/pages/index.astro109-122](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/index.astro#L109-L122)
- [src/pages/index.astro193-210](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/index.astro#L193-L210)
- [src/api/enquiry.ts93-110](https://github.com/egac-web/egac/blob/0ba542fc/src/api/enquiry.ts#L93-L110)
- [tests/unit/api.test.ts304-329](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/api.test.ts#L304-L329)

---

## Age-Based Routing Logic

After an enquiry is submitted, `handleEnquiry` resolves the athlete's age group using `resolveAgeGroup` and determines the path using `routeEnquiry`:

```
booking_type='taster'

booking_type='waitlist'

User clicks link

When invited

Enquiry submitted
dob provided

resolveAgeGroup(dob, today, allGroups)

routeEnquiry(ageGroup)

Create invite
sendBookingInvite

addToWaitlist
sendAcademyWaitlist

Email with booking link
+ available dates

Email confirming
waitlist position

Booking flow

Academy response flow
```

The `booking_type` field on the `age_groups` table drives this decision. Most age groups have `booking_type='taster'`; only younger groups (typically U11 and below) have `booking_type='waitlist'`.

**Sources:**

- [src/api/enquiry.ts158-166](https://github.com/egac-web/egac/blob/0ba542fc/src/api/enquiry.ts#L158-L166)
- [src/api/enquiry.ts168-210](https://github.com/egac-web/egac/blob/0ba542fc/src/api/enquiry.ts#L168-L210)
- Diagram 3 (Data Model) in high-level system architecture

---

## Taster Session Booking Flow

When an athlete is routed to the taster path, the system:

1. Creates an `invites` record with a secure token via `createInviteForEnquiry`
2. Generates a list of available session dates via `getNextNTuesdayDates`
3. Sends a `booking_invite` email with a link: `https://egac.pages.dev/book?token={token}`
4. User clicks link, selects date, and submits to `POST /api/booking`

### Token Lifecycle
StateDescriptionTransitions`pending`Created, email not yet sent→ `sent` (after email success)`sent`Email delivered successfully→ `accepted` (after booking) / `expired` (after 14 days)`accepted`Booking createdTerminal state`expired`14+ days old, no bookingTerminal state`failed`Email send failed 3+ timesTerminal state
The function `isInviteExpired` at [src/lib/business/tokens.js](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/business/tokens.js) checks if `created_at` is more than 14 days ago.

### Booking Validation

When processing `POST /api/booking`, `handleBooking` performs:

1. **Token lookup**: `getInviteByToken(token)` at [src/api/booking.ts37-38](https://github.com/egac-web/egac/blob/0ba542fc/src/api/booking.ts#L37-L38)
2. **Invite status check**: Rejects if `status='accepted'` or `'expired'` at [src/api/booking.ts40-44](https://github.com/egac-web/egac/blob/0ba542fc/src/api/booking.ts#L40-L44)
3. **Age group resolution**: Re-resolves based on `enquiry.dob` and `session_date` at [src/api/booking.ts51-62](https://github.com/egac-web/egac/blob/0ba542fc/src/api/booking.ts#L51-L62)
4. **Session validation**: `validateBookingRequest` checks date is valid for age group at [src/api/booking.ts73-74](https://github.com/egac-web/egac/blob/0ba542fc/src/api/booking.ts#L73-L74)
5. **Duplicate check**: `getBookingByInviteAndDate` prevents double-booking at [src/api/booking.ts77-78](https://github.com/egac-web/egac/blob/0ba542fc/src/api/booking.ts#L77-L78)
6. **Capacity check**: `countBookingsForDateAndGroup` enforces session limits at [src/api/booking.ts81-87](https://github.com/egac-web/egac/blob/0ba542fc/src/api/booking.ts#L81-L87)

**Sources:**

- [src/api/booking.ts25-146](https://github.com/egac-web/egac/blob/0ba542fc/src/api/booking.ts#L25-L146)
- [tests/unit/api.test.ts407-534](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/api.test.ts#L407-L534)

---

## Academy Waitlist Response Flow

For athletes routed to the academy path, the system:

1. Creates an `academy_waitlist` record via `addToWaitlist` at [src/api/enquiry.ts170](https://github.com/egac-web/egac/blob/0ba542fc/src/api/enquiry.ts#L170-L170)
2. Sends an `academy_waitlist` email confirming they've joined the queue
3. When the club sends invites (manual admin action), the athlete receives a link: `https://egac.pages.dev/academy?token={token}`
4. User responds via `POST /api/academy/respond` with `response='yes'` or `'no'`

### Response Handling

The `handleAcademyRespond` function at [src/api/academy.ts](https://github.com/egac-web/egac/blob/0ba542fc/src/api/academy.ts) (not in provided files, but tested at [tests/unit/api.test.ts540-659](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/api.test.ts#L540-L659)):

- Accepts `{token, response}` where `response` must be `'yes'` or `'no'`
- Calls `recordWaitlistResponse` to update the waitlist entry
- Is **idempotent**: re-submitting the same response returns 200 with `already_responded=true`
- Appends an event to the enquiry audit trail via `appendEnquiryEvent`

**Sources:**

- [src/api/enquiry.ts169-187](https://github.com/egac-web/egac/blob/0ba542fc/src/api/enquiry.ts#L169-L187)
- [tests/unit/api.test.ts540-659](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/api.test.ts#L540-L659)

---

## Email Delivery and Fault Tolerance

All email operations are **non-blocking** and **fault-tolerant**. The enquiry is always saved to the database first, and email failures do not prevent the API from returning success.

```
success=true

success=false

handleEnquiry

insertEnquiry

sendBookingInvite
or sendAcademyWaitlist

Return 201

console.error
```

From [src/api/enquiry.ts194-209](https://github.com/egac-web/egac/blob/0ba542fc/src/api/enquiry.ts#L194-L209):

```
if (result.success) {
  await markInviteSent(invite.id, env);
  // ...
} else {
  console.error(JSON.stringify({
    level: 'error',
    event: 'invite_email_failed',
    enquiry_id: enquiry.id,
    error: result.error,
  }));
  // Still saves invite_id to enquiry
}
```

This pattern ensures that:

- Users always receive a success message after submitting the form
- Failed emails are logged for admin review
- The cron job `handleRetryInvites` can retry failed sends later

See [Email System](/egac-web/egac/8-email-system) for complete email architecture.

**Sources:**

- [src/api/enquiry.ts168-220](https://github.com/egac-web/egac/blob/0ba542fc/src/api/enquiry.ts#L168-L220)
- [src/api/booking.ts108-136](https://github.com/egac-web/egac/blob/0ba542fc/src/api/booking.ts#L108-L136)
- [tests/unit/api.test.ts380-394](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/api.test.ts#L380-L394)
- [tests/unit/api.test.ts522-533](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/api.test.ts#L522-L533)

---

## Content-Type Support

The enquiry endpoint supports three content types to maximize compatibility:
Content-TypeUse CaseParsed By`application/json`JavaScript fetch()`parseBody<T>()``application/x-www-form-urlencoded`Plain HTML forms`request.formData()``multipart/form-data`Future file uploads`request.formData()`
The function `parsePublicEnquiryBody` at [src/api/enquiry.ts39-84](https://github.com/egac-web/egac/blob/0ba542fc/src/api/enquiry.ts#L39-L84) handles all three formats and normalizes them to a common object structure.

**Sources:**

- [src/api/enquiry.ts39-84](https://github.com/egac-web/egac/blob/0ba542fc/src/api/enquiry.ts#L39-L84)
- [tests/unit/api.test.ts232-252](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/api.test.ts#L232-L252)

---

## Error Handling and User Feedback

### Client-Side Errors

The form script at [src/pages/index.astro217-245](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/index.astro#L217-L245) implements:

- **Field-level errors**: Red border + message below field
- **Error summary**: Panel at top listing all issues
- **Scroll-to-error**: Auto-scrolls summary into view
- **Focus management**: Focuses first invalid field

### Server-Side Error Codes
Status CodeError CodeMeaningUser Action400`INVALID_JSON`Malformed request bodyDeveloper error404`INVITE_NOT_FOUND`Token invalidContact club409`INVITE_USED`Booking already createdView existing booking409`SLOT_FULL`Session at capacityChoose different date409`DUPLICATE_BOOKING`Already booked this dateNo action needed410`INVITE_EXPIRED`Token older than 14 daysContact club for new invite422`VALIDATION_ERROR`Invalid field dataFix highlighted fields
The client handles these at [src/pages/index.astro308-336](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/index.astro#L308-L336) by displaying user-friendly messages.

**Sources:**

- [src/pages/index.astro217-245](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/index.astro#L217-L245)
- [src/pages/index.astro308-336](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/index.astro#L308-L336)
- [src/api/booking.ts38-74](https://github.com/egac-web/egac/blob/0ba542fc/src/api/booking.ts#L38-L74)
- [tests/unit/api.test.ts254-282](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/api.test.ts#L254-L282)
- [tests/unit/api.test.ts444-501](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/api.test.ts#L444-L501)

---

## Progressive Enhancement

The form is designed to work without JavaScript:

1. **HTML-first**: Standard `<form method="POST" action="/api/enquiry">` at [src/pages/index.astro31](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/index.astro#L31-L31)
2. **Native validation**: `required` attributes on fields
3. **Server-side processing**: API accepts both JSON and form-encoded bodies

When JavaScript is available, the script at [src/pages/index.astro177-338](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/index.astro#L177-L338) intercepts submission and:

- Performs client-side validation before sending
- Shows inline error messages
- Displays a success screen without page reload
- Submits as JSON for better error structure

**Sources:**

- [src/pages/index.astro31](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/index.astro#L31-L31)
- [src/pages/index.astro247-336](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/index.astro#L247-L336)
- [src/api/enquiry.ts39-84](https://github.com/egac-web/egac/blob/0ba542fc/src/api/enquiry.ts#L39-L84)

---

## Related Subsystems

For detailed information on specific components:

- **Enquiry Form Details**: See [Enquiry Form](/egac-web/egac/3.1-enquiry-form) for field-by-field validation rules
- **Taster Booking Logic**: See [Taster Session Booking](/egac-web/egac/3.2-taster-session-booking) for capacity management and date selection
- **Academy Waitlist Details**: See [Academy Waitlist](/egac-web/egac/3.3-academy-waitlist) for season enrollment and invite process
- **API Contracts**: See [Public APIs](/egac-web/egac/5.1-public-apis) for complete request/response schemas
- **Email Templates**: See [Email Templates](/egac-web/egac/8.2-email-templates) for template variables and preview system
- **Age Group Rules**: See [Age Group System](/egac-web/egac/6.1-age-group-system) for athletics season date logic