# Dev setup

Everything below runs with **zero mff-core checkout present on disk** — that's the whole point of the dev harness (`dev/`). It stands in for real core via `coreApi` (see `docs/MFF-Core_module-core-api.md`), so the module's actual code (`backend/`, `frontend/`) is what's under test, never a copy of it.

This dev-harness database (`<id>_dev`, `dev/.env.dev`) is a separate, non-interacting system from the real-install database (`mff_module_<id>`) that mff-core's provisioner sets up on a real Install/Update — don't conflate the two. `dev/.env.dev` is this standalone dev harness's own throwaway file, unaffected by any of this: there's no real core present to ask for config here, so it's not what "no module holds its own `.env`" (`CLAUDE.md`'s "MFF architecture rules") is about. What that rule does affect is the real-install case — the provisioner writes MODULE_<ID>_DB_* credentials into core's own App_backend/.env, not a module-root file - see `docs/MFF-Core_module-core-api.md`'s "Database credentials also go through getModuleConfig() now" section.

## First time

```bash
nvm use          # matches .nvmrc, same version mff-core itself pins
npm install
cp dev/.env.dev.example dev/.env.dev   # edit DB_HOST/DB_USER/DB_PASSWORD if not localhost/root/no-password
npm run dev:db:setup                    # creates <id>_dev and imports backend/db/schema.sql
```

## Running

```bash
npm run dev
```

Starts both halves together (`concurrently`):
- **Backend harness** (`dev/server.js`) — a standalone Express app, your real `backend/index.js` requiring against `dev/mockCoreApi.backend.js`. Listens on `http://localhost:4100` (override via `DEV_PORT` in `.env.dev`). Logged in as a fixed `devuser` persona with `developer` role; override per-request with `curl -H "x-mock-roles: administrator,facility_manager" ...` to test role-gated routes.
- **Frontend harness** (`dev/src/App.dev.js` via `dev/src/index.js`) — a genuinely separate mini CRA app, your real `frontend/index.js` requiring against `dev/src/mockCoreApi.frontend.js`. Renders whatever got registered (first dashboard tab or menu page found) directly, no Dashboard/sidebar chrome. Opens on `http://localhost:3000` as usual for CRA.

`npm run dev:frontend` first copies `../frontend` into `dev/src/module-frontend` (`dev/scripts/sync-frontend.js`) — a plain symlink doesn't work here, since CRA's babel-loader only transforms files inside its own `src/`, and a symlinked-in JSX file resolving outside that directory fails to parse regardless of what `frontend/index.js` actually contains. This is a **one-shot copy, not a live sync**, and it's specific to this throwaway harness — after editing anything under `frontend/`, re-run `npm run dev:frontend` (or `npm run dev`) to see the change reflected here. Real hot-reload on this module's own code happens once it's actually `git clone`d into a real mff-core checkout's `App_frontend/src/core/<id>/` (see `docs/MFF-Core_module-database-provisioning.md`) — a real checkout there, not a copy.

## Honest limitation

Anything that joins against *real* core data — `setup_users`, another module's tables, anything not owned by this module — can only be smoke-tested against hand-seeded fixture rows in your own `<id>_dev` database here. There's no way around that without a real core checkout present, which defeats the point of the harness. When you need that kind of test, `git clone` this module into a real local mff-core checkout's `App_frontend/src/core/<id>/` instead (see `docs/MFF-Core_module-database-provisioning.md`) — that's what integration testing is for, this harness is for everything before that point.

## Vendored/duplicated code, on purpose

`dev/mockCoreApi.backend.js`'s `checkRole` and `dev/src/mockCoreApi.frontend.js`'s `useRoles` are small, intentional copies of core's real logic — not "fixed once at the source," because this code only ever runs in the throwaway harness. The real deployed module always goes through core's actual implementations via `coreApi`, required fresh at install time. If a permission bug only shows up in production and never in the harness, diff these two files against mff-core's `App_backend/src/core/middleware/auth.js` and `App_frontend/src/core/hooks/useRoles.js` — they may have drifted.
