# Enquiry Form
Relevant source files
- [src/api/booking.ts](https://github.com/egac-web/egac/blob/0ba542fc/src/api/booking.ts)
- [src/api/enquiry.ts](https://github.com/egac-web/egac/blob/0ba542fc/src/api/enquiry.ts)
- [src/api/health.ts](https://github.com/egac-web/egac/blob/0ba542fc/src/api/health.ts)
- [src/lib/business/age.ts](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/business/age.ts)
- [src/lib/business/enquiry.ts](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/business/enquiry.ts)
- [src/lib/business/templates.ts](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/business/templates.ts)
- [src/lib/business/tokens.ts](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/business/tokens.ts)
- [src/pages/index.astro](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/index.astro)
- [tests/unit/api.test.ts](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/api.test.ts)
- [tests/unit/business.test.ts](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/business.test.ts)

The enquiry form is the primary public entry point for registering interest in East Grinstead Athletics Club. It collects athlete information, determines age group eligibility, and routes users to the appropriate next step—either taster session booking invites or academy waitlist enrollment. This page documents the form UI, validation, submission flow, and routing logic.

For information about the taster session booking process that follows successful enquiry submission, see [Taster Session Booking](/egac-web/egac/3.2-taster-session-booking). For details on academy waitlist enrollment, see [Academy Waitlist](/egac-web/egac/3.3-academy-waitlist). For the complete API specification, see [Public APIs](/egac-web/egac/5.1-public-apis).

---

## Form Structure

The enquiry form is implemented as a server-rendered Astro page with progressive enhancement via client-side JavaScript. The form supports two modes: self-enquiry (athlete registering themselves) and on-behalf enquiry (parent/guardian registering a child).

**Sources:**[src/pages/index.astro1-175](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/index.astro#L1-L175)

### Form Fields
Field NameHTML NameTypeRequiredDescriptionEnquiry Mode`enquiry_for`Radio (`self`/`other`)YesDetermines if user is registering themselves or someone elseFull Name`name`TextYesContact person's name (enquirer)Email`email`EmailYesContact email addressPhone`phone`TelNoOptional contact phone numberAthlete Name`athlete_name`TextConditionalRequired only when `enquiry_for=other`Date of Birth`dob`DateYesAthlete's date of birth (used for age group resolution)
**Sources:**[src/pages/index.astro31-138](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/index.astro#L31-L138)

### Form Modes

The form dynamically adapts based on the selected enquiry mode:

```
Form loads with enquiry_for='self' selected

Self Mode
(enquiry_for='self')

Other Mode
(enquiry_for='other')

Fields visible:
• name (Your full name)
• email
• phone
• dob (Your date of birth)

Fields visible:
• name (Your full name)
• email
• phone
• athlete_name (Athlete's name)
• dob (Athlete's date of birth)

Maps to API:
contact=enquirer
athlete=enquirer

Maps to API:
contact=enquirer
athlete=different person
```

The mode switching is handled by the `syncEnquiryMode()` function, which toggles field visibility and updates field labels dynamically.

**Sources:**[src/pages/index.astro193-210](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/index.astro#L193-L210)

---

## Client-Side Validation

The form implements comprehensive client-side validation before submission to provide immediate user feedback and reduce server load.

### Validation Rules
FieldValidation RuleError Message`name`Must not be empty"Full name is required"`email`Must not be empty"Email address is required"`email`Must match email regex `/^[^\s@]+@[^\s@]+\.[^\s@]+$/`"Please enter a valid email address"`athlete_name`Must not be empty when `enquiry_for=other`"Athlete's name is required"`dob`Must not be empty"Date of birth is required"
**Sources:**[src/pages/index.astro259-283](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/index.astro#L259-L283)

### Error Display System

The form uses a multi-layered error display approach:

1. **Error Summary Panel** - Displays all validation errors in a list at the top of the form
2. **Inline Field Errors** - Individual error messages below each field
3. **Visual Indicators** - Red borders on invalid fields

```
No

Yes

User clicks 'Send enquiry'

clientValidation()

All fields
valid?

showFieldError()
for each invalid field

showSummaryErrors()
Display error-summary panel

Focus first invalid field

Wait for user correction

clearErrors()

Proceed to API submission
```

The error system is implemented using DOM manipulation functions that add/remove CSS classes and update `aria-describedby` attributes for accessibility.

**Sources:**[src/pages/index.astro217-245](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/index.astro#L217-L245)[src/pages/index.astro229-237](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/index.astro#L229-L237)

---

## Submission Flow

### Request Construction

When the form passes client-side validation, it constructs a JSON payload that supports both enquiry modes:

```
"POST /api/enquiry"
"Form Script"
"enquiry-form"
User
"POST /api/enquiry"
"Form Script"
"enquiry-form"
User
alt
[Response OK (201) or 409]
[Response 422]
[Other error]
alt
[Validation fails]
[Validation passes]
Click "Send enquiry"
submit event
preventDefault()
clientValidation()
Show errors
Disable button
Set text: "Sending…"
Build JSON payload
fetch() with JSON body
{message: "..."}
Hide form-section
Show confirmation
Success message
{error: "..."}
showFieldError()
Validation error
Error response
showSummaryErrors()
Generic error
```

The payload includes both new field names (prefixed with `enquirer_` and `athlete_`) and legacy field names for backward compatibility:

**Sources:**[src/pages/index.astro289-306](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/index.astro#L289-L306)

### Payload Structure

```
{
  enquiry_for: 'self' | 'other',
  enquirer_name: string,
  enquirer_email: string,
  enquirer_phone?: string,
  athlete_name: string,
  athlete_dob: string,
  // Legacy keys (retained during API transition):
  name: string,
  email: string,
  phone?: string,
  dob?: string
}
```

**Sources:**[src/api/enquiry.ts23-37](https://github.com/egac-web/egac/blob/0ba542fc/src/api/enquiry.ts#L23-L37)

---

## Server-Side Processing

### API Handler: `handleEnquiry`

The `POST /api/enquiry` endpoint processes form submissions through a multi-stage pipeline:

```
No

Yes

taster

waitlist

POST /api/enquiry

parsePublicEnquiryBody()
Accept JSON or form-urlencoded

Normalize payload
Map enquirer_* and athlete_* fields

validateEnquiryData()
Server-side validation

Valid?

Return 422
VALIDATION_ERROR

insertEnquiry()
Save to D1.enquiries

listAgeGroups(env)
Fetch from D1.age_groups

resolveAgeGroup(dob, date, groups)
Determine age on Aug 31

updateEnquiry()
Store age_group_id

routeEnquiry(ageGroup)

booking_type?

createInviteForEnquiry()
Generate secure token

addToWaitlist()
Add to academy_waitlist

sendBookingInvite()
Email with booking link

sendAcademyWaitlist()
Email waitlist confirmation

markInviteSent()

markWaitlistInviteSent()

appendEnquiryEvent()
'booking_invite_sent'

appendEnquiryEvent()
'academy_waitlist_added'

Return 201
{message: 'Enquiry received'}

End
```

**Sources:**[src/api/enquiry.ts86-223](https://github.com/egac-web/egac/blob/0ba542fc/src/api/enquiry.ts#L86-L223)

### Content-Type Support

The handler accepts multiple content types to support both JavaScript and plain HTML form submissions:

- `application/json` - Standard JSON POST (used by fetch)
- `application/x-www-form-urlencoded` - Plain HTML form submission
- `multipart/form-data` - Form submission with file upload support

**Sources:**[src/api/enquiry.ts39-84](https://github.com/egac-web/egac/blob/0ba542fc/src/api/enquiry.ts#L39-L84)

---

## Age Group Resolution

Age group determination follows UK Athletics rules, where an athlete's age group is based on their age on **31 August** of the athletics season the enquiry falls in, not their current age.

### Age Group Resolution Logic

```
Yes

No

Enquiry with DOB

Get current date

athleticsSeasonEndYear(today)
Determine season end year

Calculate cutoff date:
31 August [season_end_year]

ageForAthletics(dob, today)
Age on cutoff date

Fetch all active age groups
from D1.age_groups

Filter:
WHERE active = 1
ORDER BY sort_order ASC

Find first group where:
age >= age_min_aug31
AND age <= age_max_aug31

Match
found?

Return AgeGroupConfig

Return null
(no matching group)

End
```

**Sources:**[src/lib/business/age.ts119-135](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/business/age.ts#L119-L135)[src/lib/business/age.ts76-83](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/business/age.ts#L76-L83)

### Example Age Calculation

For an enquiry submitted on **14 October 2026**:
DOBAge on 31 Aug 2027Age Groupbooking_type15 Sep 201412U13taster1 Sep 201214U15taster1 Sep 201610U11 Academywaitlist31 Aug 201313U15taster
The season calculation logic:

- Enquiry in **September or later** → season ends **next August**
- Enquiry in **August or earlier** → season ends **same August**

**Sources:**[src/lib/business/age.ts38-44](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/business/age.ts#L38-L44)[tests/unit/business.test.ts105-125](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/business.test.ts#L105-L125)

---

## Enquiry Routing

After age group resolution, the `routeEnquiry()` function determines the athlete's path through the system:

### Routing Decision Tree

```
null

AgeGroupConfig

'taster'

'waitlist'

resolveAgeGroup() returns AgeGroupConfig or null

AgeGroup
found?

Return 'taster'
(default route)

ageGroup.booking_type

Return 'taster'
• Create invite
• Send booking link
• Email available dates

Return 'waitlist'
• Add to academy_waitlist
• Send waitlist confirmation
• No immediate booking
```

**Sources:**[src/lib/business/enquiry.ts69-72](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/business/enquiry.ts#L69-L72)

### Taster Path

Athletes aged **11+ on Aug 31** (U13 and above):

1. `createInviteForEnquiry()` generates a secure token
2. `sendBookingInvite()` sends email with booking URL
3. Email includes next 8 Tuesday dates with capacity information
4. Invite expires after 14 days
5. User books specific taster session via invite token

**Sources:**[src/api/enquiry.ts189-209](https://github.com/egac-web/egac/blob/0ba542fc/src/api/enquiry.ts#L189-L209)

### Waitlist Path

Athletes aged **9-10 on Aug 31** (U11 Academy):

1. `addToWaitlist()` creates academy_waitlist entry
2. `sendAcademyWaitlist()` sends confirmation email
3. No immediate booking available
4. Admin manages waitlist manually
5. Invites sent when season opens or capacity available

**Sources:**[src/api/enquiry.ts169-187](https://github.com/egac-web/egac/blob/0ba542fc/src/api/enquiry.ts#L169-L187)

---

## Validation System

### Server-Side Validation: `validateEnquiryData`

The server performs validation independent of client-side checks to ensure data integrity:
ValidationRuleError Message`contact_name`Must exist, be string, not empty after trim"contact_name is required"`contact_email`Must exist, be string, not empty after trim"contact_email is required"`contact_email` formatMust match `/^[^\s@]+@[^\s@]+\.[^\s@]+$/`"contact_email must be a valid email address"`dob` formatIf provided, must parse as valid Date"dob must be a valid date in YYYY-MM-DD format"`dob` rangeAge must be 4-100 years"dob does not represent a plausible age (must be between 4 and 100 years ago)"`phone` typeIf provided, must be string"phone must be a string"
**Sources:**[src/lib/business/enquiry.ts20-60](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/business/enquiry.ts#L20-L60)

### Validation Flow

```
"Field Normalizer"
"validateEnquiryData"
"parsePublicEnquiryBody"
"handleEnquiry"
"Field Normalizer"
"validateEnquiryData"
"parsePublicEnquiryBody"
"handleEnquiry"
alt
[JSON]
[Form-encoded]
alt
[Validation fails]
[Validation passes]
request
Detect content-type
parseBody<T>()
formData.get()
PublicEnquiryBody
Extract enquirer_* fields
Map to contact_* fields
Normalized data
{ contact_name, contact_email, dob, ... }
Check required fields
Validate email format
Validate dob range
throw Error(message)
catch → return 422
return (no error)
Continue processing
```

**Sources:**[src/api/enquiry.ts113-125](https://github.com/egac-web/egac/blob/0ba542fc/src/api/enquiry.ts#L113-L125)[src/lib/business/enquiry.ts20-60](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/business/enquiry.ts#L20-L60)

---

## Response Handling

### Success Response (201 Created)

The endpoint always returns `201 Created` with a generic success message, even if email sending fails. This ensures enquiries are never lost due to transient email delivery issues:

```
{
  "ok": true,
  "message": "Enquiry received"
}
```

**Sources:**[src/api/enquiry.ts222](https://github.com/egac-web/egac/blob/0ba542fc/src/api/enquiry.ts#L222-L222)

### Error Responses
StatusCodeCondition400`INVALID_JSON`Request body cannot be parsed405-Method not POST422`VALIDATION_ERROR`Field validation failed
**Sources:**[src/api/enquiry.ts88-91](https://github.com/egac-web/egac/blob/0ba542fc/src/api/enquiry.ts#L88-L91)[src/api/enquiry.ts104-106](https://github.com/egac-web/egac/blob/0ba542fc/src/api/enquiry.ts#L104-L106)[src/api/enquiry.ts122-124](https://github.com/egac-web/egac/blob/0ba542fc/src/api/enquiry.ts#L122-L124)

### Client-Side Response Handling

The form JavaScript interprets the response and displays appropriate confirmation messages:

```
201 OK

409 Conflict

422 Validation

Other error

Yes

No

Receive API response

HTTP
status?

Parse response.message

showFieldError(message)

Show generic error

Message contains
'waitlist' or
'academy'?

Display:
'We've added your child to
the Academy waitlist...'

Display:
'Thanks — check your email
for a booking link...'

Hide form-section

Show confirmation panel

Re-enable submit button

End
```

**Sources:**[src/pages/index.astro308-336](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/index.astro#L308-L336)

---

## Email Sending (Fire-and-Forget)

Email sending is implemented as a non-blocking operation that never causes enquiry rejection:

```
success: true

success: false

Enquiry saved to D1

try { sendEmail() }

Email
success?

markInviteSent()
appendEnquiryEvent('sent')

console.error()
Log email_send_failed

catch (sendError)

console.error()
Log post_processing_failed

Always return 201

Retry cron job will
attempt resend later
```

Failed email sends are logged and queued for retry by the `handleRetryInvites` cron job, which runs every 30 minutes.

**Sources:**[src/api/enquiry.ts168-220](https://github.com/egac-web/egac/blob/0ba542fc/src/api/enquiry.ts#L168-L220)

---

## Testing

### Unit Tests

The enquiry handler has comprehensive test coverage:
Test CaseDescriptionStatus CodeValid non-academy enquiryStandard taster path submission201Form-urlencoded bodySupports HTML form POST201Missing emailRequired field validation422Invalid JSONMalformed request body400GET requestMethod not allowed405Repeat emailIdempotent submissions allowed201On-behalf enquiryMaps athlete_name correctly201Academy-age enquiryRoutes to waitlist201Email send failureStill returns 201 (non-blocking)201OPTIONS preflightCORS support204
**Sources:**[tests/unit/api.test.ts199-401](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/api.test.ts#L199-L401)

### Mock Strategy

Tests use `vitest` mocks for all repository and email functions:

```
vi.mock('../../src/lib/repositories/enquiries.js', () => ({
  insertEnquiry: vi.fn(),
  getEnquiryByEmail: vi.fn(),
  appendEnquiryEvent: vi.fn(),
  updateEnquiry: vi.fn(),
}));
```

This allows testing API logic in isolation without D1 database or Resend API dependencies.

**Sources:**[tests/unit/api.test.ts12-18](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/api.test.ts#L12-L18)

---

## Key Functions Reference
FunctionLocationPurpose`handleEnquiry`[src/api/enquiry.ts86-223](https://github.com/egac-web/egac/blob/0ba542fc/src/api/enquiry.ts#L86-L223)Main API handler for POST /api/enquiry`parsePublicEnquiryBody`[src/api/enquiry.ts39-84](https://github.com/egac-web/egac/blob/0ba542fc/src/api/enquiry.ts#L39-L84)Parse JSON or form-encoded request body`validateEnquiryData`[src/lib/business/enquiry.ts20-60](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/business/enquiry.ts#L20-L60)Server-side validation logic`resolveAgeGroup`[src/lib/business/age.ts119-135](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/business/age.ts#L119-L135)Determine age group from DOB and date`ageForAthletics`[src/lib/business/age.ts76-83](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/business/age.ts#L76-L83)Calculate age on Aug 31 cutoff`routeEnquiry`[src/lib/business/enquiry.ts69-72](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/business/enquiry.ts#L69-L72)Decide taster vs waitlist path`insertEnquiry`Referenced via repositorySave enquiry to D1.enquiries`createInviteForEnquiry`Referenced via repositoryGenerate invite token for taster bookings`addToWaitlist`Referenced via repositoryAdd to academy_waitlist table`sendBookingInvite`Referenced via email templatesSend taster booking invitation email`sendAcademyWaitlist`Referenced via email templatesSend academy waitlist confirmation email
**Sources:**[src/api/enquiry.ts1-224](https://github.com/egac-web/egac/blob/0ba542fc/src/api/enquiry.ts#L1-L224)[src/lib/business/enquiry.ts1-87](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/business/enquiry.ts#L1-L87)[src/lib/business/age.ts1-271](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/business/age.ts#L1-L271)