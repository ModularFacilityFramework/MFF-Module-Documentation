# Developer Setup Guide

This guide describes how to set up a development environment for the application.

## Prerequisites
- Node.js: use the version in `.nvmrc` (v20). If you use `nvm`, run `nvm use`.
- MariaDB/MySQL server for backend database.
- Git and a POSIX shell (macOS/Linux or WSL).

## Install with the installer
On Debian 12/13, as a normal user with `sudo`, run:
```bash
bash scripts/install.sh
```
The installer:
- Installs git, curl and openssl, nvm and Node 20, MariaDB, and Apache (and enables `proxy`, `proxy_http`, `proxy_wstunnel`, `headers`, `ssl` and `rewrite`).
- Asks plain-English questions: the sub-folder (e.g. `/MFF-cloud`), the public origin, mail, and whether to write the Apache rules into an existing config or a separate site file.
- Generates `JWT_ACCESS_SECRET`, `JWT_REFRESH_SECRET` and `DATA_ENCRYPTION_KEY`. It makes you confirm that you backed up the encryption key.
- Writes `App_backend/.env` with mode `0600`. An existing `.env` is kept as `.env.bak-<time>`, and keys the installer does not manage are preserved.
- Runs `npm ci` for the backend and the frontend.
- Runs `setup-database.js`, then `create-admin-user.js`.
- Runs `npm run build`.
- Writes the systemd unit and the Apache config (it runs the Apache configtest first, then reloads), and ends with health checks.

In order, the steps are: tools, nvm/Node 20, MariaDB + Apache, `.env`, `npm ci`, `setup-database.js`, admin user, build, systemd unit, Apache rules, checks.

The installer does not set up modules: it does not ask whether you want modules, the provisioner account, `GITHUB_TOKEN` or `WORKSPACE_BASE_DIR`, and it does not run `setup-provisioner.js`. On a re-run it leaves any `DB_PROVISION_*`, `GITHUB_TOKEN`, `WORKSPACE_BASE_DIR` and `MODULE_<ID>_*` lines in an existing `.env` untouched, and it never writes them on a fresh install. To use modules, follow "After install: modules" below.

It is safe to re-run: the secrets and the database are kept.

Flags: `--dry-run`, `--yes`, `--answers FILE` (a template is in `scripts/installer/answers.example`), `--stage-dir DIR`, `--skip-packages`, `--skip-database`, `--skip-admin`, `--skip-npm`, `--skip-build`, `--no-start`.

The manual steps below remain as the fallback.

## First-time setup
1. Clone the repository and enter the project root.
2. Run the automated bootstrap:
   ```bash
   npm run dev:setup
   ```
   This will:
   - Warn if the `node` on your PATH doesn't match `.nvmrc` (Node 20)
   - Install root tooling (Husky, lint-staged, ESLint, Prettier)
   - Install backend and frontend dependencies
   - Initialize the Husky pre-commit hook (runs lint-staged)
   - Copy `App_backend/.env.example` to `App_backend/.env` if it doesn't already exist

3. Edit `App_backend/.env` with real values (database, JWT secrets, encryption key — see `App_backend/README.md` / `docs/MFF-Core_configuration.md`). There is only one `.env` in the whole app; the frontend has none — it fetches its runtime config from the backend's `GET /api/config` at startup.

4. Set up the database and first admin user:
   ```bash
   cd App_backend/install
   node setup-database.js
   node create-admin-user.js
   ```

## WebSockets through the web server

A module that uses WebSockets (`"websocket": true`, `docs/MFF-Core_module-core-api.md`) needs the web server to pass upgrades through to the backend. The installer (`scripts/install.sh`, `render.js`) and the README already enable and write this for Apache. By hand:

- **Apache:** enable `proxy_wstunnel` (`sudo a2enmod proxy_wstunnel`) and add `upgrade=websocket` to the API `ProxyPass` line, for example `ProxyPass /MFF-cloud-api http://127.0.0.1:5000/MFF-cloud-api upgrade=websocket` (`ProxyPassReverse` is unchanged). Then `sudo apachectl configtest && sudo systemctl restart apache2`.
- **nginx (manual):** in the API `location`, add `proxy_http_version 1.1; proxy_set_header Upgrade $http_upgrade; proxy_set_header Connection "upgrade";`, and a long `proxy_read_timeout`.

## After install: modules
Modules are set up by hand, once the core is installed (with the installer or the manual steps above). The details of what the provisioning account does are in `docs/MFF-Core_module-database-provisioning.md`.

1. Add these to `App_backend/.env`:
   - `DB_PROVISION_USER` — e.g. `mff_provisioner`. It must be distinct per deployment on a shared MariaDB server.
   - `DB_PROVISION_PASSWORD` — generate it with `openssl rand -base64 24`. It must not contain `'`, a backslash or a newline.
   - `GITHUB_TOKEN` — optional (see `docs/MFF-Core_configuration.md`).
   - `WORKSPACE_BASE_DIR` — only needed for cluster modules.
2. Once per deployment, run:
   ```bash
   cd App_backend/install && nvm exec 20 node setup-provisioner.js
   ```
   It asks for `sudo`. Its self-test creates and then drops a scratch module database.
3. From the repo root, restart the backend:
   ```bash
   npm run restart-backend
   ```
4. In the app, as an administrator, open Preferences → Modules and use Install.

## Running locally
- Backend API:
  ```bash
  cd App_backend
  npm start
  ```
- Frontend:
  ```bash
  cd App_frontend
  npm start
  ```

## Code quality
- Pre-commit hook runs `lint-staged` (ESLint + Prettier) on staged files.
- Run manually across the repo:
  ```bash
  npm run lint
  npm run format
  ```
- Or per package:
  ```bash
  npm run lint:backend
  npm run lint:frontend
  ```

## Tests
- Backend tests:
  ```bash
  cd App_backend && npm test
  ```
  `tests/install.test.js` exercises `install/schema.sql` and `install/create-admin-user.js` against a real MariaDB — set `DB_HOST`/`DB_USER`/`DB_PASSWORD`/`DB_NAME` to a database the configured user can create tables and rows in (any existing database works; the test only creates its own uniquely-named rows and cleans them up). Without those set, this suite logs a warning and skips rather than failing — the rest of the suite still runs.
- Frontend tests (CRA):
  ```bash
  cd App_frontend && npm test
  ```

## Useful tips
- Set `CORS_ORIGINS` to your dev origin if needed (e.g., http://localhost:3000).
- Use `.editorconfig` and Prettier extension in your editor for consistent formatting.
