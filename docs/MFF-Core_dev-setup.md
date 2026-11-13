# Developer Setup Guide

This guide describes how to set up a development environment for the application.

## Prerequisites
- Node.js: use the version in `.nvmrc` (v20). If you use `nvm`, run `nvm use`.
- MariaDB/MySQL server for backend database.
- Git and a POSIX shell (macOS/Linux or WSL).

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
