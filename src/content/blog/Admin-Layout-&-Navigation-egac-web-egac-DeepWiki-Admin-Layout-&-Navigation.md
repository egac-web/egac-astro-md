---
title: "Admin Layout and Navigation"
description: This document describes the admin interface layout system"
pubDate: 2026-03-18
---
# Admin Layout & Navigation
Relevant source files
- [astro.config.mjs](https://github.com/egac-web/egac/blob/0ba542fc/astro.config.mjs)
- [src/layouts/AdminBase.astro](https://github.com/egac-web/egac/blob/0ba542fc/src/layouts/AdminBase.astro)
- [src/layouts/Base.astro](https://github.com/egac-web/egac/blob/0ba542fc/src/layouts/Base.astro)
- [src/pages/admin/bookings.astro](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/bookings.astro)
- [src/pages/admin/reports.astro](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/reports.astro)
- [src/styles/tailwind.css](https://github.com/egac-web/egac/blob/0ba542fc/src/styles/tailwind.css)

This document describes the administrative interface layout system, including the sidebar navigation structure, page composition patterns, and visual design. The `AdminBase.astro` component serves as the wrapper for all admin pages, providing consistent navigation, authentication checks, and styling.

For information about specific admin features (enquiries, bookings, reports, etc.), see the individual sections: [Enquiries Management](/egac-web/egac/4.2-enquiries-management), [Bookings Management](/egac-web/egac/4.3-bookings-management), [Reports Dashboard](/egac-web/egac/4.4-reports-dashboard), [System Settings](/egac-web/egac/4.5-system-settings), and [Academy Management](/egac-web/egac/4.6-academy-management). For authentication mechanisms, see [API Reference](/egac-web/egac/5-api-reference).

---

## Layout Architecture

The admin interface uses a two-column layout with a persistent sidebar and a flexible content area. The `AdminBase` layout component wraps all admin pages and enforces consistent structure.

### Component Structure

```
AdminBase.astro
(Layout Component)

Sidebar Navigation
(aside element)

Main Content Area
(slot)

admin/enquiries.astro

admin/bookings.astro

admin/reports.astro

admin/academy.astro

admin/templates.astro

admin/settings.astro

Navigation Items
(navItems array)

EGAC Branding
(header section)

Footer
(environment & logout)
```

**Sources:**[src/layouts/AdminBase.astro1-77](https://github.com/egac-web/egac/blob/0ba542fc/src/layouts/AdminBase.astro#L1-L77)

---

## AdminBase Component Interface

The `AdminBase` component accepts two props that control page metadata and navigation state:
PropTypeRequiredDescription`title``string`YesPage title, displayed in browser tab as "{title} — EGAC Admin"`activeNav``'dashboard' | 'enquiries' | 'bookings' | 'academy' | 'templates' | 'settings'`NoIdentifies which navigation item should be highlighted
The `activeNav` prop determines which navigation link receives active styling (background color and text highlighting).

**Sources:**[src/layouts/AdminBase.astro5-8](https://github.com/egac-web/egac/blob/0ba542fc/src/layouts/AdminBase.astro#L5-L8)[src/layouts/AdminBase.astro12](https://github.com/egac-web/egac/blob/0ba542fc/src/layouts/AdminBase.astro#L12-L12)

---

## Navigation System

The navigation is defined as a static array of items mapped to admin routes. Each item has an identifier, display label, and target URL.

### Navigation Item Configuration

```
navItems Array

id: 'dashboard'
label: 'Dashboard'
href: '/admin/reports'

id: 'enquiries'
label: 'Enquiries'
href: '/admin/enquiries'

id: 'bookings'
label: 'Bookings'
href: '/admin/bookings'

id: 'academy'
label: 'Academy'
href: '/admin/academy'

id: 'templates'
label: 'Templates'
href: '/admin/templates'

id: 'settings'
label: 'Settings'
href: '/admin/settings'
```

Each navigation item is rendered as an anchor element with conditional styling based on whether `activeNav` matches the item's `id`. Active items receive the class `bg-[#2d5ea5] text-white`, while inactive items receive `text-[#d9e4f6] hover:bg-[#2d5ea5] hover:text-white`.

**Sources:**[src/layouts/AdminBase.astro14-21](https://github.com/egac-web/egac/blob/0ba542fc/src/layouts/AdminBase.astro#L14-L21)[src/layouts/AdminBase.astro41-61](https://github.com/egac-web/egac/blob/0ba542fc/src/layouts/AdminBase.astro#L41-L61)

---

## Sidebar Layout

The sidebar is a fixed-width column (240px / `w-60`) on desktop viewports, positioned with `sticky` behavior at the top of the viewport. On mobile, it collapses to a full-width horizontal layout with a bottom border.

### Sidebar Structure

```
aside.w-full.lg:w-60
(Sidebar Container)

Header Section
(px-5 py-7)

Navigation Section
(nav px-3 py-4)

Footer Section
(px-5 py-5)

EGAC Link
(text-2xl font-bold)

ul.space-y-1
(Navigation Items)

Environment Badge
(Production)

Logout Link
(/admin/logout)
```

**Visual Characteristics:**
ElementClassesPurposeSidebar container`bg-[#214c8f] text-slate-100`Blue background matching EGAC brandingHeader section`border-b border-[#2f5ca0]`Separated branding areaNav items`rounded-md px-3 py-2.5`Clickable navigation linksActive nav`bg-[#2d5ea5] text-white`Highlighted current pageInactive nav`text-[#d9e4f6] hover:bg-[#2d5ea5]`Hover state feedbackFooter`border-t border-[#2f5ca0]`Separated utility area
**Sources:**[src/layouts/AdminBase.astro34-68](https://github.com/egac-web/egac/blob/0ba542fc/src/layouts/AdminBase.astro#L34-L68)

---

## Responsive Behavior

The layout adapts to viewport size using Tailwind's responsive prefixes:

### Desktop (lg breakpoint and above)

- Sidebar: Fixed 240px width, sticky positioned at top
- Layout: Two-column flex layout with sidebar on left
- Footer: Visible with environment and logout controls

### Mobile (below lg breakpoint)

- Sidebar: Full-width horizontal bar at top of page
- Layout: Single-column stack
- Footer: Hidden (environment/logout controls not displayed)
- Border: Bottom border instead of right border

**Sources:**[src/layouts/AdminBase.astro33-68](https://github.com/egac-web/egac/blob/0ba542fc/src/layouts/AdminBase.astro#L33-L68)

---

## Main Content Area

The main content area occupies the remaining horizontal space and wraps page-specific content via Astro's `<slot />` element. All content receives consistent padding:

```
div.flex-1
(Main Container)

main
(px-4 py-6 sm:px-6
lg:px-8 lg:py-8)

slot
(Page Content)
```

Padding scales responsively:

- Mobile: `px-4 py-6` (16px horizontal, 24px vertical)
- Small: `px-6` (24px horizontal)
- Large: `px-8 py-8` (32px both directions)

**Sources:**[src/layouts/AdminBase.astro70-74](https://github.com/egac-web/egac/blob/0ba542fc/src/layouts/AdminBase.astro#L70-L74)

---

## Usage Pattern in Admin Pages

Admin pages import and wrap their content with `AdminBase`, passing `title` and `activeNav` props. Authentication checks are performed before the layout is rendered.

### Typical Admin Page Structure

```
---
(Frontmatter)

import AdminBase
from layouts/AdminBase.astro

checkAdminToken()
(Authentication)

Load page-specific data
(repositories, queries)

---

AdminBase Component
(title & activeNav props)

Page-specific markup
(tables, forms, charts)

Client-side scripts
(optional)
```

### Example: Bookings Page

The bookings page demonstrates the standard pattern:

1. **Imports** — Brings in `AdminBase` layout ([src/pages/admin/bookings.astro6](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/bookings.astro#L6-L6))
2. **Authentication** — Checks admin token, redirects if invalid ([src/pages/admin/bookings.astro14-17](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/bookings.astro#L14-L17))
3. **Data Loading** — Fetches bookings, age groups, and enriches with enquiry data ([src/pages/admin/bookings.astro19-52](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/bookings.astro#L19-L52))
4. **Render** — Wraps content in `<AdminBase title="Bookings" activeNav="bookings">` ([src/pages/admin/bookings.astro55](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/bookings.astro#L55-L55))
5. **Content** — Page-specific UI components ([src/pages/admin/bookings.astro56-177](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/bookings.astro#L56-L177))
6. **Scripts** — Client-side interactivity for attendance buttons ([src/pages/admin/bookings.astro179-252](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/bookings.astro#L179-L252))

**Sources:**[src/pages/admin/bookings.astro1-253](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/bookings.astro#L1-L253)

---

## Styling System

The admin interface uses Tailwind CSS utility classes imported via the base layout. The color scheme is defined with custom hex values:
Color UsageHex ValueTailwind ClassSidebar background`#214c8f``bg-[#214c8f]`Sidebar border`#1f457f``border-[#1f457f]`Section dividers`#2f5ca0``border-[#2f5ca0]`Active navigation`#2d5ea5``bg-[#2d5ea5]`Inactive text`#d9e4f6``text-[#d9e4f6]`Page background`#eff2f7``bg-[#eff2f7]`
The public-facing `Base.astro` layout uses a different color scheme (gray-based) to visually distinguish public pages from admin pages.

**Sources:**[src/layouts/AdminBase.astro10](https://github.com/egac-web/egac/blob/0ba542fc/src/layouts/AdminBase.astro#L10-L10)[src/layouts/AdminBase.astro32-67](https://github.com/egac-web/egac/blob/0ba542fc/src/layouts/AdminBase.astro#L32-L67)[src/styles/tailwind.css1-3](https://github.com/egac-web/egac/blob/0ba542fc/src/styles/tailwind.css#L1-L3)[src/layouts/Base.astro10](https://github.com/egac-web/egac/blob/0ba542fc/src/layouts/Base.astro#L10-L10)

---

## Environment Indicator & Logout

The sidebar footer (visible only on desktop) displays system status and provides a logout mechanism:

```
Footer Section
(hidden lg:block)

EGAC Admin
(font-medium)

Environment Badge
(Production)

Logout Link
(href: /admin/logout)

Green dot
(h-2 w-2 bg-emerald-400)
```

The environment indicator is currently hardcoded to "Production". The logout link navigates to `/admin/logout`, which clears the admin token cookie.

**Sources:**[src/layouts/AdminBase.astro63-67](https://github.com/egac-web/egac/blob/0ba542fc/src/layouts/AdminBase.astro#L63-L67)

---

## Navigation Visual Indicators

Each navigation item includes a small circular bullet point that matches the text color, providing a visual rhythm to the sidebar:

```
<span class="inline-block h-2 w-2 rounded-full bg-current opacity-80" aria-hidden="true"></span>
```

The `bg-current` class makes the bullet inherit the link's text color, and `opacity-80` provides subtle contrast. The `aria-hidden="true"` attribute prevents screen readers from announcing the decorative element.

**Sources:**[src/layouts/AdminBase.astro54](https://github.com/egac-web/egac/blob/0ba542fc/src/layouts/AdminBase.astro#L54-L54)

---

## Contrast with Public Layout

The public `Base.astro` layout differs significantly from `AdminBase`:
AspectAdminBaseBaseNavigationPersistent sidebar with 6 sectionsSimple header with logoBackgroundBlue sidebar + light gray contentWhite header + gray-50 contentWidthFull-width with sidebarMax-width 2xl (672px) centeredFooterEnvironment + logout (desktop)Copyright noticePurposeMulti-page admin dashboardSingle-page public forms
**Sources:**[src/layouts/AdminBase.astro1-77](https://github.com/egac-web/egac/blob/0ba542fc/src/layouts/AdminBase.astro#L1-L77)[src/layouts/Base.astro1-43](https://github.com/egac-web/egac/blob/0ba542fc/src/layouts/Base.astro#L1-L43)

---

## Integration with Admin Pages

### Reports Dashboard Example

The reports dashboard uses `AdminBase` with `activeNav="dashboard"`:

```
<AdminBase title="Reports" activeNav="dashboard">
  <div class="space-y-6">
    <h1 class="text-4xl font-semibold text-slate-900">Dashboard</h1>
    <!-- Metric cards, charts, etc. -->
  </div>
</AdminBase>
```

This highlights the "Dashboard" link in the sidebar while rendering the reports content in the main area.

**Sources:**[src/pages/admin/reports.astro147](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/reports.astro#L147-L147)[src/pages/admin/reports.astro6](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/reports.astro#L6-L6)

### Common Page-Level Patterns

All admin pages follow these conventions:

1. **Authentication first** — Token validation happens before any data loading
2. **Server-side data loading** — Repositories are called in the frontmatter
3. **Type safety** — TypeScript types imported from `schema.js`
4. **Consistent spacing** — Top-level content wrapped in `space-y-5` or `space-y-6`
5. **Semantic HTML** — Proper heading hierarchy (h1 → h2 → h3)

**Sources:**[src/pages/admin/bookings.astro14-17](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/bookings.astro#L14-L17)[src/pages/admin/reports.astro14-16](https://github.com/egac-web/egac/blob/0ba542fc/src/pages/admin/reports.astro#L14-L16)

---

## File Locations
PathPurpose`src/layouts/AdminBase.astro`Core admin layout component`src/layouts/Base.astro`Public layout (for comparison)`src/pages/admin/*.astro`Individual admin pages using AdminBase`src/styles/tailwind.css`Global Tailwind imports`astro.config.mjs`Astro + Cloudflare adapter configuration
**Sources:**[src/layouts/AdminBase.astro1-2](https://github.com/egac-web/egac/blob/0ba542fc/src/layouts/AdminBase.astro#L1-L2)[src/layouts/Base.astro1-2](https://github.com/egac-web/egac/blob/0ba542fc/src/layouts/Base.astro#L1-L2)[src/styles/tailwind.css1-3](https://github.com/egac-web/egac/blob/0ba542fc/src/styles/tailwind.css#L1-L3)[astro.config.mjs1-14](https://github.com/egac-web/egac/blob/0ba542fc/astro.config.mjs#L1-L14)
