# Dev setup

The `abcfold` module has **no local dev harness** and no `npm run dev`. The `dev/` folder this file used to describe (`dev/server.js`, `dev/mockCoreApi.*`, `dev/.env.dev`, `npm run dev`, `npm run dev:db:setup`) was removed from the module.

## Local steps

```bash
nvm use        # matches .nvmrc
npm install
npm run lint
```

These are the only local steps.

## Running and testing the module

Running or testing this module means `git clone`-ing it into a real mff-core checkout's `App_frontend/src/core/abcfold/` — see `docs/MFF-Core_module-database-provisioning.md`. The module holds no `.env` of its own; its database credentials come from core through `coreApi.getModuleConfig()` — see `docs/MFF-Core_module-core-api.md`.
