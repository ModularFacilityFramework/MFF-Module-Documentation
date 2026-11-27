# pandda2inspect module configuration

All of this module's config — database credentials included, no
exceptions, no separate mechanism — comes from one call,
`coreApi.getModuleConfig()` (see `docs/MFF-Core_module-core-api.md` and
`docs/MFF-Core_env-policy.md`), reading core's one `App_backend/.env`,
namespaced `MODULE_PANDDA2INSPECT_<KEY>`. `backend/config/database.js`
and `backend/config/pandda2InspectConfig.js` both call it the same way;
neither reads a file of its own. `DB_HOST`/`DB_USER`/`DB_PASSWORD`/`DB_NAME`
come back from the exact same object as `APP_DIR`/`GITLAB_URL`/`GITLAB_TOKEN`
— core's provisioner writes the database ones automatically on Install
(every module gets its own database, unconditionally, nothing to declare
for that part); the other keys are this module's own, declared in
`module.manifest.json`'s `config` array (see
`docs/MFF-Core_module-manifest.md`) so Install/Update names any
still-unset required value directly in its response instead of the
module silently not working. An admin still has to add the actual value
to `App_backend/.env` themselves for anything not database-provisioned or
core-generatable, and restart core's own backend for this deployment for
it to take effect (the same restart already shown after any
Install/Update).

## The embedded app is served by this module itself, not an external URL

