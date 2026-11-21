# Desk module: Remote Desktop

Module `desk` (mount `/desk`, version 0.1.10). It gives developers a Remote Desktop session through an existing Apache Guacamole, without the user ever seeing Guacamole's login.

## What the user sees
One Dashboard tab and a sidebar entry, both named "Remote Desktop". The tab registers position 20 and the admin override `MODULE_DESK_DASHBOARD_POSITION`. Since mff-core 0.1.192 neither affects the order: tabs and sidebar entries are listed alphabetically by label (`docs/MFF-Core_module-core-api.md`, "Dashboard tab order").

## How a session starts
1. The button sends `POST /api/desk/session` (`authenticateToken` and `checkRole` `developer`).
2. The backend mints an in-memory UUID token (24 h time to live; lost when the backend restarts) and returns `openUrl`.
3. The browser opens `openUrl` in a new tab, at `/api/desk/proxy/?token=...`.
4. The proxy sets an HttpOnly, SameSite=Strict cookie `desk_rd_session` holding that token (its path is the proxy mount).
5. The proxy intercepts Guacamole's `POST /api/tokens` and logs in server-side with a service account, so users never see Guacamole's login.
6. It reverse-proxies everything else to Guacamole (`http-proxy-middleware`).

## Configuration
Declared in the manifest, read through `coreApi.getModuleConfig()`; in `App_backend/.env` they are `MODULE_DESK_*`. All are required:
- `GUAC_HOST`
- `GUAC_PORT`
- `GUAC_CONTEXT_PATH`
- `GUAC_USERNAME`
- `GUAC_PASSWORD`

The module has no database tables.

## Setup outside the module
- A running Guacamole (guacd and the web app), reachable from the backend.
- A service account in Guacamole that has the connection(s) assigned.
- The connection's RDP target (for example xrdp) configured in Guacamole.

## Known gap
WebSocket upgrade is not proxied: `coreApi` gives modules only an Express Router. Guacamole therefore falls back to its HTTP tunnel.

## Differences from the old NUSBF app
- The old app was one button, gated only by a hard-coded username in the frontend, with a constant cookie and hard-coded Guacamole credentials.
- The desk module has a backend role check, credentials from configuration, a per-session cookie, and token expiry.
