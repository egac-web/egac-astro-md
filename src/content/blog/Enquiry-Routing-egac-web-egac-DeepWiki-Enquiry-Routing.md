# Enquiry Routing
Relevant source files
- [src/api/booking.ts](https://github.com/egac-web/egac/blob/0ba542fc/src/api/booking.ts)
- [src/api/enquiry.ts](https://github.com/egac-web/egac/blob/0ba542fc/src/api/enquiry.ts)
- [src/api/health.ts](https://github.com/egac-web/egac/blob/0ba542fc/src/api/health.ts)
- [src/lib/business/age.ts](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/business/age.ts)
- [src/lib/business/enquiry.ts](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/business/enquiry.ts)
- [src/lib/business/templates.ts](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/business/templates.ts)
- [src/lib/business/tokens.ts](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/business/tokens.ts)
- [src/pages/admin/academy.astro](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/academy.astro)
- [src/pages/admin/settings.astro](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/settings.astro)
- [tests/unit/api.test.ts](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/api.test.ts)
- [tests/unit/business.test.ts](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/business.test.ts)

## Purpose and Scope

Enquiry Routing is the business logic that determines how a submitted enquiry is processed after age group resolution. This page documents the routing decision logic that directs enquiries to either the **taster session booking path** or the **academy waitlist path** based on the athlete's age group configuration.

For information about how age groups are resolved from date of birth, see [Age Group System](/egac-web/egac/6.1-age-group-system). For details on session date generation and validation, see [Session Scheduling](/egac-web/egac/6.3-session-scheduling). For the complete API flow that uses this routing logic, see [Public APIs](/egac-web/egac/5.1-public-apis).

---

## Routing Decision Logic

The routing decision is made by the `routeEnquiry()` function, which examines the resolved age group's `booking_type` field and returns one of two paths.

**Core Function**

[src/lib/business/enquiry.ts62-72](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/business/enquiry.ts#L62-L72)

```
export function routeEnquiry(ageGroup: AgeGroupConfig | null): 'taster' | 'waitlist' {
  if (!ageGroup) return 'taster';
  return ageGroup.booking_type;
}
```

The function takes a single parameter: the age group resolved from the athlete's date of birth and session date (see [Age Group System](/egac-web/egac/6.1-age-group-system)). It returns a string literal type indicating which processing path to follow.

### Decision Tree

```
Yes

No

'taster'

'waitlist'

routeEnquiry(ageGroup)

ageGroup === null?

return 'taster'

ageGroup.booking_type

return 'taster'

return 'waitlist'
```

**Sources:**[src/lib/business/enquiry.ts62-72](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/business/enquiry.ts#L62-L72)

### Routing Outcomes
RouteOutcomeDatabase EntitiesEmail SentUser Experience`'taster'`Booking invite created`invites` record created`booking_invite` templateUser receives email with available session dates and booking link`'waitlist'`Academy waitlist entry created`academy_waitlist` record created`academy_waitlist` templateUser receives email confirming waitlist positionDefault (no age group)Treated as taster`invites` record created`booking_invite` templateUser receives booking invite despite no age group match
**Sources:**[src/api/enquiry.ts166-210](https://github.com/egac-web/egac/blob/0ba542fc/src/api/enquiry.ts#L166-L210)[src/lib/business/enquiry.ts62-72](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/business/enquiry.ts#L62-L72)

---

## Age Group Configuration and Routing

The routing decision is entirely determined by the `booking_type` field in the age group configuration stored in the D1 `age_groups` table. This field is set by administrators in the settings interface.

### Age Group Fields Relevant to Routing
FieldTypeRole in Routing`booking_type``'taster' | 'waitlist'`**Primary routing determinant** — directly maps to the route returned by `routeEnquiry()``age_min_aug31``number`Used by `resolveAgeGroup()` to match athlete's athletics age to this group`age_max_aug31``number`Used by `resolveAgeGroup()` to match athlete's athletics age to this group`active``number` (0 or 1)Inactive groups are excluded from resolution, preventing routing to them
**Sources:**[src/lib/business/age.ts90-105](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/business/age.ts#L90-L105)[src/pages/admin/settings.astro24-86](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/settings.astro#L24-L86)

### Typical Configuration Pattern

The EGAC platform typically configures:

- **U11 and younger**: `booking_type = 'waitlist'` (Academy programme)
- **U13 and older**: `booking_type = 'taster'` (Taster sessions)

This reflects the club's policy that younger athletes join structured academy seasons, while older athletes can try individual taster sessions before committing.

```
D1 age_groups Table

ag_u11
age_min: 9
age_max: 10
booking_type: 'waitlist'

ag_u13
age_min: 11
age_max: 12
booking_type: 'taster'

ag_u15
age_min: 13
age_max: 14
booking_type: 'taster'

Athlete age 9
on 31 Aug

Route: 'waitlist'

Athlete age 11
on 31 Aug

Route: 'taster'

Athlete age 13
on 31 Aug

Route: 'taster'
```

**Sources:**[src/lib/business/age.ts119-135](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/business/age.ts#L119-L135)[tests/unit/business.test.ts53-99](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/business.test.ts#L53-L99)

---

## Routing Implementation in API Handler

The enquiry routing logic is invoked within the `handleEnquiry` API handler after the enquiry is persisted to the database and the age group has been resolved.

### Processing Sequence

```
"Email Templates"
"Academy Repository"
"Invite Repository"
"routeEnquiry
(business/enquiry.ts)"
"resolveAgeGroup
(business/age.ts)"
"listAgeGroups
(repositories)"
"insertEnquiry
(repositories)"
"handleEnquiry
(src/api/enquiry.ts)"
Client
"Email Templates"
"Academy Repository"
"Invite Repository"
"routeEnquiry
(business/enquiry.ts)"
"resolveAgeGroup
(business/age.ts)"
"listAgeGroups
(repositories)"
"insertEnquiry
(repositories)"
"handleEnquiry
(src/api/enquiry.ts)"
Client
alt
[path === 'waitlist']
[path === 'taster']
POST /api/enquiry
Insert enquiry record
enquiry
Fetch all age groups from D1
allGroups[]
resolveAgeGroup(dob, today, allGroups)
ageGroup | null
routeEnquiry(ageGroup)
'taster' | 'waitlist'
addToWaitlist(enquiry.id)
invitation
sendAcademyWaitlist(enquiry, invitation)
createInviteForEnquiry(enquiry.id)
invite
sendBookingInvite(enquiry, invite, dates)
201 Created
```

**Sources:**[src/api/enquiry.ts86-223](https://github.com/egac-web/egac/blob/0ba542fc/src/api/enquiry.ts#L86-L223)

### Code Flow

The routing implementation follows this sequence in [src/api/enquiry.ts157-210](https://github.com/egac-web/egac/blob/0ba542fc/src/api/enquiry.ts#L157-L210):

1. **Resolve Age Group** (lines 158-164)

```
const allGroups = await listAgeGroups(env);
const ageGroup = resolveAgeGroup(enquiry.dob, new Date().toISOString().split('T')[0], allGroups);
if (ageGroup) {
  await updateEnquiry(enquiry.id, { age_group_id: ageGroup.id }, env);
}
```
2. **Determine Route** (line 166)

```
const path = routeEnquiry(ageGroup);
```
3. **Execute Taster Path** (lines 188-209)

- Create invite token in `invites` table
- Generate available session dates using `getNextNTuesdayDates()`
- Send `booking_invite` email with booking URL
- Mark invite as sent and log event
- Set `enquiry.processed = 1`
4. **Execute Waitlist Path** (lines 169-186)

- Create entry in `academy_waitlist` table
- Send `academy_waitlist` email confirming placement
- Mark waitlist invite as sent and log event
- Set `enquiry.processed = 1`

**Sources:**[src/api/enquiry.ts157-210](https://github.com/egac-web/egac/blob/0ba542fc/src/api/enquiry.ts#L157-L210)

---

## Taster Path Details

When `routeEnquiry()` returns `'taster'`, the system creates a booking invite and sends the user a list of available session dates.

### Key Operations
OperationRepository FunctionEffectCreate invite token`createInviteForEnquiry()`Inserts record into `invites` table with status `'pending'` and random secure tokenGenerate dates`getNextNTuesdayDates(weeksAhead)`Calculates next N Tuesdays from today for display in emailSend email`sendBookingInvite()`Dispatches `booking_invite` template with variables populatedMark sent`markInviteSent()`Updates invite status to `'sent'` and records `sent_at` timestampLog event`appendEnquiryEvent()`Adds `booking_invite_sent` event to enquiry's event log
**Sources:**[src/api/enquiry.ts188-209](https://github.com/egac-web/egac/blob/0ba542fc/src/api/enquiry.ts#L188-L209)[src/lib/repositories/invites.ts](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/repositories/invites.ts)

### Invite Token Lifecycle

```
createInviteForEnquiry()

markInviteSent()
after email delivery

markInviteAccepted()
when user books

Cron job
after 14 days

After 3 send attempts
by retry cron

pending

sent

accepted

expired

failed
```

**Sources:**[src/lib/repositories/invites.ts](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/repositories/invites.ts)[src/api/cron/expire-invites.ts](https://github.com/egac-web/egac/blob/0ba542fc/src/api/cron/expire-invites.ts)

---

## Waitlist Path Details

When `routeEnquiry()` returns `'waitlist'`, the system adds the athlete to the academy waitlist instead of creating a booking invite.

### Key Operations
OperationRepository FunctionEffectCreate waitlist entry`addToWaitlist()`Inserts record into `academy_waitlist` table with status `'waiting'` and secure tokenSend email`sendAcademyWaitlist()`Dispatches `academy_waitlist` template confirming placementMark sent`markWaitlistInviteSent()`Updates waitlist entry's `send_attempts` counterLog event`appendEnquiryEvent()`Adds `academy_waitlist_added` event to enquiry's event log
**Sources:**[src/api/enquiry.ts169-186](https://github.com/egac-web/egac/blob/0ba542fc/src/api/enquiry.ts#L169-L186)[src/lib/repositories/academy.ts](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/repositories/academy.ts)

### Waitlist Entry Lifecycle

```
addToWaitlist()

Admin sends
season invite

recordWaitlistResponse('yes')

recordWaitlistResponse('no')

Admin action

Admin action

waiting

invited

accepted

declined

withdrawn

ineligible
```

**Sources:**[src/lib/repositories/academy.ts](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/repositories/academy.ts)[src/pages/admin/academy.astro](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/academy.astro)

---

## Default Routing Behavior

When `resolveAgeGroup()` returns `null` (athlete's age doesn't match any configured age group), the routing logic defaults to the taster path. This prevents enquiries from being lost.

### Null Age Group Scenarios
ScenarioCauseRouting OutcomeNo DOB provided`enquiry.dob` is nullRoute: `'taster'` (default)DOB unparseableInvalid date format`resolveAgeGroup()` returns null → Route: `'taster'`Age out of rangeAthlete too young (<9) or too old (>19) for any configured group`resolveAgeGroup()` returns null → Route: `'taster'`All age groups inactiveAdmin has disabled all age groups`resolveAgeGroup()` returns null → Route: `'taster'`
**Sources:**[src/lib/business/enquiry.ts69-71](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/business/enquiry.ts#L69-L71)[src/lib/business/age.ts119-135](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/business/age.ts#L119-L135)

### Design Rationale

The default-to-taster behavior ensures:

1. **No enquiry loss**: Every valid submission receives a response email
2. **Human review**: Admin can manually assess edge cases via the bookings interface
3. **Fail-safe**: System remains functional even with misconfigured age groups

**Sources:**[tests/unit/business.test.ts258-272](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/business.test.ts#L258-L272)

---

## Routing and Email Templates

Each route triggers a different email template with distinct content and action links.

### Template Mapping

```
Route: 'taster'

Route: 'waitlist'

booking_invite
template

academy_waitlist
template

Variables:
• contact_name
• booking_url
• available_dates_list

Variables:
• contact_name
• response_url
• season info
```

**Sources:**[src/lib/email/templates.ts](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/email/templates.ts)[src/api/enquiry.ts169-209](https://github.com/egac-web/egac/blob/0ba542fc/src/api/enquiry.ts#L169-L209)

### Email Content Differences
AspectTaster Path EmailWaitlist Path EmailTemplate key`'booking_invite'``'academy_waitlist'`Primary CTA"Book your session" button with available dates"Respond to invitation" button (if applicable)Available datesLists next N Tuesdays dynamicallyNot applicableToken usageSingle-use booking invite token with 14-day expiryWaitlist response token (no expiry)Follow-up actionUser books via `/book/:token` pageUser waits for season invite via admin action
**Sources:**[src/lib/email/templates.ts](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/email/templates.ts)[src/pages/book/[token].astro](), [src/pages/academy/respond/[token].astro]()

---

## Testing the Routing Logic

The routing logic is covered by unit tests that verify correct path selection for various age group configurations.

### Test Coverage

[tests/unit/business.test.ts258-272](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/business.test.ts#L258-L272)

```
describe('routeEnquiry', () => {
  it('routes to waitlist for U11 Academy age group', () => {
    const ageGroup = makeAgeGroup({ booking_type: 'waitlist' });
    expect(routeEnquiry(ageGroup)).toBe('waitlist');
  });
 
  it('routes to taster for U13 age group', () => {
    const ageGroup = makeAgeGroup({ booking_type: 'taster' });
    expect(routeEnquiry(ageGroup)).toBe('taster');
  });
 
  it('defaults to taster when no age group resolved (no DOB)', () => {
    expect(routeEnquiry(null)).toBe('taster');
  });
});
```

### Integration Test Scenarios

[tests/unit/api.test.ts331-378](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/api.test.ts#L331-L378)

The API tests verify end-to-end routing behavior:

- **Taster path test**: Verifies invite creation and `booking_invite` email dispatch for U13 age group (lines 217-230)
- **Waitlist path test**: Verifies `addToWaitlist()` call and `academy_waitlist` email for U11 age group (lines 331-378)
- **Email failure resilience**: Confirms enquiry is saved even if email send fails (lines 380-394)

**Sources:**[tests/unit/business.test.ts258-272](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/business.test.ts#L258-L272)[tests/unit/api.test.ts217-394](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/api.test.ts#L217-L394)

---

## Administrative Control

Administrators control routing behavior by configuring age groups in the settings interface. Changing a group's `booking_type` field immediately affects all future enquiries resolved to that group.

### Configuration Interface

[src/pages/admin/settings.astro230-291](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/settings.astro#L230-L291)

The settings page displays all age groups with inline editing of:

- `booking_type` dropdown: `'taster'` or `'waitlist'`
- Age range (`age_min_aug31` and `age_max_aug31`)
- Session configuration (days, times, capacity)
- Active status toggle

Changes take effect immediately upon form submission. No restart or cache clearing is required.

**Sources:**[src/pages/admin/settings.astro24-86](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/settings.astro#L24-L86)

---

## Error Handling and Logging

The routing logic itself is pure and cannot fail (it always returns a valid route). However, the downstream effects (creating invites/waitlist entries, sending emails) can fail.

### Failure Handling Strategy

```
Success

Failure

Success

Failure

Route determined

Try create invite/waitlist

Try send email

Log error to console
(JSON structured)

Set enquiry.processed = 1

Return 201 to client
```

**Sources:**[src/api/enquiry.ts168-220](https://github.com/egac-web/egac/blob/0ba542fc/src/api/enquiry.ts#L168-L220)

### Error Log Format

[src/api/enquiry.ts177-184](https://github.com/egac-web/egac/blob/0ba542fc/src/api/enquiry.ts#L177-L184)

```
console.error(
  JSON.stringify({
    level: 'error',
    event: 'academy_email_failed',
    enquiry_id: enquiry.id,
    error: result.error,
  })
);
```

Email failures are logged but **never block the enquiry submission**. The enquiry is always saved with `processed = 1`, and the cron retry jobs attempt redelivery.

**Sources:**[src/api/enquiry.ts168-220](https://github.com/egac-web/egac/blob/0ba542fc/src/api/enquiry.ts#L168-L220)[src/api/cron/retry-invites.ts](https://github.com/egac-web/egac/blob/0ba542fc/src/api/cron/retry-invites.ts)

---

## Summary

Enquiry routing is a lightweight decision layer that maps age group configuration to processing paths. The entire routing logic is encapsulated in a single pure function (`routeEnquiry()`) that examines the `booking_type` field from the resolved age group.

### Key Points

1. **Single source of truth**: The D1 `age_groups.booking_type` field determines all routing behavior
2. **Two paths only**: Taster (booking invites) or Waitlist (academy placement)
3. **Fail-safe default**: Unknown age groups default to taster path
4. **Immediate effect**: Admin changes to age group configuration apply to next enquiry
5. **Resilient**: Email failures don't prevent enquiries from being routed and saved

**Sources:**[src/lib/business/enquiry.ts62-72](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/business/enquiry.ts#L62-L72)[src/api/enquiry.ts157-210](https://github.com/egac-web/egac/blob/0ba542fc/src/api/enquiry.ts#L157-L210)