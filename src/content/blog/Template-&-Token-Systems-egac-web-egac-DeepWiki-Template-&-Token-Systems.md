---
title: "Templates and Token Systems"
description: "This page documents the template rendering and secure token generation used across the platform"
pubDate: 2026-03-18
---
# Template & Token Systems
Relevant source files
- [src/lib/business/age.ts](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/business/age.ts)
- [src/lib/business/enquiry.ts](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/business/enquiry.ts)
- [src/lib/business/templates.ts](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/business/templates.ts)
- [src/lib/business/tokens.ts](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/business/tokens.ts)
- [src/lib/email/client.ts](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/email/client.ts)
- [src/lib/email/preview.ts](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/email/preview.ts)
- [src/lib/email/templates.ts](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/email/templates.ts)
- [tests/unit/business.test.ts](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/business.test.ts)
- [tests/unit/email.test.ts](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/email.test.ts)

This document covers the template rendering engine and secure token generation utilities used throughout the EGAC platform. The template system powers all email communications by replacing `{{variable}}` placeholders with contextual data. The token system provides cryptographically secure identifiers for booking invites, academy waitlist entries, and membership OTPs with configurable expiry rules.

For information about how templates are stored and retrieved from the database, see [Configuration & Templates Repositories](/egac-web/egac/7.4-configuration-and-templates-repositories). For details on how templates are integrated into the email sending workflow, see [Email System](/egac-web/egac/8-email-system).

---

## Template Rendering System

The template engine is a pure function module that performs string substitution with strict validation rules. All email templates use the `{{variable_name}}` syntax for placeholders.

### Variable Substitution Rules

