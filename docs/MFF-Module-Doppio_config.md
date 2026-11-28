# doppio module configuration

One config mechanism, no exceptions (`docs/MFF-Core_env-policy.md` is the
authoritative statement of this rule). There is no such thing as
"non-database config" as a separate category here: every setting this
module needs — database credentials included — comes back from the exact
same `coreApi.getModuleConfig()` call, made once in `backend/index.js`'s
`register(coreApi)` and passed down to `backend/config/database.js`
(`DB_HOST`/`DB_USER`/`DB_PASSWORD`/`DB_NAME`) and
`backend/config/doppioConfig.js` (everything else below) alike. This
module never reads a file of its own, never calls `dotenv.config()`/
`dotenv.parse()` on anything of its own — there is exactly one `.env` in
the whole system, `App_backend/.env`, owned by core.

Database credentials need no declaration anywhere — core's provisioner
writes them automatically on Install, per-module, always. Every other key
below is declared in `module.manifest.json`'s `config` array
(`docs/MFF-Core_module-manifest.md`): `SLURM_PARTITION` is `required: true`
(Install/Update's response names it if unset); the rest have working
defaults and are declared only for documentation. An admin sets any of
them by adding `MODULE_DOPPIO_<KEY>=...` directly to `App_backend/.env` and
restarting core's own backend (the same restart Install/Update already
tells them to run) — there is no other place to put a value and no other
step needed for it to take effect.

## What this module does

Runs CCP-EM's `doppio-web` as a SLURM job (submitted the same way ABCFold
submits its own jobs: `coreApi.callModule(token, 'slurm', 'POST',
'/external/jobs', {workDir, jobName})`, `workDir` from
`coreApi.getUserWorkspacePath(req.user)` — see
`docs/MFF-Module-SLURM_slurm-config.md`), then reverse-proxies the user's
browser to that job's live `doppio-web` process for the session's lifetime.

SLURM's own external API never creates, reads, or serves `workDir`'s
contents — this module owns that entirely. `backend/services/doppioSlurm.js`
writes a `submit.sh` into `workDir` that launches `doppio-web` on a free
port, bound to the address of `hostname -f`, waits for its own log to confirm `"Doppio is now
running."`, and writes `host`/`port` to `workDir/doppio-status.json` (or a
`.error` sibling on failure) once ready. `backend/routes/sessionRoutes.js`
polls for that file to learn when a session is actually reachable.

## The installed Doppio and each session's settings (v0.1.54)

Doppio is installed on this machine as `/home/linuxhomes/software/doppio_1.6.0b1/app/doppio-web` (1.6.0b1). The module runs that install; set `MODULE_DOPPIO_BIN` to the full path if the job's `PATH` does not include it, and any compute node must see it.

Doppio reads its settings only from `$XDG_CONFIG_HOME/ccpem/doppio/doppio-config/app-settings.json` (`~/.config` when unset). `submit.sh` therefore gives each session its own config home, `<work_dir>/config`:
- every entry of the job user's config home is linked into it, including `ccpem/pipeliner` (Pipeliner's settings come from the same place), except `ccpem/doppio`;
- the user's `doppio-config` folder (settings, themes, images), if any, is copied, and in the copy `projectDirectory` and `knownProjects` are set to `<work_dir>/project`; other settings the user has are kept.

The user's own Doppio settings and `~/ccpem-project` are not used or changed. `DOPPIO_RUNTIME_DIR` is `<work_dir>/runtime`. If the settings cannot be prepared, the job stops with an error status and Doppio is not started. Each new session starts with an empty Doppio project. `XDG_CONFIG_HOME` is inherited by what Doppio starts. `doppio-web.log` and `submit.sh` in the work folder contain the session's Doppio token.

Tested on 28/11/2026 against this install: the module's `submit.sh`, run directly with bash (not through SLURM) with a throwaway home, and the module's proxy routes with stand-ins for core's login gate, database and projects. Sign-in, the redirect to `ui/`, UI files, API reads and a save from the UI (`PATCH`) worked, also inside an iframe in a browser. Not tested: a real core, a real SLURM job, a separate compute node, running a Doppio job, loading a structure in the 3D viewer.

