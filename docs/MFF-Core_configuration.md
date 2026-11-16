# Configuration Management

This document lists all configuration options for the application, where they are used, defaults, and guidance for secrets management.

## Overview
- Backend reads configuration from environment variables via `App_backend/.env` and `process.env`. This is the only `.env` in the whole system — see below.
- Frontend build-time variables (`PUBLIC_URL`, `REACT_APP_BASENAME`, `REACT_APP_API_BASE_PATH`) are injected as process env vars by `scripts/build.js` (which itself reads `PUBLIC_URL_PATH`/`API_BASE_PATH` from `App_backend/.env`) into the `npm run build` subprocess — there is no `App_frontend/.env` file; the frontend has none, ever.
- Secrets (JWT keys, DB credentials, encryption keys, SMTP creds) must never be committed to source control.

## Backend environment variables (App_backend/.env)

There is only **one** `.env` file in the whole application, in `App_backend/`. The frontend has no `.env` of its own — it fetches its runtime configuration from the backend's `GET /api/config` endpoint at startup instead. See `App_backend/.env.example` for the authoritative, up-to-date list; the variables below are grouped by purpose.

Core
- `BACKEND_PORT` — backend server port
- `API_BASE_PATH` (default `/api`) — base path prefix for all endpoints. Normalized to leading slash, no trailing slash.
- `NODE_ENV` — `development` (default) or `production`
- `APP_NAME`, `APP_BANNER_TITLE` — branding, served to the frontend via `/api/config`
- `FACILITY_NAME` — shown in ticket notification emails; defaults to "Facility Management" if unset

Database
- `DB_HOST` — database host (e.g., 127.0.0.1)
- `DB_USER` — database username
- `DB_PASSWORD` — database password (secret)
- `DB_NAME` — database name
- `DB_CONNECTION_LIMIT` — optional pool size (default 5)

Uploads
- `UPLOADS_BASE_PATH` — external URL base for files
- `UPLOAD_DIR_NAME` (default `uploads`) — private, authenticated, encrypted-at-rest uploads directory name under `App_backend/`
- `PUBLIC_UPLOAD_DIR_NAME` (default `public-uploads`) — public, unauthenticated directory for the three branding assets (favicon, login logo, faculty logo), served at `<API_BASE_PATH>/public-files` (nested under this deployment's own base path so it can't collide with another MFF-core deployment sharing the same Apache vhost)

Workspaces
- `WORKSPACE_BASE_DIR` — base directory for per-user workspaces. If unset, `getUserWorkspacePath()` reports `not_configured` (see `docs/MFF-Core_module-core-api.md`).

Frontend links and CORS
- `PUBLIC_URL_PATH` — the sub-folder the app is served under (e.g. `/MFF-cloud`). **Read at build time** by `scripts/build.js`, and it must match the Apache `Alias`. Changing it needs a frontend rebuild.
- `FRONTEND_URL` — absolute URL to the frontend, **including the deployment subpath** (e.g. `https://host/MFF-cloud`). Every link in an email is built from it: password reset, "review pending registrations" (admins), "log in" (registration received / account activated), and the Support link in ticket emails. If unset, emails are still sent but without a link.
- `CORS_ORIGINS` — comma-separated list of allowed origins (leave empty to allow all in dev)

Email
- `SMTP_HOST`, `SMTP_PORT`, `SMTP_SECURE`, `SMTP_AUTH`, `SMTP_USER`, `SMTP_PASS`, `SMTP_REJECT_UNAUTHORIZED` — SMTP settings for outbound mail. `SMTP_REJECT_UNAUTHORIZED`: unset counts as `true`, so the suggested production value is `true`.
- `FROM_EMAIL` — default "From" address for SMTP mail
- `SYSTEM_EMAIL`, `MAIL_ADMIN_EMAIL` — addresses used for local-mail-fallback and admin notifications

Auth & Security
- `JWT_ACCESS_SECRET` (secret) — required in every environment; the app refuses to start without it
- `JWT_REFRESH_SECRET` (secret) — required in every environment; the app refuses to start without it
- `JWT_ACCESS_EXPIRES_IN` (default `15m`)
- `JWT_REFRESH_EXPIRES_IN` (default `7d`)
- `DATA_ENCRYPTION_KEY` (secret) — must be exactly 32 bytes, base64 or hex encoded (`openssl rand -base64 32`); used to encrypt MFA secrets and file contents at rest. The app throws at startup if this isn't a conforming 32-byte key — there is no silent hash-fallback.
- `SUPERUSER_ROLE` — role name that bypasses all `checkRole()` permission checks (break-glass superuser). Leave unset/empty to disable this bypass entirely.
- `DEPLOYMENT_MODE` (default `standard`) — `standard` or `predetermined`, decided once at install time, enforced against the database afterward (changing it on an existing deployment makes the backend refuse to start). **Permanent once the database exists.** See `docs/MFF-Core_predetermined-mode.md`.
- `CORS_ORIGINS` — see above
- `ENABLE_METRICS` (`true`/`false`) — expose `/api/metrics`
- `LOG_LEVEL` — defaults to `debug` in development, `info` in production

Modules
- `DB_PROVISION_USER`, `DB_PROVISION_PASSWORD` (secret) — the provisioning account that creates each module's database and user. Leave unset if you don't plan to install modules. See `docs/MFF-Core_module-database-provisioning.md`.
- `GITHUB_TOKEN` (secret) — GitHub access token; see `App_backend/.env.example`.
- `SYSTEMD_SERVICE_NAME` — name of the systemd unit that runs the backend. Auto-restart after a module Install/Update does not need it (the unit name is detected); see `docs/MFF-Core_module-install-finalize.md`.
- `MODULE_<ID>_*` — per-module values (module id upper-cased), added to this same `.env`, either by an admin or written by core itself (for example `MODULE_<ID>_DB_HOST`/`_USER`/`_PASSWORD`/`_NAME`). A module reads its own namespace through `coreApi.getModuleConfig()`. See `docs/MFF-Core_env-policy.md`.

## Secrets Management
- Do not commit `.env`; it's gitignored, and `.env.example` is the checked-in template.
- For production, prefer a secret manager (HashiCorp Vault, AWS/GCP/Azure secrets, etc.) to populate `.env` rather than editing it by hand on the server.
- Rotate JWT secrets periodically; revoking refresh tokens will force re-authentication.
- Avoid printing secrets in logs.

## Configuration validation and normalization
- Backend normalizes `API_BASE_PATH` to always start with `/` and have no trailing `/`.
- `JWT_ACCESS_SECRET`, `JWT_REFRESH_SECRET`, and `DATA_ENCRYPTION_KEY` are validated at startup in every environment (not just production) — the app throws immediately rather than falling back to an insecure default.

## Example quickstart
1) Copy the example (backend only — there is no frontend `.env`):
```
cp App_backend/.env.example App_backend/.env
```
2) Edit values as needed.
3) Start services:
```
(cd App_backend && npm start)
(cd App_frontend && npm start)
```
