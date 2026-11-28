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
writes a `submit.sh` into `workDir` that launches `doppio-web` bound to
`0.0.0.0` on a free port, waits for its own log to confirm `"Doppio is now
running."`, and writes `host`/`port` to `workDir/doppio-status.json` (or a
`.error` sibling on failure) once ready. `backend/routes/sessionRoutes.js`
polls for that file to learn when a session is actually reachable.

## Projects (v0.1.51)

Every Doppio session runs in a Science project (`docs/MFF-Module-Science_projects.md`). The manifest has `"requires": ["slurm", "science"]` and `"actsAsUsers": true`. Needs core 0.1.227 or later (`coreApi.projects`, `coreApi.ui.ProjectTabs`/`ProjectTask`), and Science with its `/external/template` routes. Not yet tested in a browser against a real core.

- The Doppio page has core's ProjectTabs at the top (page-state key `doppio.sessions`). Each project tab lists the caller's own sessions in that project in a sortable table (`doppio.sessions.table`), with the chosen one shown below (kept per project, `doppio.sessions.selected:<projectId>`). The Launch button sits in ProjectTask; it is hidden while the caller has an active session in that project, and in "Project template". Other members' sessions are not shown.
- Core's browser lock (a person with no writable project) is a screen aid; the server checks every launch itself.
- Launching is refused without a project, with read-only or no access, for an archived project, and for "Project template".
- Each session is a project item: `item_type` `doppio_session`, `item_id` the session id, status the session state, link `/dashboard?tab=doppio`. It is sent again when a request sees a changed state; a `403` or `409` is final for that state, other failures are retried on a later request and shown as a warning. A state change nobody views is not sent.
- A proxy ticket needs write access to the session's project (adopted template sessions excepted). A ticket already in use keeps working while it stays active.
- Sessions created before this change are moved into Science's "Project template" by a background job (first run 15 s after start, delay doubling up to 1 h, until none are left), acting as each session's owner; owners core refuses (e.g. inactive or without Doppio access) stay unadopted and are logged. They are listed under the "Project template" tab.
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
- **SSH-tunnel to the compute node.** `doppio-web` is launched bound to
  `0.0.0.0` and this module's backend proxies to it directly over plain
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
sessions.

### `ALL /sessions/:id/proxy/*`

Not behind `coreApi.authenticateToken` (an `<iframe src>` can't send an
`Authorization` header) — ticket/cookie-authenticated instead, see
"Security model" above. Reverse-proxies to the session's `doppio-web`
process.

### `GET`/`PUT /page-access`

Unchanged, generic role-gated tab-visibility admin (`doppio_page_access`).

## Open items — verify against a real cluster + real `doppio-web` install

Not guessed as fact anywhere in this module's code — flagged for whoever
next has access to a real deployment:

1. `app-settings.json`'s exact schema and `doppio-web`'s lookup path/precedence
   for it. `doppioSlurm.js`'s `buildAppSettings()` writes a best-guess
   `{"project_directory": "<workDir>/project"}` — doppio-web's own docs
   only describe it as "the current project setting", without documenting
   the key name or lookup path precisely.
2. Whether `doppio-web`'s Uvicorn stack needs WebSockets for any live UI
   update. This module's proxy is HTTP-only today and registers no upgrade
   handler.
3. Stability of grepping `"Doppio is now running."` in `doppio-web.log`
   across `doppio-web` versions, and whether the 120s default startup
   timeout is realistic.
4. Whether `hostname -f` on a real compute node resolves to something
   reachable from the app server over plain HTTP — the whole "no SSH
   tunnel" model assumes this.
5. Whether the shared workspace path is mounted read/writable from the app
   server process itself (`pollStatusFile()` reads it directly), not just
   from compute nodes — ABCFold's own result-reading already assumes this,
   but worth confirming per-deployment.
6. Whether core's own frontend has any CSP/iframe-sandboxing that would
   block this same-origin-different-path iframe.

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
   "WebSocket"), so this is no longer a core gap. If open item 2 shows
   `doppio-web` needs WebSockets, this module must add `"websocket": true` to
   its manifest and register an upgrade handler that authenticates with its own
   proxy ticket (`identify`) and forwards to the session; nothing of that is
   built or tested.
