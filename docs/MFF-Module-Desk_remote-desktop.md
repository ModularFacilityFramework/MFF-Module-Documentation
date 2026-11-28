# Desk module: Remote Desktop

Module `desk` (mount `/desk`, version 0.1.16). It gives a Remote Desktop session through an existing Apache Guacamole, without the user ever seeing Guacamole's login. It is private: see "Who may use it".

## What the user sees
One left-hand menu entry, "Desktop" (`registerMenuPage`, path `/desk`). It is the only way in: there is no Dashboard tab. Core orders menu entries alphabetically by label (`docs/MFF-Core_module-core-api.md`, "Dashboard tab order").

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
1. The button sends `POST /api/desk/session` (`authenticateToken` and `checkRole` `developer`). It refuses "view as" identities.
2. The backend mints an in-memory UUID token (8 h time to live; lost when the backend restarts) and returns `openUrl`.
3. The browser opens `openUrl` in a new tab, at `/api/desk/proxy/?token=...`.
4. The proxy sets an HttpOnly, SameSite=Strict cookie `desk_rd_session` holding that token (its path is the proxy mount).
5. The proxy intercepts Guacamole's `POST /api/tokens` and logs in server-side with a service account, so users never see Guacamole's login.
6. It reverse-proxies everything else to Guacamole (`http-proxy-middleware`).

## Proxy sign-in
- The manifest has `"identifiedRoutes": [{"path": "/proxy"}]`. `/proxy` is signed in through `coreApi.registerIdentifiedRoute` with desk's own token (`?token=` on the first hit) or the `desk_rd_session` cookie (`docs/MFF-Core_module-core-api.md`, "Module's own sign-in").
- Core checks module access on every request.
- The `developer` role is required on every proxy HTTP request and on the WebSocket tunnel.
- The desk token is no longer accepted on `/proxy` as `Authorization: Bearer`.
- Logging out of the app does not end a desk session before it expires.

## Configuration
Declared in the manifest, read through `coreApi.getModuleConfig()`; in `App_backend/.env` they are `MODULE_DESK_*`. These are required:
- `GUAC_HOST`
- `GUAC_PORT`
- `GUAC_CONTEXT_PATH`
- `GUAC_USERNAME`
- `GUAC_PASSWORD`

Optional:
- `ALLOWED_USERS` (`MODULE_DESK_ALLOWED_USERS`): comma-separated MFF usernames, not case-sensitive. If it is not set, the built-in default applies: the owner only (`namlb`). If it is set but empty, nobody can use Desk. A change needs a backend restart.

The module has no database tables.

## Setup outside the module
- A running Guacamole (guacd and the web app), reachable from the backend.
- A service account in Guacamole that has the connection(s) assigned.
- The connection's RDP target (for example xrdp) configured in Guacamole.

## WebSocket tunnel
Desk handles Guacamole's WebSocket tunnel through `coreApi.registerUpgradeHandler('/proxy/websocket-tunnel', handler, { identify })` (`docs/MFF-Core_module-core-api.md`, "WebSocket"). The manifest has `"websocket": true`, and the module depends on `ws`.
- The `identify` callback reads the HttpOnly `desk_rd_session` cookie (the per-session token minted by `POST /session`, 8 h expiry). Guacamole's own `token` query parameter therefore passes through untouched.
- Core re-checks the user's real identity and module access on every connect.
- The tunnel also requires the `developer` role on every connect and reconnect, the same as the HTTP `/proxy` routes: `403`, or `503` if the role lookup fails. An already-open tunnel is not cut when the role is revoked; the check applies at the next connect.
- The backend relays frames to Guacamole's `/websocket-tunnel` with the `guacamole` subprotocol.
- `http-proxy-middleware`'s own `ws` option is off: it forwarded every upgrade without authentication.
- Production needs `mod_proxy_wstunnel` and `upgrade=websocket` on the API `ProxyPass` (`docs/MFF-Core_dev-setup.md`, "WebSockets through the web server"). Without it Guacamole's client falls back to its HTTP tunnel, which still works.
- Requires mff-core 0.1.233 or newer. On an older core Remote Desktop does not work (the module gate answers `401`), including the HTTP-tunnel fallback.

## Differences from the old NUSBF app
- The old app was one button, gated only by a hard-coded username in the frontend, with a constant cookie and hard-coded Guacamole credentials.
- The desk module checks the allowed users and the role in the backend, takes credentials from configuration, and uses a per-session cookie and token expiry.
