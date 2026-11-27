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

Status: tested on 27/11/2026 in an isolated dev core (core v0.1.227) with real Science and a fake SLURM; the tabs, reload and leave/return, the disabled Run forms and the API refusals pass. Not tested there: the "You need a project first" lock, "+ New project", a second ordinary user and losing project access. It is not deployed.

- Needs core v0.1.227 or later. Core's project tabs (`coreApi.ui.ProjectTabs`, `rememberAs` `abcfold.main`, `moduleId` `abcfold`) are at the top of ABCFold.
- Every new or resubmitted ABCFold job runs in a Science project. Write access is required and the project must not be archived. Access is checked again when a queued job is sent. The manifest has `"requires": ["slurm","science"]`.
- Results shows only the selected project's jobs; old jobs are under "Project template". Each project remembers its own open job and sub-tab. Setup is not per project.
- The Run tab runs in the selected project, inside core's `ProjectTask`. It is disabled, with a reason, on the Project template, on read-only or archived projects, and for a user with no writable project. Resubmitting from another tab warns that the job moves. Rerun is off where nothing new can be created (use Resubmit or Clone, then choose a writable tab).
- The server refuses a new or resubmitted job without a project (`400`), in an archived project or the Project template (`409`, also for administrators and developers, who have write access to the template), or without write access (`403`).
- Jobs stay private to their owner. A job in a project its owner can no longer read is not shown. If core cannot decide access, the list answers `503`.
- Each job is a Science project item: `item_type` `"job"`, title `"J<n> <name>"`, a status, and the link `/dashboard?tab=abcfold`. The item is updated on status changes and removed on delete or move. If Science cannot be reached, this is retried about every 15 minutes and never blocks a job.
- Running jobs' statuses are re-read about every 5 minutes with no browser open.
- Jobs created before projects are moved by a timer into Science's "Project template" (`GET /external/template`, `POST /external/template/items`), with the neutral title `"ABCFold job <id>"`. Running one of them again requires choosing a real project.

**Database:** `abcfold_jobs` adds `project_id`, `in_template`, `project_item_state` and `project_item_retry_at`, and there is a new table, `abcfold_item_removals`. The module adds the missing columns itself at startup. The new table comes from Update / Apply Schema.
