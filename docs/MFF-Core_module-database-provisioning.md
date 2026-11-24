# Module Database Provisioning

Every installed module gets its own dedicated MariaDB database and MariaDB user, provisioned automatically by the Install/Update buttons on Setup → Preferences → Modules. A module's data is never stored in core's own database, and never shared with any other module's database — one module's schema, bug, or bad update can't touch core's data or another module's data.

This is separate from core's own database setup (`App_backend/install/setup-database.js`, see `docs/MFF-Core_configuration.md`). That creates core's one database, once, interactively, via `sudo`. This document covers the account that lets the *running backend* provision a *new* database per module, non-interactively, from a button click.

The resulting credentials are stored and read back the same way every other module config value is — see `docs/MFF-Core_env-policy.md` for the full policy this follows.

## What a module needs to provide (module-developer contract)

Everything below "One-time setup" is ops/admin setup, done once per deployment by whoever runs the server — none of it is something a module's own code does or needs to know about. From a module's own point of view, the entire database contract is one file:

**`backend/db/schema.sql`** — plain `CREATE TABLE IF NOT EXISTS` statements for your module's own tables, conventionally prefixed with your module `id` (matching `tablePrefix` in `module.manifest.json` — see `docs/MFF-Core_module-manifest.md`; not enforced by code, just convention). Never `DROP`, never `ALTER`/`MODIFY` an existing column — see "Schema diffing" below for why that's a hard rule, not a style preference.

That's it. You do not write an install script, a migration runner, or any database-connection code of your own for setup:

