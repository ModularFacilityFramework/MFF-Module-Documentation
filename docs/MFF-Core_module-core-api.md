# What core actually gives a module (coreApi)

This is the exhaustive, current `coreApi` object a module receives — not a summary, the literal shape as it exists in code right now, on both the backend and the frontend. A module never imports core's own files directly; everything it can reach goes through this one object, handed to it once when its entry file registers itself.

## Backend — `App_backend/src/core/index.js`, passed to your `backendEntry`'s `register(coreApi)`

```js
{
    authenticateToken,                    // middleware - requires a valid access token
    authenticateTokenFromHeaderOrQuery,   // same, but also accepts ?token= (for <img>/<a> src, which can't send headers)
    checkRole,                            // checkRole(['administrator', ...]) - middleware factory
    requirePage,                          // requirePage('run') - middleware factory, see "Per-page access" below
    pool,                                 // core's own MariaDB connection pool (see caveat below)
    listRoles,                            // () => Promise<string[]> - every role name that exists in this installation
    getUserRoles,                         // (req.user) => Promise<string[]> - the role names THIS user holds (see "The current user's roles")
    listUsers,                            // () => Promise<Array<{id, username, email, first_name, last_name, status, account_type, registration_pending, group_id, job_position, is_pi, roles: string[]}>>
    listGroups,                           // () => Promise<Array<{id, group_name, archived_at}>> - see "`listGroups()`"
    getGroupPis, listGroupPis,            // see "`getGroupPis(groupId)` / `listGroupPis()`"
    getUserWorkspacePath,                 // (user) => Promise<{active, path, reason}> - see below
    callModule,                           // (token, moduleId, method, path, opts) => Promise<{ok, status, data, error}> - see below
    callModuleAsUser,                     // (userId, targetModuleId, method, path, opts) => Promise<{ok, status, data, error, ...}> - background identity, see below
    getUserWorkspacePathForUserId,        // (userId) => Promise<{active, path, reason}> - background identity, see below
    registerUpgradeHandler,               // (path, handler, { page?, identify? }) - WebSocket upgrade, see "WebSocket: `registerUpgradeHandler`"
    getModuleConfig,                      // () => Record<string, string> - see below
    toChartSlices, chartResponse,         // build the data a DrillDownPie loads - see "Charts: the shared drill-down pie"
    apiBasePath,                          // e.g. "/mff-api" - the prefix your own routes are mounted under
    config: {
        apiBasePath, appName, bannerTitle, superuserRole, nodeEnv
    }
}
```

### Current user (backend)

There is no `getCurrentUser()` — you don't need one. Once a request has passed through `authenticateToken` (yours or core's, same middleware), Express already has it: `req.user = { id, username, email, group_id, roles }` (`roles` is `string[]` — see "The current user's roles" below for its exact contract). That *is* the current user, for that request — no extra lookup needed for any of those four fields. `listRoles`/`listUsers` below are for a different question — "what exists system-wide" — not "who is making this request."

### Who may use your module

Core decides this, not you: an admin grants each module to roles, groups and/or users (Setup → Preferences → Modules → Access), and core hides the module from everyone else and answers `404` to its API before your router runs. You need no code for it and should not build a "no access" page — see `docs/MFF-Core_module-access.md`. `req.user` additionally carries `group_id` (the user's single group, or `null`).

### Per-page access: `requirePage(pageKey)`

```js
router.get('/run', coreApi.authenticateToken, coreApi.requirePage('run'), (req, res) => { ... });

// A route genuinely shared by two of your own pages: an array is "any of these" -
// a user granted either page can use it, not just one granted both.
router.get('/settings', coreApi.authenticateToken, coreApi.requirePage(['setup', 'run']), (req, res) => { ... });
```

For a page you declared in `module.manifest.json`'s `"pages"` (`docs/MFF-Core_module-manifest.md`), this is the same idea as `checkRole`, but for "can this user use this *specific page* of my module" rather than a role check — an admin grants/restricts it per page at Setup → Preferences → 🔐 Access, same user/group/role model as whole-module access. Already bound to your own module's id — you only ever pass a page key (or an array of them), never a moduleId, so you structurally cannot gate a page in a module that isn't yours. Answers `403`, not `404` (unlike the module-level gate, the caller already knows this route exists — they're already past it). A page with no grants at all is open to everyone who already has module-level access, same "nothing granted = open" rule as everywhere else in this file.