This module serves the embedded app's own build directly
(`backend/index.js`, `express.static()` mounted at `/app` under this
module's own route prefix) — every user's browser loads the same static
bundle from there, the same way any client-side web app (including
mff-core's own frontend) already works. There is **no**
`MODULE_PANDDA2INSPECT_APP_URL` — the frontend computes the iframe `src`
itself (`frontend/components/PanddaInspectTab.js`), off the module's own
already-known API base and id.

| Variable | Default | Purpose |
|---|---|---|
| `MODULE_PANDDA2INSPECT_APP_DIR` | *(none — required for the embedded-app tab and the Update action)* | Absolute local path, on the host this backend runs on, to the embedded app's existing git checkout. Doubles as: where `POST /deploy` runs `git pull`/build, and what `backend/index.js` serves `<APP_DIR>/dist` from at `/app`. |
| `MODULE_PANDDA2INSPECT_DECISIONS_DIR` | *(none — required for saving submissions)* | Absolute local path, on the host this backend runs on, to the folder each submitted decision is saved in. Created if it doesn't exist; the backend's user must be able to write there. |
| `MODULE_PANDDA2INSPECT_GITLAB_URL` | *(none — optional)* | Only needed if the checkout's own git remote/credentials aren't already enough for a plain `git pull` (e.g. a token-authenticated HTTPS remote). Paired with `MODULE_PANDDA2INSPECT_GITLAB_TOKEN`. |
| `MODULE_PANDDA2INSPECT_GITLAB_TOKEN` | *(none — optional)* | Used with `MODULE_PANDDA2INSPECT_GITLAB_URL` to build an authenticated pull URL, passed as a single argv element to `git pull` — never interpolated into a shell string, never logged (redacted from the deploy log/response). |

## What this module does NOT do

- **Deploy the embedded app for the first time.** `MODULE_PANDDA2INSPECT_APP_DIR`
  must already be an existing, working git checkout before the Update
  action is used — this module only runs `git pull` + a fixed build
  sequence against it, never the initial `git clone`.

## Restricting the setup tab

`module.manifest.json` declares one page, `setup` (the "pandda2.inspect
setup" tab), without `openByDefault`. With nothing granted, only
developers and administrators (plus `SUPERUSER_ROLE`, if set) can use it,
and they keep access whatever is granted, so a grant cannot lock any of
them out (see `MFF-Core_module-access.md`). Its routes (`/branches`,
`/deploy`, `/deploy/status`, `/decisions/download`) check
`coreApi.requirePage('setup')` and also require the developer or
administrator role. Granting the page to anyone else at Setup →
Preferences → 🔐 Access → Pandda2-inspect → Pages therefore does not let
them use those routes: the role check still refuses them. `frontend/index.js`
registers the tab with `pageKey: 'setup'`, so core hides it from anyone
without access to the page (needs core v0.1.144+).

## Who can load the app's files

Core requires a login on every module route except those in the manifest's `publicRoutes`, and an iframe can't send the login header. So the manifest lists `GET /app/*` as public and this module checks those requests itself (`backend/services/appTicket.js`): every file under `/app` needs a valid signed cookie, otherwise `401`. Only `GET /config`, which needs a login and passes core's module access check, sets it: valid for one hour, `HttpOnly`, `SameSite=Strict`, sent only to the module's `/app/` path, and `Secure` when the request is https (a proxy must send `X-Forwarded-Proto`). The inspection tab renews it every 10 minutes. The signing key is made anew at each backend start. A user who logs out or loses access can still load the app's files from that browser until the last cookie expires (at most an hour); saving a decision always needs a login.

## Inspections belong to projects

The manifest has `"requires": ["science"]` and `"actsAsUsers": true`, so Science must be installed before this module is installed or updated. It needs core v0.1.227 or later. Core's `ProjectTabs` sits at the top of the inspection tab, one tab per project the user can see; the chosen tab is remembered per user. A tab shows that project's inspections (who saved each one, when, and the file name, in a sortable table whose sort is remembered) and, only where the user can add to the project (not read-only, archived or the Project template), the app. The app reloads when the project changes, so anything unsaved inside it is lost then. Each saved decision is one inspection: a row in `pandda2inspect_inspections` (project, file name, who saved it) and a project item in Science (`item_type` `inspection`, link `/dashboard?tab=pandda2inspect`).

Decisions saved before this go into Science's "Project template", found through Science's route, never by title. It runs about 15 seconds after the backend starts, retries until it succeeds (calling Science as an administrator or developer), and only takes files older than the backend's start. The author comes from the file name; if no single user matches, the item has no author and only the template's owner and administrators see it (`saved_by` 0). New work cannot be saved into the template.

## `GET /config`

Any authenticated user who passes core's module access check. Returns
`{ deployEnabled, appPath }`. `deployEnabled` is `true` only when
`MODULE_PANDDA2INSPECT_APP_DIR` is set (and therefore `/app` is actually
mounted and serving something). When the app is configured, it also sets the
signed cookie described in "Who can load the app's files".

## `POST /decisions`

`POST /decisions?projectId=<id>`, for a user who can write to that project (checked with `coreApi.projects.access`). Saves the JSON body as a new file in `MODULE_PANDDA2INSPECT_DECISIONS_DIR`, named `<username>_<timestamp>.json` (e.g. `jsmith_2026-11-05T13-40-12-345Z.json`), records its project and lists it in Science. Returns `{ fileName, inspectionId, projectId, registered, registrationError }`; `registered: false` means the decision is saved but is not listed in Science, and the tab shows a warning. Nothing is ever overwritten. Characters unsafe in a file name are replaced with `_` in the username, since core doesn't restrict them. Limited to core's global 1 MB JSON body size. `400` if the setting isn't configured or no project is given, `404` for a project the user can't read, `403` for one they can only read, `409` for an archived project or the Project template, `503` if projects aren't available.

The embedded app itself knows nothing about this: it still tries to download
`decision.json`, and `frontend/components/catchDecisionDownload.js` stops that
download inside the iframe and posts the file here instead. It depends on the
app's download keeping that file name — if the app renames it, the download
reaches the user's browser again and nothing is saved. If saving fails, the
tab shows an error above the app.

## `GET /inspections`

`GET /inspections?projectId=<id>`, for a user who can read that project. Returns `{ ownOnly, inspections: [{ id, fileName, savedBy, savedAt }] }`, newest first; `savedBy` is "First Last (username)", or empty for an unknown author. In the Project template a user is shown only their own, unless they own the template or are an administrator or developer; if core doesn't say whether a project is the template, it is treated as one. `400` without a project, `404` for a project the user can't read, `501` on a core without projects. It lists records only; it gives no access to the decision files.

## `GET /decisions/download`

`developer`/`administrator` only — the setup tab's "Download all
decisions" button. Every `.json` file in
`MODULE_PANDDA2INSPECT_DECISIONS_DIR` in one zip
(`pandda2inspect_decisions_<timestamp>.zip`, UTC, e.g.
`pandda2inspect_decisions_2026-11-10T15-20-03-123Z.zip`), built by `backend/services/zip.js`
rather than a zip dependency. `404` if no decision has been saved yet,
`400` if the setting isn't configured.

## `POST /deploy`

`developer`/`administrator` only. Starts an update in the background and
answers `202` straight away (`409` if one is already running, `400` if
`MODULE_PANDDA2INSPECT_APP_DIR` isn't set at all). Runs, in order, in
`MODULE_PANDDA2INSPECT_APP_DIR`: `git fetch` + checkout + fast-forward of
the branch (with an authenticated URL only if
`MODULE_PANDDA2INSPECT_GITLAB_URL`/`_TOKEN` are set), `npm ci`,
`node_modules/.bin/vite build --base=<computed>` — stopping at the first
failing step. `npm ci` gets a 20 minute timeout, `vite` 4 hours (it copies
the app's whole `public/`, gigabytes of dataset images, into `dist`). There is no
`tsc -b` step: it failed on harmless unused imports in the app's own source
and doesn't change the bundle. The `--base` value is computed fresh each
run as `coreApi.apiBasePath + mountPath + '/app/'`, so the same build
self-corrects for whatever this deployment's actual `API_BASE_PATH` is —
it's baked into the built JS/CSS/asset URLs by Vite, and has to exactly
match where `backend/index.js` actually serves the result from.

Every process is spawned via `child_process` `spawn`/`spawnSync` with a
fixed argv array, **never** a shell string; there is no admin-configurable command
text (an earlier version had one and was reverted — see git history —
because a raw command string handed to a shell is the same vulnerability
class as a command-injection bug this codebase's own security audit
already found and fixed in `emailService.js`'s `sendLocalMail`).
`appGit.js` also strips `NODE_ENV` from the spawned processes' environment
— inheriting `NODE_ENV=production` from core's own `App_backend/.env`
made `npm ci` silently skip `devDependencies` (confirmed live: broke the
embedded app's own `typescript`/`vite` install, `tsc: not found`).

Every attempt — success or failure — is logged to
`pandda2inspect_deploy_log` (who triggered it, success, output, when).

## `GET /deploy/status`

`developer`/`administrator` only. The running or most recent update:
`{ running, success, output, step, stepSeconds, totalSeconds }`, with the
combined stdout/stderr so far, token redacted — what the setup tab's debug
window polls. In memory only, so a backend restart clears it (`running:
false`, `success: null`, empty `output`).

Note: triggering a rebuild from an HTTP route is a deliberate departure
from core's own convention (core only ever surfaces a rebuild/restart as a
copy-pasteable command an admin runs themselves — see
`docs/MFF-Core_module-database-provisioning.md` and
`modulesManagementRoutes.js`'s own header comment). This module's owner
made that call knowingly, for a fast remote-dev workflow — it is not a
pattern core was asked to endorse or that any other module should assume
is sanctioned.
