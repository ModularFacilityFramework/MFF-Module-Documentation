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

## Job queue

One job per user is with Slurm at a time; the rest are queued. With core v0.1.193 or later the queue advances with no browser open (`"actsAsUsers": true` in the manifest).

## Projects

Status: the code is on `origin/main` (a133313), not yet deployed or tested in a running core.

- Every new or resubmitted ABCFold job runs in a Science project, chosen with core's `ProjectPicker`. Write access is required and the project must not be archived. Access is checked again when a queued job is sent. The manifest has `"requires": ["slurm","science"]`.
- The Results list can be filtered by project. Jobs stay private to their owner. A job in a project its owner can no longer read is not shown. If core cannot decide access, the list answers `503`.
- Each job is a Science project item: `item_type` `"job"`, title `"J<n> <name>"`, a status, and the link `/dashboard?tab=abcfold`. The item is updated on status changes and removed on delete or move. If Science cannot be reached, this is retried about every 15 minutes and never blocks a job.
- Running jobs' statuses are re-read about every 5 minutes with no browser open.
- Jobs created before projects are moved by a timer into Science's "Project template" (`GET /external/template`, `POST /external/template/items`), with the neutral title `"ABCFold job <id>"`. Running one of them again requires choosing a real project.

**Database:** `abcfold_jobs` adds `project_id`, `in_template`, `project_item_state` and `project_item_retry_at`, and there is a new table, `abcfold_item_removals`. The module adds the missing columns itself at startup. The new table comes from Update / Apply Schema.