**It does not authenticate on its own — exactly like `checkRole`, mount `authenticateToken` before it, always.** Don't assume `moduleAccessGate` already did this for you: it only authenticates when the *whole module* is restricted, so an otherwise-open module (the common case — most modules have no module-level grants) reaches your route with no `req.user` at all unless you authenticate yourself. Skipping `authenticateToken` doesn't fail safe here, it fails locked: `req.user` is `undefined`, so a restricted page's grant check always comes back `false` and every request gets `403`, logged in or not.

See `docs/MFF-Core_module-access.md` for the full contract, including the frontend half (`pages` on `GET /api/modules`).

### `listRoles()`

```js
const roles = await coreApi.listRoles(); // ['administrator', 'developer', 'facility_manager', 'facility_staff', 'facility_superuser', 'facility_user']
```
Reads `setup_roles` directly — the same table `checkRole` itself is ultimately backed by. This is the full, authoritative list, not derived from traffic or guessed.

### `listUsers()`

```js
const users = await coreApi.listUsers();
// [{ id, username, email, first_name, last_name, status, account_type, registration_pending, group_id, job_position, is_pi, roles: ['administrator'] }, ...]
```
`account_type` is `standard` or `no_login` (a no-login account is a contact/record that never logs in, but is still listed and can be a group member or PI); `registration_pending` is 1 for a self-registration an admin has not approved or rejected yet.
Sorted by first name, then last name, then username (case-insensitive); it used to be sorted by id (v0.1.199). `job_position` is the job position's name, or null. `is_pi` is a boolean: true when the job position is named "PI". Core has no separate `is_pi` column; the job position named "PI" is the marker.

