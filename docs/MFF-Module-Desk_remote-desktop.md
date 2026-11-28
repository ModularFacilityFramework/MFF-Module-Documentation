# Desk module: Remote Desktop

Module `desk` (mount `/desk`, version 0.1.18). It gives a Remote Desktop session through an existing Apache Guacamole, without the user ever seeing Guacamole's login. It is private: see "Who may use it".

## What the user sees
One left-hand menu entry, "Desktop" (`registerMenuPage`, path `/desk`). It is the only way in: there is no Dashboard tab. Core orders menu entries alphabetically by label (`docs/MFF-Core_module-core-api.md`, "Dashboard tab order").

- Clicking "Desktop" in the left-hand menu does two things: the remote desktop opens straight away in a new browser tab, which gets the focus, and the app shows Desk's own page.
- Every click opens a new tab. Reloading the page, typing its address or using back/forward opens none.
- The page keeps an "Open Remote Desktop" button (also for when the browser blocks the pop-up).
- Opening on the click needs mff-core 0.1.237 or later (`registerMenuPage` `onSelect`). On an older core only the button works.
- The page has two tabs, "Configuration" and "Health". The chosen tab, unsaved settings, the number of connections shown, the table sort and the open connection are remembered per user.

## Configuration tab
Editable. A change applies from the next desktop opened.

- **The connection to open directly:** picked from the connections Guacamole lists for the service account. The default is Guacamole's own start page. A saved connection that no longer exists falls back to the start page.
- **How long a desktop sign-in lasts:** 1–8 hours, default 8.
- **How long to watch for the first full frame:** 10–120 seconds, default 30.

Read-only on the page: the Guacamole address, the guacd address, whether the account and password are set (never their values) and the allowed accounts. These stay in core's `.env`. The address is deliberately not editable from the browser, because Desk sends the Guacamole password to it.

## Health tab
### Chain status
- Service states from systemd (guacd, Tomcat, xrdp, xrdp-sesman), only when Guacamole runs on the app server.
- Whether Guacamole's web app answers.
- Whether guacd's port accepts a connection. This is checked at most once a minute; each check leaves a line in guacd's log.
- A "Test Guacamole sign-in" button. Its result is reused for a minute.

### Connections
The last 10, 25, 50 or 100 connections, in a sortable table with the slowest step highlighted. The last 500 connections are kept.

The steps, in order:
1. **MFF sign-in:** core's check of the desktop tab's requests.
2. **Guacamole page:** the tab loading Guacamole's client.
3. **Guacamole token:** the service-account sign-in.
4. **Open tunnel:** Guacamole's client starting the connection, including any time on its connection list.
5. **guacd:** from the tunnel being opened until guacd is ready.
6. **RDP/xrdp connect:** from guacd being ready until the first picture.
7. **Attach + first frame:** an ESTIMATE. It runs from the first picture until the biggest repaint that follows, normally the desktop arriving after the RDP server's login screen.

- Watching stops after the set time, at the first key press or mouse click, or when the connection closes.
- A row's details show when picture data arrived and the longest wait.
- Timings come from what passes through Desk's own proxy and WebSocket relay. Nothing is added to Guacamole or the RDP machine.
- If the WebSocket tunnel does not work and Guacamole falls back to its HTTP tunnel, the RDP and first-frame steps cannot be timed.

### Logs for a connection
- The logs of guacd, Guacamole's web app (Tomcat), xrdp and xrdp-sesman, limited to that connection's time. A file with no recognisable times shows its last lines instead.
- Desk reads only what the backend's own account may read, and says which logs it cannot read.
- Typical need: permission to read xrdp's log files and the system journal (often the `adm` group, which also opens other system logs), and read access to Tomcat's log file if it writes one.
- The xrdp logs read are the app server's own, so they only help if the remote desktop runs on it.
- Passwords and tokens in log lines are masked.

## Who may use it
Only the accounts on Desk's own list of allowed users (`ALLOWED_USERS`, see "Configuration") can use it. Such an account also needs the `developer` role and core's module access.

- Every Desk route, `/proxy` and the WebSocket tunnel answer `404` to every other account, developers and administrators included.
- "View as" and background or delegated calls from other modules are refused too. "View as" gets `404`.
- Desk registers its allowed-user check with `coreApi.registerAccessRestriction` (`docs/MFF-Core_module-core-api.md`, "Limiting a module or page to named accounts"). On mff-core 0.1.234 or later, every account not on the list (developers, administrators, `SUPERUSER_ROLE` and "view as" included) gets no "Desktop" menu entry, and a `404`.
- On an older core those accounts still see the menu entry; Desk still refuses them itself.

