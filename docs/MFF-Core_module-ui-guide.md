# Module UI Guide

How core's own pages (`Dashboard.js`, `PreferencesTab.js`, `Layout.js`, etc.) actually look, extracted from the real code — not a design system that was planned in advance, just the conventions that are already consistently there. A module matching these stays visually native instead of looking bolted-on.

There is no CSS framework or component library anywhere in core (no Bootstrap, MUI, Tailwind). Everything is plain inline `style={{...}}` objects, with a couple of small CSS Modules (`AppLayout.module.css`, `Sidebar.module.css`) for the persistent shell only. A module should do the same — inline styles matching the values below, or its own CSS Module — rather than pulling in a UI library, which would clash immediately next to everything else.

## Where a module's content actually renders

The persistent shell (header + sidebar, `Layout.js`) never unmounts between routes. Depending on which slot a module uses (`App_frontend/src/core/moduleSlots.js`):

- **`registerDashboardTab(moduleId, { key, label, component, position, sidebarEntry, pageKey })`** — embeds as an extra tab inside `Dashboard.js`'s existing tab bar. Your `component` renders *inside* Dashboard's content wrapper, which already applies `padding: 20px`. Don't add your own outer full-page padding/background on top of that — just render your content directly, the same way `UserTab`/`Tickets`/etc. do.
  - **Add `sidebarEntry: true` if you also want a direct left-sidebar button that jumps straight to this tab** (tab bar and all) — no second component, no second route. This is almost certainly what you want if your instinct was to reach for `registerMenuPage` just to get a sidebar entry: use this instead, and your content stays inside Dashboard's shared tab bar, where users already expect to find every module.
- **`registerMenuPage(moduleId, { key, label, path, component, fullscreen, pageKey })`** — its own route, reached via the left sidebar, but **never renders inside Dashboard or its tab bar at all** — a genuinely separate page. Use this only when your content really doesn't belong alongside Setup/Support/other modules' tabs (e.g. you need a fundamentally different layout, or `fullscreen: true`). If you're unsure which to pick, pick `registerDashboardTab` with `sidebarEntry: true` — that's the default a module should reach for, not this.

### Restricting a tab/page to specific roles/groups/users

Pass **`pageKey`** (one of the keys your own `module.manifest.json` `"pages"` list declares — see `docs/MFF-Core_module-access.md`) and core hides the tab/sidebar entry/route entirely from anyone not granted that page, the same way an inactive whole module has no tab/route at all for anyone. Note this starts locked down, not open: a declared page with no grant configured yet defaults to developer/administrator only, so it's normal for a brand-new page to be invisible to everyone else until an admin actually grants it to other roles/groups/users at Setup → Preferences → 🔐 Access. This is done in core (`Dashboard.js`/`Sidebar.js`/`App.js` checking `isPageAllowed()` from `moduleSlots.js` against the per-user `pages` array `GET /api/modules` already returns for your module) rather than inside your own component, because `register(coreApi)` runs once at app startup, before login — your module has no way to know yet which pages this particular user can see, and nothing re-runs `register()` after login/logout to let you decide then. Don't render a "no access" message inside an always-registered tab as a substitute for this — that still leaks the tab's existence to someone who shouldn't see it. Your backend route(s) should still call `coreApi.requirePage(pageKey)` as well — the frontend hiding the tab is a UX nicety, not the actual enforcement.

Either way, you're never responsible for the header or sidebar — only your own content area.

### Where your tab sits in the bar

The Dashboard tabs, and the left-sidebar entries that come after the fixed "Dashboard" entry, are listed **alphabetically by their displayed label** (case-insensitive; a leading emoji is ignored). So choose the `label` with that in mind. A module's `position` in `registerDashboardTab` and the admin override `MODULE_<ID>_DASHBOARD_POSITION` no longer affect the order: both are still accepted but ignored, so a module has no need to set either.

## Color palette

Every color below is one already in real, repeated use across core — not invented for this guide.

| Purpose | Color | Used for |
|---|---|---|
| Primary / active state | `#007bff` | Active tab underline, primary buttons (Update), header accents |
| Success / positive | `#28a745` | Install button, satisfied-requirement chips, success messages |
| Danger / destructive | `#dc3545` | Remove/Reset/Log Out buttons, missing-requirement chips, error text |
| Info / secondary action | `#17a2b8` | Download-backup button |
| Neutral / secondary | `#6c757d` | Sync button, neutral secondary actions |
| Warning | `#ffc107` border / `#fff3cd` background / `#856404` text | Config-error banner, "Requirements unknown" chips |
| Panel/table header background | `#f8f9fa` | Table `<thead>`, inactive tab background, sub-tab bar background |
| Card/panel background | `#ffffff` | Content panels sitting on the `#f8f9fa` page |
| Borders | `#ddd`, `#dee2e6`, `#eee` | Table borders, tab bar borders, panel borders (roughly darkest to lightest, in that order) |
| Body text | `#333` | Primary text on light backgrounds |
| Secondary/meta text | `#666` | IDs, secondary column text |
| Muted/disabled text | `#999` | Version numbers, placeholder-ish text |