The `renderTemplate()` function implements three core rules:
RuleBehaviorRationale**Missing variables throw**If template contains `{{booking_url}}` but `booking_url` is not in the variable map, an error is thrownPrevents emails with unfilled placeholders like "Book here: {{booking_url}}" from being sent**Extra variables ignored**If variable map contains `extra_field` but template doesn't use it, no error occursAllows reusing variable maps across multiple templates without duplication**Case-sensitive matching**`{{contact_name}}` and `{{Contact_Name}}` are distinct tokensEnforces consistent naming conventions
**Sources:**[src/lib/business/templates.ts21-43](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/business/templates.ts#L21-L43)

### Core Template Functions

```
Usage Contexts

src/lib/business/templates.ts

renderTemplate(template, vars)
Returns: string
Throws on missing vars

validateTemplateVariables(html, allowedVars)
Returns: string[] (unknown vars)

extractTemplateVariables(template)
Returns: string[] (all vars)

buildSampleVars(overrides)
Returns: Record<string, string>

Email Sending
src/lib/email/templates.ts

Admin Preview
src/lib/email/preview.ts

Template Editor Validation
```

**Sources:**[src/lib/business/templates.ts1-112](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/business/templates.ts#L1-L112)[src/lib/email/templates.ts1-286](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/email/templates.ts#L1-L286)[src/lib/email/preview.ts1-104](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/email/preview.ts#L1-L104)

### Template Variable Registry

All email templates across the platform use a standardized set of variables. The `SAMPLE_TEMPLATE_VARS` constant provides default preview values:
VariablePurposeExample ValueUsed In`contact_name`Primary recipient name`"Jane Smith"`All templates`parent_name`Parent/guardian name`"Jane Smith"`Academy emails`child_name`Athlete name`"Emily Smith"`Academy emails`booking_url`Invite link with token`"/book/abc123def456"``booking_invite``available_dates_list`Formatted date list`"• Tuesday 14 April 2026\n• Tuesday 21 April 2026"``booking_invite``session_date`Confirmed session date`"Tuesday 14 April 2026"``booking_confirmation`, `booking_reminder``session_time`Session start time`"18:30"``booking_confirmation`, `booking_reminder``age_group_label`Age group code`"U13"`Booking emails`venue`Session location`"East Grinstead Leisure Centre"`Booking emails`add_to_calendar_url`Google Calendar link`"https://calendar.google.com/..."``booking_confirmation``response_url`Academy accept/decline link`"/academy/respond/xyz789"``academy_waitlist``membership_url`Membership signup link`"/join?token=mem123"``membership_invite`
**Sources:**[src/lib/business/templates.ts86-112](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/business/templates.ts#L86-L112)

### Template Rendering Example

```
// Template content from D1
const template = "Hi {{contact_name}}, book at {{booking_url}}.";
 
// Variable map built from enquiry and invite
const vars = {
  contact_name: "Jane Smith",
  booking_url: "https://egac.pages.dev/book/abc123def456"
};
 
// Rendering (strict validation)
const html = renderTemplate(template, vars);
// Result: "Hi Jane Smith, book at https://egac.pages.dev/book/abc123def456."
 
// Missing variable causes error
renderTemplate("Hi {{contact_name}}, see {{missing_var}}", vars);
// Throws: "Template contains unreplaced variables: {{missing_var}}"
```

**Sources:**[tests/unit/business.test.ts456-486](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/business.test.ts#L456-L486)[tests/unit/email.test.ts228-310](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/email.test.ts#L228-L310)

---

## Token Generation & Validation

The token system provides cryptographically secure random identifiers for time-limited operations. All tokens are hex-encoded random bytes stored in D1 and validated server-side.

### Token Generation

```
Usage Sites

Token Generation Flow

generateSecureToken(byteLength)

crypto.getRandomValues()

Hex encoding

createInviteForEnquiry()
Default: 24 bytes → 48 chars

addToWaitlist()
Default: 24 bytes → 48 chars

createMembershipOtp()
Default: 24 bytes → 48 chars
```

**Sources:**[src/lib/business/tokens.ts18-20](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/business/tokens.ts#L18-L20)[src/lib/db/client.ts](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/db/client.ts) (via import)

The `generateSecureToken()` function accepts a byte length parameter. The resulting hex string is twice as long:

- `generateSecureToken(24)` → 48-character hex string (default)
- `generateSecureToken(16)` → 32-character hex string

**Sources:**[tests/unit/business.test.ts425-428](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/business.test.ts#L425-L428)

### Token Expiry System

Token expiry is calculated based on creation timestamp and a configurable TTL:

```
Constants

Convenience Wrappers

Core Expiry Function

isTokenExpired(createdAt, ttlDays, now)
Returns: boolean

isInviteExpired(createdAt, now)
TTL = 14 days

isMembershipOtpExpired(createdAt, now)
TTL = 7 days

tokenAgeInHours(createdAt, now)
Returns: number

INVITE_TTL_DAYS = 14

MEMBERSHIP_OTP_TTL_DAYS = 7
```

**Sources:**[src/lib/business/tokens.ts22-71](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/business/tokens.ts#L22-L71)

### TTL Constants and Their Usage
ConstantValueToken TypePurpose`INVITE_TTL_DAYS`14 daysBooking invitesUsers have 2 weeks to select a taster session date after submitting an enquiry`MEMBERSHIP_OTP_TTL_DAYS`7 daysMembership invitesUsers have 1 week to complete club membership after attending a taster session
The expiry logic follows these rules:

1. **Unparseable timestamps treated as expired** — If `createdAt` is malformed, `isTokenExpired()` returns `true`
2. **Threshold is exclusive** — A token created exactly 14 days ago is considered expired
3. **Default `now` parameter** — All functions accept an optional `now` parameter for testing; defaults to `new Date()`

**Sources:**[src/lib/business/tokens.ts27-59](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/business/tokens.ts#L27-L59)[tests/unit/business.test.ts430-448](https://github.com/egac-web/egac/blob/0ba542fc/tests/unit/business.test.ts#L430-L448)

### Token Age for Retry Logic

The `tokenAgeInHours()` function calculates token age in hours rather than a boolean expired/not-expired check. This supports the retry cron job, which only retries invites older than 1 hour to avoid race conditions:

```
// Cron job: only retry invites created >1 hour ago
const invitesForRetry = pendingInvites.filter(inv => 
  tokenAgeInHours(inv.created_at) > 1
);
```

**Sources:**[src/lib/business/tokens.ts62-71](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/business/tokens.ts#L62-L71)

---

## Email Template Pipeline

The email template pipeline connects template rendering to email delivery. Each email type has a dedicated sender function that orchestrates the full flow.

### Template-to-Email Flow

```
"Resend API"
"sendEmailWithRetry()"
"renderTemplate()"
"D1 email_templates"
"getEmailTemplateByKey()"
"sendBookingInvite()"
"API Handler"
"Resend API"
"sendEmailWithRetry()"
"renderTemplate()"
"D1 email_templates"
"getEmailTemplateByKey()"
"sendBookingInvite()"
"API Handler"
contact_name, booking_url,
available_dates_list
enquiry, invite, availableDates, env
key='booking_invite'
SELECT * WHERE key=? AND active=1
template row
EmailTemplate
Build variable map
template.html, vars
rendered HTML
template.text_content, vars
rendered text
to, subject, html, text
POST /emails
{id: "email_xyz"}
{success: true, id: "..."}
SendEmailResult
```

**Sources:**[src/lib/email/templates.ts37-83](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/email/templates.ts#L37-L83)

### Email Sender Functions

The platform implements five email sender functions, each following the same pattern:

```
Dependencies

Email Senders - src/lib/email/templates.ts

sendBookingInvite()
Line 37-83
Template: booking_invite

sendBookingConfirmation()
Line 90-139
Template: booking_confirmation

sendAcademyWaitlist()
Line 146-187
Template: academy_waitlist

sendMembershipInvite()
Line 194-233
Template: membership_invite

sendBookingReminder()
Line 239-285
Template: booking_reminder

getEmailTemplateByKey()

renderTemplate()

sendEmailWithRetry()
```

**Sources:**[src/lib/email/templates.ts1-286](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/email/templates.ts#L1-L286)

### Standard Email Sender Pattern

Each sender function follows this six-step pattern:

1. **Fetch template from D1** — Call `getEmailTemplateByKey(key, env)` with the template key
2. **Check template exists** — Return `{success: false}` if template not found or inactive
3. **Build variable map** — Construct variables from function parameters (enquiry, booking, etc.)
4. **Render HTML** — Call `renderTemplate(template.html, vars)` inside try-catch
5. **Render text** — Conditionally render `template.text_content` if present
6. **Send with retry** — Call `sendEmailWithRetry()` and return result

**Error handling:** All senders catch rendering errors and return `{success: false, error: message}` rather than throwing. This ensures a malformed template never crashes the enquiry or booking flow.

**Sources:**[src/lib/email/templates.ts37-83](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/email/templates.ts#L37-L83)[src/lib/email/templates.ts90-139](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/email/templates.ts#L90-L139)

### Variable Construction by Email Type

Different email types require different variable sets:

**booking_invite** ([src/lib/email/templates.ts37-83](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/email/templates.ts#L37-L83)):

```
{
  contact_name: enquiry.name || 'there',
  booking_url: `${siteUrl}/book/${invite.token}`,
  available_dates_list: availableDates.map(d => `• ${formatDateForDisplay(d)}`).join('\n')
}
```

**booking_confirmation** ([src/lib/email/templates.ts90-139](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/email/templates.ts#L90-L139)):

```
{
  contact_name: enquiry.name || 'there',
  session_date: formatDateForDisplay(booking.session_date),
  session_time: booking.session_time ?? '',
  age_group_label: booking.age_group_label ?? '',
  venue: venueName,
  add_to_calendar_url: calendarUrl
}
```

**academy_waitlist** ([src/lib/email/templates.ts146-187](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/email/templates.ts#L146-L187)):

```
{
  parent_name: enquiry.name || 'there',
  child_name: enquiry.name || 'your child',
  response_url: `${siteUrl}/academy/respond/${invitation.token}`
}
```

**membership_invite** ([src/lib/email/templates.ts194-233](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/email/templates.ts#L194-L233)):

```
{
  contact_name: enquiry.name || 'there',
  membership_url: `${siteUrl}/join?token=${membershipToken}`
}
```

**booking_reminder** ([src/lib/email/templates.ts239-285](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/email/templates.ts#L239-L285)):

```
{
  contact_name: enquiry.name || 'there',
  session_date: formatDateForDisplay(booking.session_date),
  session_time: booking.session_time ?? '',
  age_group_label: booking.age_group_label ?? '',
  venue: venueName
}
```

**Sources:**[src/lib/email/templates.ts37-285](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/email/templates.ts#L37-L285)

### Site URL Resolution

All email senders use `getPublicSiteUrl(env)` to construct absolute URLs for links:
EnvironmentLogicResultExplicit `SITE_URL` setUse `env.SITE_URL` trimmed and without trailing slashValue from environment variableProduction without `SITE_URL`Return hardcoded production URL`https://egac-cl9.pages.dev`Other environmentsReturn default staging URL`https://egac.pages.dev`
This ensures emails always contain working links regardless of deployment environment.

**Sources:**[src/lib/email/templates.ts22-30](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/email/templates.ts#L22-L30)

---

## Preview System

The preview system renders templates with sample data for admin review without sending emails. It gracefully handles missing variables by displaying placeholder text rather than throwing errors.

### Preview Functions

```
Return Type

Business Logic

Repository Layer

src/lib/email/preview.ts

generatePreview(templateKey, sampleVarOverrides, env)

generatePreviewById(templateId, sampleVarOverrides, env)

getEmailTemplateByKey(key, env)

getEmailTemplateById(id, env)

buildSampleVars(overrides)

renderTemplate(template, vars)

PreviewResult
{html, subject, templateKey, templateName}
```

**Sources:**[src/lib/email/preview.ts1-104](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/email/preview.ts#L1-L104)

### Graceful Error Handling in Preview Mode

Unlike production email sending (where missing variables throw errors), the preview system replaces unreplaced variables with visible placeholders:
ContextMissing Variable BehaviorRationaleProduction (`sendBookingInvite`, etc.)Throw error, return `{success: false}`Prevent malformed emails from being sentPreview (`generatePreview`, `generatePreviewById`)Replace `{{var}}` with `[var]`Allow admin to see template structure even with incomplete variable set
**Implementation:**

```
// Production rendering (strict)
const html = renderTemplate(template.html, vars);
// Throws if vars are missing
 
// Preview rendering (permissive)
try {
  html = renderTemplate(template.html, vars);
} catch {
  // Replace unresolved tokens with [placeholder] format
  html = template.html.replace(/\{\{([^}]+)\}\}/g, (_, key) => `[${key}]`);
}
```

**Sources:**[src/lib/email/preview.ts25-62](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/email/preview.ts#L25-L62)

### Sample Variable Overrides

Admins can provide custom sample values for specific variables while preserving defaults for others:

```
// Use all defaults except for contact_name
const preview = await generatePreview('booking_invite', {
  contact_name: 'Test User'
}, env);
 
// Merged result combines override with SAMPLE_TEMPLATE_VARS defaults
// contact_name: "Test User"
// booking_url: "https://egac-cl9.pages.dev/book/abc123def456"
// available_dates_list: "• Tuesday 14 April 2026\n• Tuesday 21 April 2026\n• Tuesday 28 April 2026"
```

**Sources:**[src/lib/email/preview.ts25-62](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/email/preview.ts#L25-L62)[src/lib/business/templates.ts109-112](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/business/templates.ts#L109-L112)

---

## Integration Points

### Template System Usage
ModuleFunctionPurpose`src/lib/email/templates.ts`All sender functionsProduction email rendering with strict validation`src/lib/email/preview.ts``generatePreview()`, `generatePreviewById()`Admin preview with graceful error handlingAdmin template editor`validateTemplateVariables()`, `extractTemplateVariables()`Real-time validation and variable reference display
**Sources:**[src/lib/email/templates.ts1-286](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/email/templates.ts#L1-L286)[src/lib/email/preview.ts1-104](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/email/preview.ts#L1-L104)

### Token System Usage
ModuleFunctionPurpose`src/lib/repositories/invites.js``createInviteForEnquiry()`Generate booking invite token`src/lib/repositories/academy.js``addToWaitlist()`Generate academy waitlist response token`src/lib/repositories/membership.js``createMembershipOtp()`Generate membership signup token`src/pages/api/cron/expire-invites.ts`Cron handlerCheck invite expiry with `isInviteExpired()``src/pages/api/cron/retry-invites.ts`Cron handlerCheck invite age with `tokenAgeInHours()`
**Sources:**[src/lib/business/tokens.ts1-71](https://github.com/egac-web/egac/blob/0ba542fc/src/lib/business/tokens.ts#L1-L71)
