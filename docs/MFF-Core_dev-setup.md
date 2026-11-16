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
- Installs git, curl and openssl, nvm and Node 20, MariaDB, and Apache (and enables `proxy`, `proxy_http`, `headers`, `ssl` and `rewrite`).
- Asks plain-English questions: the sub-folder (e.g. `/MFF-cloud`), the public origin, mail, whether you want modules (yes/no), and whether to write the Apache rules into an existing config or a separate site file.
- Generates `JWT_ACCESS_SECRET`, `JWT_REFRESH_SECRET` and `DATA_ENCRYPTION_KEY`. It makes you confirm that you backed up the encryption key.
- Writes `App_backend/.env` with mode `0600`. An existing `.env` is kept as `.env.bak-<time>`, and keys the installer does not manage (such as `MODULE_<ID>_DB_*`) are preserved.
- Runs `setup-database.js`, then `setup-provisioner.js` (only if you want modules), then `create-admin-user.js`.
- Runs `npm ci` for the backend and the frontend, then `npm run build`.
- Writes the systemd unit and the Apache config (it runs the Apache configtest first, then reloads), and ends with health checks.

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