- **First Install**: core provisions your module's dedicated database and user automatically, then imports `backend/db/schema.sql` as-is. Your module's `backend/config/database.js` gets those credentials from core by calling `coreApi.getModuleConfig()` — **a module never holds its own `.env`, database credentials included; there is exactly one `.env` in the whole system, `App_backend/.env`, owned by core** (see "MFF architecture rules" in the root `CLAUDE.md`, and `docs/MFF-Core_module-core-api.md` for `getModuleConfig()` itself). You never see or handle the provisioning account, grants, or any of the ops-level detail further down this doc. Credentials come back as `DB_HOST`/`DB_USER`/`DB_PASSWORD`/`DB_NAME` (the `MODULE_<YOUR_ID>_DB_` prefix already stripped by `getModuleConfig()`, same as any other config key).
- **Every later Update, or a manual "Apply Schema" click** (Setup → Preferences → 🗄️ Database, the Database Management admin page): `backend/db/schema.sql` is re-applied automatically. A genuinely new `CREATE TABLE IF NOT EXISTS` picks up on its own. For a *new column on a table that already exists*, core diffs your live database against a throwaway reference import of your current `schema.sql` and runs the `ALTER TABLE ... ADD COLUMN` for you (see "Schema diffing" below for exactly what it will and won't do automatically).

So: add a table, or add a column to an existing table, by editing `backend/db/schema.sql` and pushing — the next Update (or an admin clicking Apply Schema) picks it up with zero other code. Nothing about accounts, grants, hosts, or `DB_PROVISION_USER` below is a module's concern; it's included here for whoever operates the deployment.

## One-time setup (ops/admin only — not a module concern)

Once per deployment, never per module. Install/Update/Reset then need no database admin work at all.

1. Put the provisioning account's credentials in `App_backend/.env` (generate the password with `openssl rand -base64 24`; it must not contain `'` or `\`):
   ```
   DB_PROVISION_USER=mff_provisioner
   DB_PROVISION_PASSWORD=<the password>
   ```
   If this MariaDB server already hosts another MFF-core deployment, use a deployment-distinct username (e.g. `mff_provisioner_<deployment>`), not `mff_provisioner` again — one username can only ever have one password per host, so two deployments sharing it collide.
2. Run, on the server:
   ```bash
   cd App_backend/install && node setup-provisioner.js
   ```
   It asks for `sudo` (same as `setup-database.js`). It is idempotent, and it never changes the password of an account that already exists.

This is a manual step after the core is installed: the installer (`scripts/install.sh`) does not run `setup-provisioner.js` or write the `DB_PROVISION_*` values. See `docs/MFF-Core_dev-setup.md`, "After install: modules". The provisioner user name must be distinct per deployment on a shared MariaDB server (see step 1).

What the script sets up:
- The provisioning account, for both `127.0.0.1` and `localhost` (MariaDB grant-matches against the *resolved* peer hostname, so a TCP connection to `127.0.0.1` is commonly matched as `localhost`).
- A database `mff_provisioning` holding one stored procedure, `grant_module_db(db, host)`, defined by root with `SQL SECURITY DEFINER`. It performs the single `GRANT ALL PRIVILEGES ON mff_module_<id>.* TO mff_module_<id>@host` and refuses any name that isn't `mff_module_<id>`.
- For the account: `EXECUTE` on that procedure, `ALL PRIVILEGES` on ``mff\_module\_%``.* (create/drop module databases), and the global `CREATE USER`.
- A self-test: as the provisioner, it creates a scratch module database and user, grants through the procedure, checks the granted user can create a table, then drops everything. If it fails, the script exits non-zero and says why — nothing is left half-working.

**Why a procedure and not a wildcard `GRANT OPTION`:** confirmed live on MariaDB 11.8.6, a wildcard grant (``mff\_module\_%``.* WITH GRANT OPTION) is not enough for the account to `GRANT` on a specific `mff_module_<id>` database (`Access denied ... to database`); adding `GRANT GRANT OPTION ON *.*` doesn't help either. Granting through a root-defined procedure needs no broad privilege and no per-module step.

**What the account can and can't do:** it can create and drop databases and users, grant only the one module-database grant above, and run nothing else as root. The escaped underscores in ``mff\_module\_%`` matter — unescaped, `_` is a single-character wildcard. It cannot read, write, or drop core's own database or any other database on the server. The one global privilege is `CREATE USER ON *.*` — MariaDB can't scope it by username pattern, so this account can create/alter/drop *any* MariaDB user. That's a real, unavoidable trade-off; give its password the same handling as any production secret.

Leave `DB_PROVISION_USER`/`DB_PROVISION_PASSWORD` unset if you don't plan to install modules — Install/Update then returns a clear error instead of a cryptic MariaDB failure. If they're set but the script was never run, Install stops before creating anything and says to run it.

Confirm the server is MariaDB ≥ 10.4.3 (`SELECT VERSION();`) — see "Naming and length limits" below.

A failed Install cleans up after itself: if anything fails after the module's database or user was created, they're dropped again, so a retry via Update starts clean.

## What Install and Update actually do

`App_frontend/src/core/<id>/` is the module's **one and only checkout** — Install's `git clone` writes it there once, and every later Update is a plain `git pull` in that same directory, never a re-clone. There is no separate build/deploy copy anywhere else: this is the exact tree CRA compiles the frontend half from and the exact tree the backend `require()`s the backend half from directly (see `moduleLoader.js`'s own file-header comment for why there's deliberately no second copy).

Both routes (`App_backend/src/core/setup/routes/modulesManagementRoutes.js`) call the same shared helper, `ensureModuleDatabase()` (`App_backend/src/core/services/moduleDatabaseProvisioner.js`), after their existing git step (`git clone` for Install, `git pull` for Update):

1. **Does `App_backend/.env` already have working `MODULE_<ID>_DB_*` credentials** for this module (`readModuleDbCredentials()`)?
   - **Yes** → back it up (`mysqldump`, gzipped, with a `.sha256` checksum, same pattern as `scripts/db_backup.sh`) to `backups/modules/<id>/<id>_<timestamp>.sql.gz`, then re-apply `backend/db/schema.sql` using the module's own existing credentials via `schemaSyncService.applySchema()` — `CREATE TABLE IF NOT EXISTS` for any genuinely new table, plus a diff-and-`ALTER TABLE ADD COLUMN` pass for any new column on an already-existing table (see "Schema diffing" below). Both are non-destructive by construction — neither can ever alter an existing column or drop anything. The backup is a safety net if something ever goes wrong, not something this flow restores from automatically — nothing in the normal path destroys anything.
   - **No** → connect with the `DB_PROVISION_USER` admin account and check whether a database named `mff_module_<id>` already exists.
     - **Doesn't exist** → fresh provision: `CREATE DATABASE`, `CREATE USER ... IDENTIFIED BY '<random hex password>'`, `GRANT ALL PRIVILEGES` scoped to that one database, import `backend/db/schema.sql` as the new user, append `MODULE_<ID>_DB_HOST`/`_USER`/`_PASSWORD`/`_NAME` to `App_backend/.env` (never touching any other line in it) with the new credentials.
     - **Already exists** (orphaned — see below) → refuse, with an error explaining why.

This is why Install can safely be retried via Update after a database-provisioning failure (see the error message it returns) — there's no re-clone needed, and the check-first logic is identical either way.

## What Reset does

`POST /modules-management/:id/reset` (same routes file, `resetModuleDatabase()` in `moduleDatabaseProvisioner.js`) recovers a module that's stuck partway through Install — cloned, but the dependency install (`npm ci`, or the no-lockfile `npm install --no-package-lock` fallback) or the database step then failed, leaving a folder (and sometimes a half-provisioned database) behind with nothing registered for it. It's only offered in the UI for a module in exactly that state (`alreadyCloned` but not in `/api/modules`) — refused outright (409) for a currently active, registered module, since that's what Remove (below) is for instead.

1. **Does `App_backend/.env` have working `MODULE_<ID>_DB_*` credentials for this module?** Yes → back it up the same way Update does, then continue. No (orphaned, or never got that far) → nothing to back up, continue anyway.
2. `DROP DATABASE IF EXISTS` / `DROP USER IF EXISTS` via the `DB_PROVISION_USER` admin account — safe no-ops if either never existed.
3. Strip this module's four `MODULE_<ID>_DB_*` lines back out of `App_backend/.env` (and out of the running process's own `process.env`), so a later Install doesn't find stale credentials pointing at a database that no longer exists.
4. Delete the module's cloned folder entirely.

The module id is now fully back to "never installed" — a plain Install works cleanly afterward, no special-cased recovery path needed. If a backup was made, it's offered as a download from the same response; nothing is deleted from `backups/modules/<id>/` automatically (same as Update's own backups — see "Backup retention" below).

## What Remove does

`POST /modules-management/:id/remove` is the mirror image of Reset: it only ever acts on a module that IS currently active/registered (404 otherwise), and is the actual uninstall flow. Same three steps as Reset — back up the database if reachable, `DROP DATABASE`/`DROP USER IF EXISTS`, delete the cloned folder — reusing `resetModuleDatabase()` directly. Same backup-download response shape as Reset. Then rebuilds the frontend and restarts the backend automatically (see `docs/MFF-Core_module-install-finalize.md`) so the module actually disappears from the running app. Once removed, the module id reappears under "Available to install", same as one that was never installed at all.

## Naming and length limits

Database and username are both derived as `mff_module_<id>`. Module ids are already required to be short/lowercase (`^[a-z][a-z0-9_]*$`), but MariaDB usernames were capped at 32 characters before version 10.4.3 (and 16 before that, in older MySQL). The provisioner rejects any module id that would push the derived name past 32 characters total, rather than let MariaDB fail unpredictably later — if you hit this, the module needs a shorter `id` in its `module.manifest.json`.

## Backup retention

`backups/modules/<id>/` is **not** pruned by anything — unlike `scripts/db_backup.sh`'s own scheduled backups (which respect `RETENTION_DAYS`), these are one-shot snapshots taken immediately before an Update, and they accumulate indefinitely. Clean them up manually if disk space matters, or extend `scripts/db_backup.sh`'s retention logic to cover this directory if it becomes worth automating.

## Manual recovery: orphaned module database

This happens if a module's database was provisioned successfully, but its `MODULE_<ID>_DB_*` lines in `App_backend/.env` were later lost — most likely from hand-editing or restoring `.env`, or from manually deleting `App_frontend/src/core/<id>/` without first dropping its database.

The provisioning account deliberately has no `SELECT` privilege anywhere (see above) — it cannot read the orphaned database's data, so it cannot back it up before resetting its user's password. Rather than silently generate a new password that won't match the surviving database (leaving the module unable to connect, with no error until first use), `ensureModuleDatabase()` detects this state and fails loudly instead.

To recover manually:
1. Connect with full admin access (not the provisioning account) and inspect `mff_module_<id>` to confirm what's actually there.
2. Back it up yourself: `mysqldump -h <host> -u root -p mff_module_<id> | gzip > recovery.sql.gz`.
3. Decide: reset the existing user's password (`ALTER USER 'mff_module_<id>'@'<host>' IDENTIFIED BY '<new password>'`), **or** drop the database/user entirely and let Install/Update provision it fresh (losing the module's data — only do this if the backup in step 2 is not actually needed, or has already been restored elsewhere).
4. Put the credentials back in `App_backend/.env`, the one `.env` in the system: re-add the four lines `MODULE_<ID>_DB_HOST`, `MODULE_<ID>_DB_USER`, `MODULE_<ID>_DB_PASSWORD` and `MODULE_<ID>_DB_NAME` (module id upper-cased), with the new password from step 3, then restart core's backend so it re-reads `.env`. These are the only place core writes and reads a module's database credentials (`moduleDatabaseProvisioner.js`); there is no module-local `.env` and no fallback to one, so never create `App_frontend/src/core/<id>/.env`. A module reads the values through `coreApi.getModuleConfig()` (keys `DB_HOST`, `DB_USER`, `DB_PASSWORD`, `DB_NAME`; no `DB_PORT` or connection limit is written).

## Schema diffing — new columns on already-existing tables

Re-applying `schema.sql` (`CREATE TABLE IF NOT EXISTS`) only ever picks up brand-new tables — it silently no-ops on a table that already exists, even if that table's own definition in `schema.sql` has gained a new column since the database was first provisioned. That gap used to mean an admin hand-running `ALTER TABLE` over SSH every time. `schemaSyncService.applySchema()` (`App_backend/src/core/services/schemaSyncService.js`) closes it, for core's own database and every module's database alike:

1. Re-applies `schema.sql` against the live target directly (existing, safe — handles new tables, unchanged).
2. Creates a throwaway *reference* database, named `mff_module__schemadiff_<prefix>_<random>` (deliberately prefixed to match the `mff_module_%` wildcard `DB_PROVISION_USER` is already granted on — **no new SQL grant is required for this**), and imports `schema.sql` into it fresh.
3. Diffs `information_schema.COLUMNS` between the live target and the reference database, per table that exists in both — MariaDB itself resolves every column's real, fully-qualified type/default, this codebase never regex-parses `schema.sql`'s SQL text.
4. Applies `ALTER TABLE ... ADD COLUMN ...` for anything the reference has and the live target doesn't. A column is deliberately **skipped**, not attempted, when it's `auto_increment`, a primary key, or `NOT NULL` with no `DEFAULT` — any of those could fail or behave surprisingly on a table that already has rows, and the admin sees exactly why in the response instead of a raw MariaDB error.
5. Always drops the throwaway reference database afterward, even on failure.

**Corollary of reusing `DB_PROVISION_USER`'s existing wildcard for the reference database**: applying schema changes to *core's own* database now also requires `DB_PROVISION_USER`/`DB_PROVISION_PASSWORD` to be configured — previously only module installs needed that pair; a module-free deployment could skip it entirely. If it's unset, Apply Schema (core or module) fails with a clear error naming exactly what's missing, never a crash or a silent no-op.

This is exposed as the **Database Management** admin page (Setup → Preferences → 🗄️ Database, developer/administrator only) — "Apply Schema" and "Backup" buttons for core and every installed module, backed by `App_backend/src/core/setup/routes/databaseManagementRoutes.js`. This is now the recommended way to pick up a schema change — no more manual SSH + `mysql`. The existing Update button also gained this for free (it already calls `ensureModuleDatabase()`, now upgraded internally to call `applySchema()` instead of a bare `CREATE TABLE IF NOT EXISTS` import) — a module's schema change is applied automatically on its next Update, same as a new table always was.

## Not built (deliberately out of scope)

- **No real migration system** for anything schema diffing above can't safely automate — a column's type changing, a column being renamed, or a column being dropped all still need a manual, deliberate `ALTER TABLE` by an admin. Schema diffing only ever *adds* (tables, columns); it never alters or removes anything that already exists.
- **Update blocks the whole app while it runs.** `mysqldump`/schema-apply run synchronously in the request handler (same trade-off `git clone`/`git pull` already made) — for a module with a lot of data, this can take long enough to hit a reverse proxy's default timeout even though the backend keeps working to completion server-side. Prefer doing large-module updates during low-traffic windows.
