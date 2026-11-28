# Why there's no install script in this repo

How installing/updating a module actually works (the one-and-only-checkout model, database provisioning, etc.) is core's own mechanism — see `docs/MFF-Core_module-database-provisioning.md`. This file only covers what is specific to *this* repo.

There used to be an install script here (`scripts/install-into-core.js`) that copied/symlinked this repo's files into an mff-core checkout. That put the module's code in two places on disk and required this repo to know mff-core's internal directory layout — both wrong. A module's code should exist in exactly one place: wherever it's actually `git clone`d. Where that is, and how it gets there, is mff-core's decision to make (see mff-core's own `modulesManagementRoutes.js`) — not something this repo orchestrates.

## Database configuration
A module has no `.env` of its own and never reads one. `backend/index.js` builds the module's pool inside `register(coreApi)` with `createPool(coreApi.getModuleConfig())` from `backend/config/database.js`, using `DB_HOST`, `DB_USER`, `DB_PASSWORD`, `DB_NAME` (`DB_PORT` and `DB_CONNECTION_LIMIT` are optional, default 3306 and 5). Nothing is created when the file is required. If a required key is missing, one log line names it and database routes answer `500` with that reason; there is no fallback to core's pool. A changed value needs a backend restart. The backend has no `dotenv` dependency.
