# Environment variable policy — the one `.env` rule

This is the single authoritative statement of this rule. Other docs (`CLAUDE.md`, `docs/MFF-Core_module-core-api.md`, `docs/MFF-Core_module-database-provisioning.md`) link here instead of each restating it — if any of them ever seems to disagree with this page, this page is correct.

## The rule

**There is exactly one `.env` file in the entire system: `App_backend/.env`, owned by core.**

- No module — backend or frontend — has its own `.env`. Not for its own settings, not for its database credentials, not for anything.
- Core's own frontend has no `.env` either (unchanged, always true) — it fetches runtime config from the backend via `/api/config`.
- This is not a preference or a convention modules can opt out of. A module that creates its own `.env` file, or calls `dotenv.config()`/`dotenv.parse()` on any file of its own, is violating this rule and must be fixed.

## What a module MUST do

**Read config via `coreApi.getModuleConfig()`. Never read a file.**

```js
// backend/index.js's register(coreApi), or anywhere else your backend code
// has access to the coreApi object passed to it:
const cfg = coreApi.getModuleConfig();
// cfg = { APP_URL: '...', SOME_TOKEN: '...', DB_HOST: '...', DB_USER: '...', ... }
```

- Takes **no argument** — core already knows which module is calling (bound to your own `module.manifest.json` id before your `register()` ever runs, see `moduleLoader.js`). You cannot pass a different module's id and read its values; there is no argument to do that with.
- Returns **only your own namespace**, prefix already stripped, as a plain object. You never see core's own secrets (JWT keys, `DB_PASSWORD`, etc.) or another module's values.
- Returns `{}` if nothing is set for you — never `null`/`undefined`/an exception. Check for the specific keys you expect.
- **This covers database credentials too**, not just your own custom settings — `DB_HOST`/`DB_USER`/`DB_PASSWORD`/`DB_NAME` come back from the exact same call, no separate mechanism. Core's provisioner writes them into `App_backend/.env` automatically on first Install; you just ask for them the same way you'd ask for anything else.

Full mechanism reference (return shape, binding, edge cases): `docs/MFF-Core_module-core-api.md`, `getModuleConfig()` section.

## What an admin MUST do to set a module's config

Add the value directly to `App_backend/.env` — the one file, nowhere else — namespaced `MODULE_<ID>_<KEY>` (module id upper-cased):

```
MODULE_PANDDA2INSPECT_APP_URL=https://example.org/pandda2-inspect
MODULE_SLURM_SSH_USER=slurm
```

Then restart core's own backend for this deployment. `process.env` is only populated once, at process startup (`dotenv.config()`) — a value added to `App_backend/.env` while the backend is already running isn't visible to `getModuleConfig()` until the next restart. This is the same restart Install/Update already tells the admin to run after any module change — not a separate step, not a new requirement.

## Some values are provisioned automatically — a module MUST declare which

A module declares its own config keys in `module.manifest.json`'s `config` array (see `docs/MFF-Core_module-manifest.md`). Install and Update read it and act on each entry:

- **`{ key, generate: "secret" }`** — core generates the value itself (a random token, nothing a human needs to know) and writes it, zero manual step, ever.
- **`{ key, required: true, description }`** — core cannot generate this (a hostname, a token issued by something external) — Install/Update's response names exactly which required keys are still unset, instead of the module silently not working with no indication why.

This does not change anything above — it's still one file, still `MODULE_<ID>_<KEY>`, still read via `getModuleConfig()`. It only removes the need for a human to run CLI commands or hand-edit `App_backend/.env` for the values core is *able* to supply itself.

## Why this exists (brief)

Before this, a module could hold its own `.env` (`App_frontend/src/core/<id>/.env`). In practice this produced two real, repeated production failures:
1. A value hand-edited into the *wrong* `.env` (core's instead of the module's, or vice versa) failed silently — the module just read as "not configured," with nothing pointing at why.
2. On a host running more than one mff-core deployment, there was no reliable link between "which `.env`" and "which systemd unit to restart" — restarting the wrong deployment's backend left changes looking like they'd had no effect.

One file, one restart target, structurally impossible to put a value in the wrong place — see `docs/MFF-Core_module-database-provisioning.md`'s "Database" section and `CLAUDE.md`'s "MFF architecture rules" for how this ties into the rest of the module-isolation design.


## What this explicitly does not mean

- It does not mean config changes take effect without a restart. It removes the "wrong file" class of error, not the "needs a restart" one.
- It does not add a database-backed, save-and-it's-live admin UI for module config. Values are still plain environment variables in a text file, edited by an admin, picked up on restart — same mechanism as every other `App_backend/.env` value this app has ever had.
- It does not change how a module's *own* database is provisioned (still automatic, still per-module, still isolated) — only where the resulting credentials are stored and how a module reads them back.
