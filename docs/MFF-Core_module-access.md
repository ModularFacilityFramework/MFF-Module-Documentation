# Module access — who can see and use an installed module (and one page within it)

An admin decides, per module, which **roles**, **groups** and **individual users** may use it (Setup → Preferences → Modules → **Access**). Anyone not granted gets no left-hand sidebar entry, no Dashboard tab, no route, and a `404` from the module's API — the module simply does not exist for them. There is no "you don't have access" screen.

The exact same model also applies *inside* a module, to one of its own declared pages at a time — see "Page access within a module" below.

## Rules

- **Nothing granted = open to everyone.** A module with no grants shows to all users. This is the state after install, and after upgrading core, so nobody is locked out.
- **Once anything is granted, only those are allowed:** the granted users, every user whose group (`setup_users.group_id`) is granted, every user holding a granted role, and **developers/administrators always** (plus the `SUPERUSER_ROLE`, if set). An admin cannot lock themselves out.
- A user belongs to exactly one group today, so "group access" means users whose group is granted.
- Any one match is enough (role OR group OR user). Page/tab access for *core* pages by role is still Setup → Preferences → 🔐 Access → Core - same `developer`/`administrator`/`SUPERUSER_ROLE` always-allowed rule as above (fixed to match: it used to only bypass for `developer`).

A user's grant on a module decides what they can **open**, not what other modules may use. A module that lists another module in its manifest `"requires"` may call it on behalf of any user who may use the *caller* directly or through a module that (transitively) requires the caller (one-way: a user of `alphapulldown` alone is served by `abcfold` and, through it, `slurm`; a user of `abcfold` alone is not served for `alphapulldown`), even if that user has no grant on the target; the user still cannot open the target directly. See `docs/MFF-Core_module-core-api.md`, "Calling a module you depend on".

A module with `"actsAsUsers": true` may also act as a user with no request in flight (a timer inside the module). The same rule applies: the user must be able to use the calling module, directly or through a module that requires it, and the target must be in the caller's `requires`. The user then needs no grant on the target. Every such call is audited in `core_module_background_audit`. See `docs/MFF-Core_module-core-api.md`, "Background identity".

## Where it is stored

`setup_module_access (module_id, subject_type 'user'|'group', subject_id)` for users and groups, and `setup_module_access_roles (module_id, role_id → setup_roles)` for roles, both in core's database. A grant for a user or group that is later deleted is harmless: it can no longer match anyone.

Page-level grants live in the same shape, scoped to a page: `setup_module_page_access (module_id, page_key, subject_type, subject_id)` and `setup_module_page_access_roles (module_id, page_key, role_id → setup_roles)`.

Existing deployments: after pulling this change, run **Setup → Preferences → Database → Apply Schema** once to create the tables. Until then core logs one warning and treats every module (and every page) as unrestricted — nothing breaks.

## Enforcement

Both come from `App_backend/src/core/services/moduleAccessService.js`:

1. **The module's API** — `moduleLoader.js` mounts an access gate in front of every module's router. An unrestricted module passes straight through, but **every module request needs a login, except the routes the module lists in `publicRoutes` in its manifest** (`docs/MFF-Core_module-manifest.md`). A restricted module answers `404` to anyone not granted.
2. **The UI** — `GET /api/modules` is filtered per caller (it accepts an optional token and never answers 401; logged out, it returns the full list as before). The frontend builds the sidebar, Dashboard tabs and routes from that one list, so a hidden module vanishes everywhere. The list is re-fetched on login/logout.

A Dashboard tab remembered from an earlier visit, or requested with `?tab=`, that the user can no longer see quietly falls back to their first tab.

## What a module must do

Nothing, for whole-module access. The access check happens in core, before the module's router. A module should **not** render its own "no access" page for this: a user without access never reaches it. (A module may still do its own finer-grained checks inside, with `req.user` and `checkRole`, exactly as before.)

## Page access within a module

The same open-by-default/grant-restricts model, scoped to one page of one module, for a module that wants core's admin to manage *which of its own pages* a user gets, instead of rolling its own role table for that.

