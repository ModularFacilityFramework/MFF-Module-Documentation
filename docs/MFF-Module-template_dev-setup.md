# Dev setup

The template has **no local dev harness** and no `npm run dev`. The `dev/` folder this file used to describe (`dev/server.js`, `dev/mockCoreApi.*`, `dev/.env.dev`, `npm run dev`, `npm run dev:db:setup`) was removed from the template.

## Local steps

```bash
npm install
npm run lint
```

These are the only local steps.

## Running and testing a module

Running or testing a module means `git clone`-ing it into a real mff-core checkout's `App_frontend/src/core/<id>/` — see `docs/MFF-Core_module-database-provisioning.md`.