**Limitations:**
- Developers and administrators still see Desk under Preferences → Modules and Access, marked with a warning, and can update or remove it.
- Whoever can change accounts (for example reset the allowed account's password), edit `App_backend/.env`, change module code, reach the server, or reach Guacamole directly can get round the list.

## How a session starts
1. Opening the desktop sends `POST /api/desk/session` (`authenticateToken` and `checkRole` `developer`). It refuses "view as" identities.
2. The backend mints an in-memory UUID token (it lives as long as a desktop sign-in lasts, see "Configuration tab"; lost when the backend restarts) and returns `openUrl`.
3. The browser opens `openUrl` in a new tab, at `/api/desk/proxy/?token=...`.
4. The proxy sets an HttpOnly, SameSite=Strict cookie `desk_rd_session` holding that token (its path is the proxy mount).
5. The proxy intercepts Guacamole's `POST /api/tokens` and logs in server-side with a service account, so users never see Guacamole's login.
6. It reverse-proxies everything else to Guacamole (`http-proxy-middleware`).

## Proxy sign-in
- The manifest has `"identifiedRoutes": [{"path": "/proxy"}]`. `/proxy` is signed in through `coreApi.registerIdentifiedRoute` with desk's own token (`?token=` on the first hit) or the `desk_rd_session` cookie (`docs/MFF-Core_module-core-api.md`, "Module's own sign-in").
- Core checks module access on every request.
- The `developer` role is required on every proxy HTTP request and on the WebSocket tunnel.
- The desk token is no longer accepted on `/proxy` as `Authorization: Bearer`.

## When the user's sign-in ends
- With mff-core 0.1.235 or later, Desk registers `coreApi.onUserSignOut` (`docs/MFF-Core_module-core-api.md`, "When a user's sign-in ends"). A logout (even with an expired access token), a run-out sign-in that the browser tries to renew, deactivation, a password reset or deletion revokes all of that user's desk sessions, on every device, and closes their open WebSocket tunnels and in-flight proxy requests.
- Not covered, so the desktop sign-in time limit applies (see "Configuration tab"): a sign-in that runs out while the browser never returns, changes made straight in the database, and events that happen while the backend is down (a backend restart drops all desk sessions anyway).
- On an older core, logging out of the app does not end a desk session before it expires.
- Removing the `developer` role does not cut an already-open tunnel; it applies at the next connect.

## Configuration
Declared in the manifest, read through `coreApi.getModuleConfig()`; in `App_backend/.env` they are `MODULE_DESK_*`. These are required:
- `GUAC_HOST`
- `GUAC_PORT`
- `GUAC_CONTEXT_PATH`
- `GUAC_USERNAME`
- `GUAC_PASSWORD`

Optional:
- `ALLOWED_USERS` (`MODULE_DESK_ALLOWED_USERS`): comma-separated MFF usernames, not case-sensitive. If it is not set, the built-in default applies: the owner only (`namlb`). If it is set but empty, nobody can use Desk. A change needs a backend restart.

More optional keys, all `MODULE_DESK_*`:

| Key | Default |
|---|---|
| `MODULE_DESK_GUACD_HOST` | the value of `GUAC_HOST` |
| `MODULE_DESK_GUACD_PORT` | guacd's standard port |
| `MODULE_DESK_GUACD_SERVICE` | `guacd` |
| `MODULE_DESK_TOMCAT_SERVICE` | `tomcat9` |
| `MODULE_DESK_GUACD_LOG_FILE` | the guacd service's journal |
| `MODULE_DESK_TOMCAT_LOG_FILE` | the Tomcat service's journal |
| `MODULE_DESK_XRDP_LOG_FILE` | `/var/log/xrdp.log` |
| `MODULE_DESK_XRDP_SESMAN_LOG_FILE` | `/var/log/xrdp-sesman.log` |

## Database
- Two tables in Desk's own database: `desk_settings` and `desk_connections` (step timings per connection). Desk session tokens are still only in memory.
- Installing or updating Desk creates the database (core's database provisioning must be set up) and needs a backend restart.
- If the database cannot be used, the page says so and keeps settings and history in memory until the restart.

## Verification status
Version 0.1.18 has been run only against a mock core, a fake Guacamole and a scratch MariaDB, not a real core, a real Guacamole or a real browser.

## Setup outside the module
- A running Guacamole (guacd and the web app), reachable from the backend.
- A service account in Guacamole that has the connection(s) assigned.
- The connection's RDP target (for example xrdp) configured in Guacamole.

## WebSocket tunnel
Desk handles Guacamole's WebSocket tunnel through `coreApi.registerUpgradeHandler('/proxy/websocket-tunnel', handler, { identify })` (`docs/MFF-Core_module-core-api.md`, "WebSocket"). The manifest has `"websocket": true`, and the module depends on `ws`.
- The `identify` callback reads the HttpOnly `desk_rd_session` cookie (the per-session token minted by `POST /session`; it expires with the desktop sign-in). Guacamole's own `token` query parameter therefore passes through untouched.
- Core re-checks the user's real identity and module access on every connect.
- The tunnel also requires the `developer` role on every connect and reconnect, the same as the HTTP `/proxy` routes: `403`, or `503` if the role lookup fails. An already-open tunnel is not cut when the role is revoked; the check applies at the next connect.
- The backend relays frames to Guacamole's `/websocket-tunnel` with the `guacamole` subprotocol.
- `http-proxy-middleware`'s own `ws` option is off: it forwarded every upgrade without authentication.
- Production needs `mod_proxy_wstunnel` and `upgrade=websocket` on the API `ProxyPass` (`docs/MFF-Core_dev-setup.md`, "WebSockets through the web server"). Without it Guacamole's client falls back to its HTTP tunnel, which still works.
- Requires mff-core 0.1.233 or newer. On an older core Remote Desktop does not work (the module gate answers `401`), including the HTTP-tunnel fallback.

## Differences from the old NUSBF app
- The old app was one button, gated only by a hard-coded username in the frontend, with a constant cookie and hard-coded Guacamole credentials.
- The desk module checks the allowed users and the role in the backend, takes credentials from configuration, and uses a per-session cookie and token expiry.