1. **Declare** your pages in `module.manifest.json`'s `"pages": [{ "key": "run", "label": "Run" }, ...]` (`docs/MFF-Core_module-manifest.md`). Purely a declaration — nothing is restricted just by declaring it.
2. **Hide the tab/page itself** (frontend): pass `pageKey` to `registerDashboardTab`/`registerMenuPage` (`App_frontend/src/core/moduleSlots.js`) — core then hides that tab, its sidebar entry, and its route entirely from anyone not granted the page, the exact same way an inactive whole module has no tab/route for anyone (see `docs/MFF-Core_module-ui-guide.md`). Do this instead of registering the tab unconditionally and rendering a "no access" message inside it with `coreApi.useModuleAccess` — that still leaks the tab's existence to someone who shouldn't see it, the same mistake whole-module access already warns against above. `coreApi.useModuleAccess(moduleId)` → `{ loading, visible, pages }` still exists for finer-grained checks *within* a page you're already showing (e.g. hiding one button) — `pages` is `null` (nothing restricted) or the array of allowed keys — but it's not a substitute for `pageKey` when the whole tab/page should disappear. See `docs/MFF-Core_module-core-api.md`.
3. **Enforce** (backend): `coreApi.requirePage(pageKey)` on your own route, mounted after `authenticateToken` (it does not authenticate on its own — see `docs/MFF-Core_module-core-api.md`) — already bound to your own module's id, so you can't gate a page in a module that isn't yours. Takes an array for a route genuinely shared by more than one of your pages (any one granted page is enough). Answers `403` (not `404` — unlike the module-level gate, the caller is already past it and knows the route exists).
4. **Admin UI**: Setup → Preferences → 🔐 Access, one tab per **installed** module (plus one for Core's own pages) — only an installed module gets a tab, and only its declared pages appear in it.

A page with no grants yet defaults to developer/administrator (and `SUPERUSER_ROLE`, if set) only — unlike whole-module access, a declared page starts locked down, not open. Granting roles/groups/users to it opens it up to exactly those, same editor as whole-module access; it never widens access beyond the whole-module check either way (a user still needs the module itself first).

## Admin API (developer/administrator only)

| Route | Purpose |
|---|---|
| `GET /api/modules-management/access-summary` | `{ <moduleId>: { users, groups, roles } }` (granted name lists, not counts) for every module with grants, plus `lockedRoles` |
| `GET /api/modules-management/:id/access` | Currently granted `users` / `groups` / `roles` for a module, plus `allRoles` (every role) and `lockedRoles` |
| `PUT /api/modules-management/:id/access` | Body `{ userIds, groupIds, roleIds }`; replaces the module's grants (all empty = open to everyone) |
| `GET /api/modules-management/:id/pages` | That module's own declared pages, plus a per-page grant summary (name lists) and `lockedRoles` |
| `GET /api/modules-management/:id/pages/:pageKey/access` | Currently granted `users` / `groups` / `roles` for one page, plus `allRoles` and `lockedRoles` |
| `PUT /api/modules-management/:id/pages/:pageKey/access` | Body `{ userIds, groupIds, roleIds }`; replaces that page's grants |
| `GET /api/page-access/:id/access` | Core-only, roles-only: currently granted `roles` for one Core page, plus `allRoles`, `lockedRoles`, `isEnabled` |

Removing a module deletes its grants, module-level and page-level both.

All four access editors above (module, module-page, and Core page) share one frontend
component, `App_frontend/src/core/setup/access/AccessEditor.js` — one "Has access" box and
one "No access" box (not split by role/group/user), with `developer`/`administrator`/
`SUPERUSER_ROLE` pinned in as locked, non-removable chips. `AccessSummary.js` renders the
same information read-only, shown inline everywhere access is referenced (Modules tab,
Access tab, Core page table) with no click needed to see who currently has access.

## Limits

- The developer "view as role/user" mode in the sidebar does not simulate module or page access.
- Changing grants takes effect on the user's next page load or login (the module list is fetched then); the API gate (`moduleAccessGate` / `requirePage`) applies immediately.