## Typography

No custom font stack — the browser default. Sizes used consistently:

- **16px, bold-ish via `<h1>`** — the one page-level title (header banner). A module should not need another one of these.
- **14px** — body copy, descriptive paragraph text above a table/section (e.g. `color: '#666', fontSize: '14px'`).
- **12–13px** — the default for almost everything interactive: table cell text, button labels, tab labels.
- **11px** — small/meta text: sidebar nav buttons, status chips, version strings, per-item counters.
- **Bold** (`fontWeight: 'bold'` or `600`) marks the active/selected state (active tab, active sidebar button) — not used for general emphasis.

## Spacing and shape

- **Border radius: `4px`** almost everywhere (buttons, panels, table containers). Small chips/badges use a slightly tighter `3px`. Sidebar nav buttons use `5px`.
- **Table cells: `padding: 12px`.**
- **Buttons: `padding: '4px 10px'`** for compact action buttons (table row actions), `'5px 10px'` to `'6px 14px'` for standalone buttons (Sync, Log Out).
- **Panels/cards: `padding: 20px'`**, `border: '1px solid #dee2e6'`, sitting on a `#f8f9fa` background.
- Buttons in a row get `marginLeft: '6px'` between them, not a flex `gap` — matches the existing pattern, though either reads fine.

## Recurring layout patterns

**Tab bar** (top-level, e.g. Dashboard's Setup/Support/Users tabs):
```js
{ display: 'flex', borderBottom: '1px solid #ddd' }
// each tab:
{
  padding: '15px 25px', cursor: 'pointer',
  backgroundColor: isActive ? '#f8f9fa' : 'transparent',
  borderBottom: isActive ? '2px solid #007bff' : 'none',
  fontWeight: isActive ? 'bold' : 'normal'
}
```

**Sub-tab bar** (e.g. Setup's Preferences/File Upload):
```js
{ display: 'flex', borderBottom: '1px solid #dee2e6', marginBottom: '20px', backgroundColor: '#f8f9fa' }
// each sub-tab:
{
  padding: '10px 20px', cursor: 'pointer',
  backgroundColor: isActive ? '#ffffff' : 'transparent',
  borderBottom: isActive ? '2px solid #007bff' : 'none',
  fontWeight: isActive ? 'bold' : 'normal',
  border: isActive ? '1px solid #dee2e6' : 'none'
}
// content panel below it:
{ backgroundColor: '#ffffff', padding: '20px', border: '1px solid #dee2e6', borderRadius: '0 0 4px 4px' }
```

**Table** (the dominant pattern for any list of things — see `PreferencesTab.js`'s Modules tab):
```js
// wrapper: { border: '1px solid #ddd', borderRadius: '4px', overflow: 'auto' }
// table: { width: '100%', borderCollapse: 'collapse' }
// thead: { backgroundColor: '#f8f9fa' }
// th/td: { padding: '12px', textAlign: 'left', borderBottom: '1px solid #ddd' }  // #eee for td
```

**Status/requirement chips** (small colored pill, e.g. green="satisfied", red="missing", amber="unknown"):
```js
{
  display: 'inline-block', padding: '2px 6px', margin: '1px 3px 1px 0',
  borderRadius: '3px', fontSize: '11px', color: 'white',
  backgroundColor: '#28a745' /* or #dc3545, or #fff3cd with color: '#856404' for warning */
}
```

**Buttons**, colored by intent, not by position — pick the color for what the action *does*:
```js
{ padding: '4px 10px', color: 'white', border: 'none', borderRadius: '4px', cursor: 'pointer', fontSize: '12px', backgroundColor: /* see palette */ }
```
Green = create/install/positive. Blue = primary/update. Red = destructive (remove/reset/logout). Teal = download/info. Gray = neutral/secondary (sync, refresh).

**Loading/empty states:**
```js
{ textAlign: 'center', padding: '20px' }  // "Loading..." text
// empty state:
{ padding: '20px', color: '#999', fontStyle: 'italic', border: '1px solid #ddd', borderRadius: '4px' }
```

**Error banners:**
```js
{ textAlign: 'center', padding: '20px', color: 'red', backgroundColor: '#ffeeee', borderRadius: '4px', margin: '10px 0' }
```

## What not to bring in

- No CSS-in-JS libraries, no UI component kits — plain inline styles or your own CSS Module, matching the values above.
- No custom fonts/font-loading — the app doesn't set one, don't introduce one.
- No full-page background/padding inside a `registerDashboardTab` component — Dashboard's own wrapper already provides that; adding your own doubles it up visually.