**UI rule for modules:** the rule applies to every list of people a module shows (tables, pickers, exports), not only drop-downs: sort by first name, then last name, case-insensitive, and label a user "First Last (username)" wherever a username is shown (core's own lists do). A PI or supervisor picker lists only users with `is_pi`. A group PI that is already linked stays displayed even if their position is not PI.

Deliberately simple — full list, no pagination or search, and a fixed sort (that complexity lives only in `GET /users`, the admin UI's own endpoint, and isn't duplicated here). If a real need for paginated backend-side access ever comes up, this is the function to extend, not a new one to add.

### `listGroups()`

```js
const groups = await coreApi.listGroups();
// [{ id: number, group_name: string, archived_at: Date | null }, ...]
```
Every group in `setup_groups`, archived ones included and marked (`archived_at` non-null = archived, null = active), ordered by `group_name` then `id`. Read-only. `id` is a plain number. Use it instead of raw SQL on `setup_groups`. Example: map a group id from `listUsers()` (`group_id`) to the group's current name.

Available from core v0.1.197. A module that must also run on an older core can feature-detect with `typeof coreApi.listGroups === 'function'`.

### `getGroupPis(groupId)` / `listGroupPis()`

```js
const pis = await coreApi.getGroupPis(groupId);
// [{ user_id, is_primary, username, first_name, last_name, name, email }, ...]  - primary first, then by first and last name
const byGroup = await coreApi.listGroupPis();
// { [groupId]: [ ...same shape... ] }
```
A group can have several PIs, exactly one primary. See `docs/MFF-Core_db-schema.md` (`setup_group_pis`).

### `getUserWorkspacePath(user)`

```js
const workspace = await coreApi.getUserWorkspacePath(req.user);
// workspace = { active: boolean, path: string | null, reason: string | null }
```

Provisions (if needed) and returns one absolute, on-disk directory belonging to `user` — the calling module's own responsibility for what goes inside it (its own subfolders, file formats, retention). Core never looks inside this directory and never assumes anything about its contents — "core must stay generic" applies here exactly as everywhere else. The directory is created (`mkdir -p`, idempotent) on first request for that user, keyed by their numeric `id` (stable — never username, which can change).

`user` must be the calling request's own `req.user` (as populated by `authenticateToken`/`authenticateTokenFromHeaderOrQuery`) — core has no notion of "the current user" outside of a request context, the same reason `callModule` requires its caller to pass its own token explicitly rather than inferring one.

Behavior worth knowing:
- **Never throws.** Feature not configured, this user not permitted, or a real filesystem error (permission denied, missing mount, disk full) all come back as `{ active: false, path: null, reason: '...' }` instead of an exception — always safe to call with no `try/catch` of your own.
- **`active: false` is the only thing your own logic needs to branch on.** `reason` exists for logging/debugging on your side, not for different handling per value — treat every `active: false` the same way (a clean error to your own caller, e.g. a 503).
- `reason` values: `'not_configured'` (this deployment's `.env` has no `WORKSPACE_BASE_DIR` set — a sysadmin-level setting, not something an admin UI toggles), `'forbidden'` (this specific user's workspace storage access is off, or the user doesn't exist, or a filesystem error occurred).
- **Access is per-individual-user, not role-based.** An administrator ticks a checkbox for each user directly in Users & Groups management (`setup_users.workspace_storage_enabled`) — the same table/UI that already manages a user's status. There is no group/role gate and no separate admin screen for this feature.
- The base directory itself (`WORKSPACE_BASE_DIR`) is deliberately **not** exposed anywhere in `coreApi` — only the already-resolved, already-created per-user path is. A module never learns the shared base path, only its own caller's slice of it.

### `callModule(token, moduleId, method, path, opts)` — the "reach another module" primitive

```js
const result = await coreApi.callModule(
    req.headers.authorization,      // the CALLING user's own token - never a service/admin identity
    'slurm',                        // moduleId, exactly as it appears in getModules() / the module's own manifest "id"
    'GET',
    '/external/resource-profile',   // relative to that module's own mountPath
    { params: { gpuCount: 2 } }     // optional: { params, body, timeoutMs }
);
// result = { ok: boolean, status: number, data: any, error: string | null }
```

This is what "Modules never import directly from each other — all cross-module communication goes through core" actually means in code. Core resolves `moduleId`'s real `mountPath` from its own registry and makes a real HTTP request to it — the same request any other authenticated client would make — over a loopback connection. It **never** hardcodes which modules exist or what their APIs look like; there is no `coreApi.slurm`, `coreApi.abcfold`, or any other module-named function anywhere in core, on principle (see `CLAUDE.md`'s "MFF architecture rules" — core must stay generic, domain-specific code is a module's job, not core's).

Behavior worth knowing:
- **Never throws.** Module-not-installed, network errors, timeouts (default 15s, override via `timeoutMs`), and non-2xx responses all come back as `{ ok: false, ... }` instead of an exception — always safe to call with no `try/catch` of your own.
- **Response body is content-type aware, not forced to text.** `application/json` is parsed for you (`data` is the parsed object), `text/*` comes back as a string, anything else (a raw file download, for example) comes back as a plain `Buffer` — untouched, not corrupted by an unconditional `.text()`.
- **`token` is yours to extract and pass through** — typically just `req.headers.authorization` from the request your own route handler is already inside. The target module's own `authenticateToken`/`checkRole` and any per-user ownership logic apply exactly as if that user had called it directly — `callModule` grants no elevated *role*. What it can bypass is the target's module and page grants, and only for a declared dependency (see "Calling a module you depend on" below).
- **`coreApi.callModule` is bound to the calling module** — core knows which module is calling.
- **Calling a module you depend on:** if the caller lists the target module's `id` in its manifest `"requires"` (`docs/MFF-Core_module-manifest.md`), core signs the loopback request: an HMAC bound to the caller, the target, the user's bearer token (by hash), valid for 60 seconds. The target's access gate accepts such a request for a user who may use the **caller**, even if that user has no grant on the target. The target's `requirePage` is skipped for that call (`req.delegatedFrom` = the caller's module id). `authenticateToken`, `checkRole` and any ownership logic still apply. The user still cannot open the target module directly (they get `404`).
- **A module that calls another module without listing it in `requires`** gets plain user-level access (the user's own grants decide), and a warning in the backend log. `requires` must therefore list every module the module calls.
- **Chains are delegated, one-way:** the gate accepts a delegated call when the user may use the calling module directly, **or** through a module that (transitively) lists the caller in `requires`. Example: a user of `alphapulldown` alone is served by `abcfold`, and through `abcfold` by `slurm`, because `alphapulldown` requires `abcfold` and `abcfold` requires `slurm`. It is one-way: a user of `abcfold` alone is not served for `alphapulldown`. Every module must list in `requires` every module it calls.
- **A named, convenient wrapper around a specific module's API (e.g. `slurm.getResourceProfile()`) is not something core provides** — that naming and interpretation of that module's specific status codes/error messages belongs to whichever module wants to call it, built in that module's own repo on top of this primitive. See `docs/MFF-Core_module-core-api.md`'s companion handoff spec (given separately) for a worked example.

### Background identity: `callModuleAsUser` and `getUserWorkspacePathForUserId`

`callModule` and `getUserWorkspacePath` need a live request: they take the caller's own token or `req.user`. A module that works with no request in flight (typically a timer inside the module, for example `abcfold` advancing a user's queue) uses these two instead. Core has no scheduler of its own; the timer is the module's.

A module must declare `"actsAsUsers": true` in its manifest first (`docs/MFF-Core_module-manifest.md`).

```js
const result = await coreApi.callModuleAsUser(
    userId,                        // the user the call acts as
    'slurm',                       // target module id; must be in this module's manifest "requires"
    'GET',
    '/external/resource-profile',  // relative to the target's mountPath
    { params: { gpuCount: 2 }, timeoutMs: 15000 }   // optional
);
// result = { ok, status, data, error }
// or, when core refuses:
// { ok: false, status: 0, data: null, error, refused: true, reason }

const ws = await coreApi.getUserWorkspacePathForUserId(userId);
// ws = { active, path, reason }
```

- **`opts`** is the same as for `callModule` and nothing else: `params` (an object, sent as the query string), `body` (any JSON-serialisable value, sent as JSON) and `timeoutMs` (default 15000).
- **Never throws.** Same convention as `callModule`.
- **`reason` on a refusal** (`refused: true`) is one of:
  - `not_declared`: the calling module's manifest lacks `actsAsUsers`.
  - `not_required`: the target is not in the calling module's manifest `"requires"`.
  - `user_unavailable`: the user is missing, inactive, a no-login account, or pending registration.
  - `no_access`: the user may not use the calling module. This is the same rule as live delegation (see "Calling a module you depend on"), including through a module that requires it.
- **`getUserWorkspacePathForUserId`** follows the same user rules. It answers `{ active: false, path: null, reason: 'forbidden' }` when refused; other reasons are those of `getUserWorkspacePath`.
- **Mechanism.** Core mints an access token for the user, valid 60 seconds, with a claim `bg: { by, target }`, and uses it only on a loopback call to the target module. The token is valid only on that module's mount path (anywhere else, for example a core route, answers `401`). The calling module never sees the token. The target sees the user as `req.user`, with `req.user.backgroundBy` set to the calling module's id. The delegation header is the same as for `callModule`, so the user needs no grant on the target. The target's `authenticateToken`, `checkRole` and ownership logic still apply to that user.
- **Audit.** Every call, allowed or refused, writes a log line (`audit: module_background`) and a row in `core_module_background_audit` (`docs/MFF-Core_db-schema.md`).

### WebSocket: `registerUpgradeHandler`

```js
coreApi.registerUpgradeHandler('/stream', (req, socket, head) => {
    // hand the socket to your own `ws` server
}, { page: 'run' });   // { page } is optional
```
`coreApi.registerUpgradeHandler(path, handler, { page?, identify? })` lets a module accept WebSocket connections. The handler is `(req, socket, head)`. `path` is relative to the module's `mountPath`: the handler serves that exact path and any sub-path below it.

**Manifest flag:** the module must declare `"websocket": true` (`docs/MFF-Core_module-manifest.md`). Without it, `registerUpgradeHandler` throws and the module is skipped.

**Client URL:** `wss://<host><apiBasePath><mountPath><path>?token=<access token>`.

**What core checks before the handler runs:**
- The URL must be under a module mount path that has a registered handler; otherwise `404` and the socket is closed. A module with no handler is always refused.
- A login is required: `Authorization: Bearer` or `?token=`. The token is removed from `req.url` and `req.query` before the module sees it.
- The module access check (`404` if the user has no access, the same as the HTTP gate), and the page check if `{ page }` is given (the same as `requirePage`).
- A developer's "view as" identity is refused (`403`).
- A request that is not a WebSocket upgrade answers `400`.

**What the handler gets:** `req.user`, `req.query`, `req.moduleId` and `req.upgradePath`. The `socket` is the raw `net.Socket`. Core adds no `ws` dependency: the module brings its own `ws`.

**`identify` (optional):** `identify: async (req) => userId | { userId } | null`, for a WebSocket client that cannot send the app access token (for example a third-party client that owns the `token` URL parameter). The module proves who is connecting with its own credential (an HttpOnly cookie, a one-time ticket). Then core leaves `req.url` untouched (no token stripping, and the app JWT is not read), loads that user's real identity from the database, and still runs the module access check, and the `page` check, at every connect, so a revoked user is refused at reconnect. `null`, a throw or an unknown user answers `401`. The module must return only ids for credentials it minted and can verify (the same trust model as manifest `publicRoutes`).

**Access token lifetime:** the app access token lives `JWT_ACCESS_EXPIRES_IN` (default 15 m) and is refreshed by the app's axios client, not by a tab opened with `window.open`.

The production web server must pass upgrades through: see "WebSockets through the web server" in `docs/MFF-Core_dev-setup.md`. Sources: `App_backend/src/core/services/moduleUpgrade.js`, `tests/moduleUpgrade.test.js`.

### `getModuleConfig()` — your own non-database config, from core's one `.env`

```js
const cfg = coreApi.getModuleConfig();
// e.g. for module id "pandda2inspect", reads MODULE_PANDDA2INSPECT_* from
// App_backend/.env and hands back { APP_URL: '...', APP_DIR: '...', ... }
// with the MODULE_PANDDA2INSPECT_ prefix already stripped
```

This is how a module gets any config value it needs, database credentials included — see `docs/MFF-Core_env-policy.md` for the full policy (the one-`.env` rule, what a module/admin must each do, why). Short version: an admin adds a value directly to `App_backend/.env`, namespaced `MODULE_<ID>_<KEY>` (module id upper-cased, e.g. `MODULE_PANDDA2INSPECT_APP_URL=https://...`), and restarts core's own backend — the same restart an Install/Update already tells the admin to run, not a separate step.

Behavior worth knowing:
- **Takes no arguments.** `moduleLoader.js` binds this to your own module's `id` (from its `module.manifest.json`, validated against your folder name before your `register()` ever runs) before handing you `coreApi` — you cannot pass a different module's id and read its values; there is no argument to do that with.
- **Only your own namespace, never raw `process.env`.** You get back exactly the `MODULE_<YOUR_ID>_*` keys that exist, prefix stripped, as a plain object — never core's own secrets (JWT keys, `DB_PASSWORD`, another module's values), matching the same "deliberately curated subset" principle every other `coreApi` entry follows.
- **Returns `{}` if nothing is set**, never `null`/`undefined`/an exception — check for the specific keys you need (`cfg.APP_URL === undefined` means unset), same "never throws" convention as `callModule`/`getUserWorkspacePath`.
- **Read at the moment you call it, not cached at module load** — but since it reads `process.env`, which is only populated once at process startup (`dotenv.config()`), a value added to `App_backend/.env` still needs a backend restart before a call to `getModuleConfig()` sees it. This removes the "wrong file" class of error, not the "needs a restart" one.

### `pool` caveat

You get core's raw connection pool because some things (like the two functions above, before they existed) had no other way to be built. Prefer `listRoles()`/`listUsers()`/`checkRole()` over querying `setup_users`/`setup_roles` directly yourself — core's own schema can change; these three functions are the stable contract, raw SQL against core's tables isn't.

## Frontend — `App_frontend/src/core/App.js`, passed to `loadModules(coreApi)`

```js
{
    api, apiGet, apiPost, apiPut, apiDelete,   // the same authenticated axios client core's own pages use
    buildFileUrl, buildPublicAssetUrl,         // URL builders for private/public uploaded files; take a file name or path and ignore any query string
    useAuth,                                   // hook - call it yourself, inside your own component
    useRoles,                                  // (userData) => { roles: string[], hasRole, hasAnyRole, hasAllRoles, canManageUsers } - see "The current user's roles"
    registerDashboardTab, registerMenuPage,    // the two extension points (see main README/CLAUDE.md)
    usePageState, clearPageState,              // remember a page's state per user - see "`usePageState`: page state persistence"
    getAllRoles,                               // () => Promise<string[]>
    getAllUsers,                               // (params) => Promise<{success, items, total, page, pages, limit}>
    DrillDownPie, fetchSlices, normalizeSlices // the shared pie chart - see "Charts: the shared drill-down pie"
}
```

### Current user (frontend)

```js
function MyModuleComponent() {
  const { userData } = coreApi.useAuth();
  // userData.user  = { id, username, email, ... }
  // userData.roles = [{ role: 'facility_user' }, ...]   <- OBJECTS, not strings. Don't compare these
  //                                                         to role names yourself - use useRoles (below).
}
```
`useAuth()` is already in the bundle above — call it inside your own component (hooks can only run in component/hook context, so it can't be pre-resolved for you). No separate `getCurrentUser()` exists on purpose — this already is that.

### The current user's roles — exact contract

One question, answered one documented way per side. **Role names are the strings in `setup_roles.role`, spelled exactly as `coreApi.listRoles()` / `getAllRoles()` returns them** (the seeded ones are lowercase, e.g. `facility_user`; a role an admin adds keeps whatever spelling it was created with). Compare with `===`/`includes` against names from `listRoles()` — no case folding is needed or done.

**Backend**

| What | Type | Contract |
|---|---|---|
| `req.user.roles` | `string[]` | Role names the user holds. Always an array — `[]` for a user with no role, never `undefined`. Order is not meaningful. Read fresh from the database on every authenticated request (not from the JWT), so a role change applies immediately. |
| `await coreApi.getUserRoles(req.user)` | `Promise<string[]>` | The same names. If `req.user.roles` is already there it is returned as-is (a copy, no query); if the object only has an `id`, the roles are read from the database; no user / no id → `[]`. Never `undefined`. A database failure rejects like any query. |

`req.user` exists **only after `coreApi.authenticateToken` (or `authenticateTokenFromHeaderOrQuery`) has run on that route**. Core's access gate in front of your router does *not* do it for you: for a module nobody has restricted it lets every request straight through (your public routes keep working) without attaching a user. So every route that needs the user must have `coreApi.authenticateToken` on it; on a route without it, `req.user` is `undefined` and `getUserRoles(undefined)` returns `[]`. `authenticateToken` rejects with `401` when there is no valid token. (`GET /api/modules` uses a separate optional-auth middleware that only sets `req.user` when a token is sent; it is not handed to modules.)

**Frontend**

| What | Type | Contract |
|---|---|---|
| `useAuth().userData` | `{ user, roles }` or `null` while logging in / logged out | `user` is the profile object; `roles` is **`[{ role: 'facility_user' }, ...]`** — an array of objects with the name in `.role`. Do not use it for comparisons. |
| `coreApi.useRoles(userData)` | object (below) | A plain function (no React hooks inside — safe to call anywhere, though you will normally call it in a component with `userData` from `useAuth()`). Accepts `null`/`undefined` and returns empty/false values. |
| `coreApi.useRoles(userData).roles` | `string[]` | The user's role names, exact spelling as `listRoles()`, de-duplicated, `[]` when none — never `undefined`. This is the frontend counterpart of `getUserRoles`. |
| `.hasRole(name)` | `(string) => boolean` | The user holds that role (case-insensitive). |
| `.hasAnyRole(names)` / `.hasAllRoles(names)` | `(string[]) => boolean` | Any / all of the given roles held. `false` for a non-array. |
| `.canManageUsers` | `boolean` | `true` when the user holds at least one of `developer`, `administrator`, `facility_manager`, `facility_staff`. This is the flag core's own Users & Groups and Preferences pages use. It is a convenience for core's own pages, not a statement about your module. |

**Developer "view as role / view as user" (sidebar):** it does not change any of the above. `req.user.roles`, `userData.roles` and `useRoles(...)` always report the **real** logged-in user. The mode only changes which *core* pages core itself shows (Page Access evaluation in the shell). It also does not simulate module access (see `MFF-Core_module-access.md`).

**Example — show a tab only if the user holds one of the roles stored for it**

```js
// frontend
function MyModule() {
  const { userData } = coreApi.useAuth();
  const { roles } = coreApi.useRoles(userData);            // ['facility_user']
  const grants = { run: ['facility_user'], results: ['facility_user'], admin: ['facility_manager'] };
  const visibleTabs = Object.keys(grants).filter((tab) => grants[tab].some((r) => roles.includes(r)));
  // -> ['run', 'results']
}

// backend
router.get('/my-tabs', coreApi.authenticateToken, async (req, res) => {
  const roles = await coreApi.getUserRoles(req.user);      // ['facility_user']
  const visibleTabs = Object.keys(grants).filter((tab) => grants[tab].some((r) => roles.includes(r)));
  res.json({ tabs: visibleTabs });
});
```

Who may open the module at all remains core's decision (Modules → Access); this only answers "which of my own tabs does this user get".

### `useModuleAccess(moduleId)` — core's own version of the pattern above

```js
const { loading, visible, pages, unreachable } = coreApi.useModuleAccess('yourmodule');
```

If you'd rather core's admin manage per-page access for you (Setup → Preferences → 🔐 Access) instead of rolling your own role table like the example just above, declare your pages in `module.manifest.json`'s `"pages"` (`docs/MFF-Core_module-manifest.md`) and read the result here:

| Field | Type | Meaning |
|---|---|---|
| `loading` | `boolean` | `true` until `GET /api/modules` has been answered by the backend. Render nothing (or a placeholder) rather than guessing while this is true. It also stays `true` while the backend cannot be reached (see `unreachable`). |
| `visible` | `boolean` | Whether *you* (this module) are even in the list for this user — mirrors whether your whole module is shown at all. `false` means only that the backend's real list lacks your module (not installed, or no access for this user); it is not an answer while `loading` is `true`. |
| `pages` | `string[] \| null` | `null` = nothing restricted, show every page you declared. An array = only these declared keys are allowed — hide tabs/routes for anything not in it. `[]` while `loading`. |
| `unreachable` | `boolean` | `true` while `GET /api/modules` cannot be reached (the backend is restarting after an Install/Update, or the request timed out). `loading` is `true` at the same time. |

**Backend unreachable:** core retries by itself (first after 2 s, the wait doubling up to 30 s) and the hook updates when the server is back, so a module needs no code for it: an unreachable backend never shows up as `visible: false`. Re-fetches automatically on login/logout (same cache-invalidation the rest of the module list uses) — you don't need to do anything on auth changes yourself. This only decides what to render; the actual enforcement is `coreApi.requirePage(pageKey)` on your own backend routes (`docs/MFF-Core_module-core-api.md`'s backend section) — never rely on this hook alone to protect anything.

**`visible: true` with `pages: []`** (module access granted, but none of your declared pages are) is a real, legitimate state, not an error — there's no core-provided screen for it. This is the same situation as the role-based tab-visibility example further up this doc: core decided you're allowed to open the module at all, and from there *which of your own pages to show* (including "none, so say something") is your module's own UI, the same way an empty `visibleTabs` already was before this hook existed. Render your own message for it.

```js
function MyModule() {
  const { loading, pages } = coreApi.useModuleAccess('yourmodule');
  if (loading) return null;
  const canSeeResults = pages === null || pages.includes('results');
  // ...
}
```

### `getAllRoles()`

```js
const roles = await coreApi.getAllRoles(); // string[]
```
Thin wrapper over `GET /files/available-roles` (the endpoint's own path, kept as-is — not worth an unrelated rename/migration just for this). You could call `coreApi.apiGet('/files/available-roles')` yourself; this just saves you from needing to know that specific URL.

### `getAllUsers(params)`

```js
const { items, total } = await coreApi.getAllUsers({ search: 'smith', sortBy: 'username' });
```
Thin, full pass-through to `GET /users` — same admin-only role gate, same pagination/search/sort query params that endpoint already supports (`page`, `limit`, `sortBy`, `sortDir`, `search`). It is **not** a new capability — a module could already reach this exact endpoint via `coreApi.apiGet('/users', { params })`; this is purely a named, documented shortcut. It inherits whatever the *current* logged-in user is actually allowed to see — a non-administrator calling this gets the same 403 core's own UI would get.

### `usePageState`: page state persistence

```js
const [tab, setTab, resetTab] = coreApi.usePageState('mymodule.jobs.tab', 'list', ['list', 'new']);
coreApi.clearPageState('mymodule.jobs.draft');
```
`coreApi.usePageState(key, defaultValue, validate?)` returns `[value, setValue(next | fn), reset]`. It keeps the value per user on the server, so it survives leaving the page, a reload and a logout. `validate` is an array of allowed values, a function `(value) => boolean`, or `{ validate }`; a saved value that fails it silently falls back to the default. `coreApi.clearPageState(key)` forgets a key.

**Keys:** named `moduleId.page.what[:id]`, using the characters `A-Za-z0-9 _ . : -`, at most 120. A value is JSON, at most 64 KB. At most 1000 keys per user. Never store secrets.

**Rule:** every page and tab of a module persists the selected tab or sub-tab; the selected year, project, target, job or item; filters, search, sort and columns; open and collapsed sections; and unsaved drafts (call `reset()` after saving a draft). A module must use `coreApi.usePageState` for this and test leave, return and reload. The HTTP routes behind it are in `docs/MFF-Core_api.md` (`/page-state`).

## Database credentials also go through `getModuleConfig()` now

`moduleDatabaseProvisioner.js` writes a module's DB credentials into `App_backend/.env` too, namespaced exactly like any other module config key (`MODULE_<ID>_DB_HOST`/`_USER`/`_PASSWORD`/`_NAME`) — no module-local `.env` anywhere, no special case in `getModuleConfig()` itself, since it already returns everything under a module's own namespace. A module's `backend/config/database.js` gets them the same way it gets any other config value:

```js
const { DB_HOST, DB_USER, DB_PASSWORD, DB_NAME } = coreApi.getModuleConfig();
```

## Dashboard tab order (`position` and `MODULE_<ID>_DASHBOARD_POSITION` are ignored)

Dashboard tabs, and the left-sidebar entries after the fixed "Dashboard" entry, are listed alphabetically by their displayed label (case-insensitive; a leading emoji is ignored). The `position` a module passes to `registerDashboardTab()` and the admin override `MODULE_<ID>_DASHBOARD_POSITION` in `App_backend/.env` are still accepted, but ignored: neither changes the order. A module does not need to set either, and does not need to do anything else to be ordered.

## Charts: the shared drill-down pie

A module never copies a pie chart, it receives one from core.

Frontend:

```jsx
const { DrillDownPie, fetchSlices } = coreApi;

<DrillDownPie
  title="Spend by supplier"
  loadData={() => fetchSlices(coreApi.api, '/finance/stats/by-supplier', { year })}
  reloadKey={year}
  drillDown
/>
```

**Slice shape:** `{ key, name, value, color?, children?, meta? }`. `value` must be > 0; zero and negative slices are dropped.

**Props:**
- `title`.
- `data` (a static array) **or** `loadData(path)` → slices. `path` is the list of slices drilled into so far; one function serves every level.
- `reloadKey` — change it to reload.
- `drillDown` — `true`, or `(slice, path) => bool`. A click goes into the slice's `children`, or calls `loadData([...path, slice])`.
- Linked mode: `onSelect(slice|null)` and an optional `selectedKey`. A click selects a slice, a second click clears it, and Reset clears it. This is used to let one pie filter another pie or a table.
- `onSliceClick(slice, path)`, `onReset()`, `rootLabel`, `colors`, `theme`, `height`, `outerRadius`, `innerRadius` (makes it a donut), `valueFormatter(v)`, `emptyText`, `loadingText`, `showLegend`, `showLabels`, and `testId` (default `'drilldown-pie'`).

**States:** loading, error (shows the message from the failed request), and empty. There is a tooltip on hover/focus, keyboard support (Enter/Space), and a breadcrumb while drilled in.

**Theming:** the CSS variables `--mff-chart-text`, `--mff-chart-muted`, `--mff-chart-border`, `--mff-chart-bg`, `--mff-chart-tooltip-bg`, `--mff-chart-tooltip-text` and `--mff-chart-error`, or the `theme` prop.

Also available: `coreApi.normalizeSlices(responseBodyOrArray)`.

**Library choice:** plain SVG, no chart library — no new dependency or vulnerability surface, identical in the browser and in jsdom tests, small bundle.

**Backend data shape:** GET endpoints return `{ success: true, data: [ { key, name, value, meta? } ] }`. Build it with:

```js
coreApi.chartResponse(rows, {
  key: 'supplier_id', name: 'supplier_name', value: 'total',
  unknownName: 'Unknown', meta: [...], sort: true
})
```

or use `coreApi.toChartSlices(rows, map)` for the bare array. Numbers and BigInt are normalised, missing names become `unknownName`, zero values are dropped, and the biggest slice comes first.

## Why nothing here is duplicated

`listRoles`/`getAllRoles` are two different functions (backend vs. frontend) but both ultimately resolve to the exact same query (`SELECT role FROM setup_roles`) via `App_backend/src/core/services/rolesService.js` — the backend one calls it directly, the frontend one calls the one HTTP endpoint that also calls it. There is exactly one place this query is written, `GET /files/available-roles` and `coreApi.listRoles()` both use it — this was true before this feature existed for `/files/available-roles`, and stays true now.