Doppio's 3D viewer (Moorhen) needs `SharedArrayBuffer`, which requires a cross-origin isolated page. Doppio sends the headers for that, but inside the iframe on core's page they have no effect, so the viewer does not start there. The Doppio tab has an "Open Doppio in a new tab" link (same ticketed URL); opened that way the page is cross-origin isolated. Browsers allow this only over HTTPS or on localhost.

## Projects (v0.1.51)

Every Doppio session runs in a Science project (`docs/MFF-Module-Science_projects.md`). The manifest has `"requires": ["slurm", "science"]` and `"actsAsUsers": true`. Needs core 0.1.227 or later (`coreApi.projects`, `coreApi.ui.ProjectTabs`/`ProjectTask`), and Science with its `/external/template` routes. Not yet tested in a browser against a real core.

- The Doppio page has core's ProjectTabs at the top (page-state key `doppio.sessions`). Each project tab lists the caller's own sessions in that project in a sortable table (`doppio.sessions.table`), with the chosen one shown below (kept per project, `doppio.sessions.selected:<projectId>`). The Launch button sits in ProjectTask; it is hidden while the caller has an active session in that project, and in "Project template". Other members' sessions are not shown.
- Core's browser lock (a person with no writable project) is a screen aid; the server checks every launch itself.
- Launching is refused without a project, with read-only or no access, for an archived project, and for "Project template".
- Each session is a project item: `item_type` `doppio_session`, `item_id` the session id, status the session state, link `/dashboard?tab=doppio`. It is sent again when a request sees a changed state; a `403` or `409` is final for that state, other failures are retried on a later request and shown as a warning. A state change nobody views is not sent.
- A proxy ticket needs write access to the session's project (adopted template sessions excepted). A ticket already in use stops working within about 30 s of its owner losing write access to the project.
- Sessions created before this change are moved into Science's "Project template" by a background job (first run 15 s after start, delay doubling up to 1 h, until none are left), acting as each session's owner; owners core refuses (e.g. inactive or without Doppio access) stay unadopted and are logged. Moved sessions are listed, to their owner only, under the "Project template" tab.
- New columns on `doppio_sessions`: `project_id`, `project_item_state`, `project_is_template` (added by core's schema diffing).

## Config keys

| Variable | Default | Purpose |
|---|---|---|
| `MODULE_DOPPIO_SLURM_PARTITION` | *(none — required)* | `--partition` for every submitted job. |
| `MODULE_DOPPIO_SLURM_GRES` | *(none — optional)* | e.g. `gpu:1`; passed as `--gres` if set. |
| `MODULE_DOPPIO_SLURM_CPUS_PER_TASK` | `4` | `--cpus-per-task`. |
| `MODULE_DOPPIO_SLURM_MEM` | `16G` | `--mem`. |
| `MODULE_DOPPIO_SLURM_TIME` | `04:00:00` | `--time` — see "session lifecycle" below, this is the *only* thing that ends a session today. |
| `MODULE_DOPPIO_LOAD_CMD` | *(none — optional)* | A verbatim shell line injected into `submit.sh` before launching `doppio-web` (e.g. `module load ccp4/doppio`). Admin-configured, trusted string — never taken from a request. |
| `MODULE_DOPPIO_BIN` | `doppio-web` | Override if not on `PATH` after `MODULE_DOPPIO_LOAD_CMD` runs. |
| `MODULE_DOPPIO_WORKDIR_SUBDIR` | `doppio-sessions` | Subfolder under `getUserWorkspacePath(user).path` where each session's own `work_dir` (`<workspace>/doppio-sessions/<sessionId>/`) is created. |
| `MODULE_DOPPIO_STARTUP_TIMEOUT_S` | `120` | How long `submit.sh` itself polls its own log for `"Doppio is now running."` before giving up and writing an error status. |
| `MODULE_DOPPIO_STATUS_POLL_INTERVAL_MS` | `3000` | Rate-limits how often `GET /sessions/:id` re-checks SLURM state and the status file, so frontend polling doesn't hammer either. |
| `MODULE_DOPPIO_STATUS_POLL_TIMEOUT_MS` | `180000` | UI give-up point, not a `scancel`: if a session is still non-terminal this long after `created_at` with no status file or terminal SLURM state ever appearing, `GET /sessions/:id` marks it `failed` client-side (the SLURM allocation itself is untouched). |

## What this module does NOT do yet

- **Cancel a running session.** The SLURM module has no cancel route to
  call (`docs/MFF-Module-SLURM_slurm-config.md` documents `POST
  /external/jobs` and `GET /external/jobs/:jobId` only) — a session's
  `MODULE_DOPPIO_SLURM_TIME` wall-time limit is its only lifecycle bound.
  The frontend (`DoppioTab.js`) deliberately has no cancel/stop button.
- **Auto-detect a GPU partition.** `MODULE_DOPPIO_SLURM_PARTITION`/`_GRES`
  are static, admin-configured defaults — no per-request partition
  selection or auto-detection against SLURM's `GET
  /external/gpu-partitions`.
- **SSH-tunnel to the compute node.** `doppio-web` is bound to the compute
  node's `hostname -f` address and this module's backend proxies to it directly over plain
  HTTP — a deliberate choice (see "Security model" below), even though
  Doppio's own docs suggest SSH port forwarding as their supported remote
  pattern.
- **Offer a disposable/"virtual account" mode.** Every session runs as the
  real logged-in user's own workspace and SLURM identity.

## Security model

- `doppio-web`'s own session cookie is only set "on a secure connection...
  local loopback... or... https"
  (`ccpem.ac.uk/docs/doppio/security.html`). Proxied over plain HTTP from
  this module's backend to a compute node, that condition is never met —
  so this module never relies on or forwards Doppio's own `Set-Cookie`.
  Instead, `backend/routes/proxyRoutes.js` appends `?token=<doppio token>`
  to every request it forwards upstream itself. The browser never learns
  doppio's own token — only this module's backend does, read off the
  `doppio_sessions.token` column.
- The browser instead authenticates to *this module's own* proxy route via
  a short-lived ticket: `POST /sessions/:id/proxy-ticket` (a normal
  authenticated call) mints one, the iframe's initial `src` carries it as
  `?ticket=`, and the proxy route hands back a scoped `httpOnly` cookie on
  first valid use so later sub-resource/AJAX requests from *within*
  doppio-web's own UI (which won't carry that query param) keep working.
  Every validated hit slides the ticket's expiry forward
  (`TICKET_IDLE_TTL_MS`, 20 minutes) — a session idle longer than that
  needs the parent tab to mint a fresh ticket and reload the iframe.

From v0.1.52 the proxy paths are signed in through core's identified routes (core 0.1.233 or later): the manifest lists `"identifiedRoutes": [{ "path": "/sessions/:id/proxy" }]` and the module calls `coreApi.registerIdentifiedRoute`. For every request (all methods) core calls the module's `identify`, which requires the unexpired ticket (`?ticket=` or the HttpOnly, SameSite=Lax, per-session-path cookie) of that session and that its owner still has write on the session's project (adopted template sessions excepted; the answer is cached 30 s, so losing access takes effect within about 30 s); core then checks the account and module access and refuses cross-site requests. No ticket is issued while viewing as another user. The proxy re-sends JSON bodies that core has already parsed, and drops doppio-web's `Set-Cookie` and any `Referrer-Policy: no-referrer`. On an older core the live view does not work.

From v0.1.53, when core reports that a user's sign-in has ended (`coreApi.onUserSignOut`, core 0.1.235 or later: logout, expiry, deactivation, password change, deletion), all of that user's proxy tickets are cleared, on every device; opening the Doppio page after signing in again issues a new one. A response already being streamed is not cut off. On an older core, and for endings core does not report, the 20-minute idle expiry is the limit (it slides while the ticket is in use).

## Routes

### `POST /sessions`

Any authenticated user with an active workspace (`coreApi.getUserWorkspacePath`).
Body: `{ projectId, jobName? }`. Creates the session's `work_dir`, writes `submit.sh`,
submits it via `coreApi.callModule(..., 'slurm', 'POST', '/external/jobs',
...)`. Returns the new session (never `token`). Refused with `400` (no
project), `403` (read-only), `404` (unreadable) or `409` (archived, or the
template). The response may carry `projectWarning`.

### `GET /sessions`

The caller's own sessions, newest first. Optional `?projectId=` (needs read
access on that project, else `404`).

### `GET /sessions/:id`

Owner-only. Rate-limit-polls SLURM job state and the `workDir` status file,
advancing `appState` through `submitting → queued → starting → running →
ended`/`failed`. Never returns `token`, the proxy ticket, or the raw
`host`/`port`. It also updates the project item; the response has
`projectId`, `inTemplate` and `projectWarning`.

### `POST /sessions/:id/proxy-ticket`

Owner-only, session must be `running`. Returns `{ ticket, expiresAt }`. Needs
write access on the session's project (`403`), except for adopted template
sessions. `403` while viewing as another user.

### `ALL /sessions/:id/proxy/*`

Signed in by core's identified route (see "Security model" above): it is not
publicly open, and not behind a core login header (an `<iframe src>` can't
send an `Authorization` header). `401` for a missing, invalid or expired
ticket, or lost write access to the project; `409` if the session is not
running. Reverse-proxies to the session's `doppio-web` process.

### `GET`/`PUT /page-access`

Unchanged, generic role-gated tab-visibility admin (`doppio_page_access`).

## Open items — verify against a real cluster + real `doppio-web` install

Not guessed as fact anywhere in this module's code — flagged for whoever
next has access to a real deployment:

1. WebSockets: none seen with 1.6.0b1 (no `WebSocket`/`EventSource` in its UI files from a text search, and none needed in the pages tried). Job running was not tried.
2. The ready line `Doppio is now running.` is confirmed for 1.6.0b1 (ready in about 5 s on this machine); other versions unchecked, and so is start-up time on a real compute node.
3. Whether `hostname -f` on a real compute node resolves to something
   reachable from the app server over plain HTTP — the whole "no SSH
   tunnel" model assumes this.
4. Whether the shared workspace path is mounted read/writable from the app
   server process itself (`pollStatusFile()` reads it directly), not just
   from compute nodes — ABCFold's own result-reading already assumes this,
   but worth confirming per-deployment.
5. Doppio sends no `X-Frame-Options` or CSP, and its UI loads in a same-origin iframe; whether core's own pages add a policy that blocks it is not yet checked.

## `coreApi` gaps flagged, not routed around

Per this repo's own `CLAUDE.md`: a capability this module needs that isn't
in `coreApi` is a gap to flag, never a reason to reach into `mff-core`
directly. Recorded here for anyone considering a future `coreApi` change
(item 2 is no longer a gap):

1. **No generic reverse-proxy/streaming primitive.** `callModule` is
   single-shot request/response by design. This module doesn't need one —
   the live-proxy hop is built entirely inside its own Express router,
   never through `callModule` — but a first-class "proxy a live sub-service
   for a session's duration" primitive would spare future modules from
   hand-rolling the ticket/cookie/token-injection scheme above.
2. **WebSocket forwarding is not implemented here.** Core now provides
   `coreApi.registerUpgradeHandler` (`docs/MFF-Core_module-core-api.md`,
   "WebSocket"), so this is no longer a core gap. If `doppio-web` turns out
   to need WebSockets (none seen so far), this module must add `"websocket": true` to
   its manifest and register an upgrade handler that authenticates with its own
   proxy ticket (`identify`) and forwards to the session; nothing of that is
   built or tested.
