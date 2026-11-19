# Module Install / Update / Remove — automatic rebuild and restart

Installing, updating or removing a module changes what must be compiled into the frontend (a module's frontend half is built into the bundle) and what the backend loads (modules are `require()`d once, at startup). The Modules page does both for you — no terminal.

## What happens, in this order

After the git/database/config steps succeed, the backend starts a background job (`App_backend/src/core/services/moduleFinalizeJob.js`):

1. **Build** — `scripts/build.js` (the same entry point as the root `npm run build`) builds the frontend into `App_frontend/build.next/`, **not** `build/`. The live site keeps serving the old build the whole time.
2. **Swap** — only if the build succeeded and `build.next/index.html` exists, `build.next` replaces `build/` (two renames; the old build is deleted last).
3. **Restart** — the backend exits and systemd starts it again, so it loads the new set of modules.

The admin page shows progress (with the build log) and reloads itself when the backend is back.

## Safety rules

- **A failed build changes nothing.** The old site keeps serving, the backend is not restarted, and the page shows the build log. The module's files stay on disk; fix the cause and click Update to retry. (Previously a failed manual `npm run build` wiped `build/` first and took the site down.)
- **One job at a time.** Install/Update/Remove return `409` while a build or restart is in flight.
- **Update skips all of this** when the pull brought nothing new, the module is already running, and no database/config value was just created.
- **Fixed commands only.** Nothing from a request or config value is ever part of a command.
- **Non-production** (`NODE_ENV` not `production`): the production build is skipped (the dev server handles the frontend); only the restart happens.

## Install and Update never leave tracked files modified

- **Dependencies:** Install and Update install a module's dependencies with `npm ci --omit=dev --omit=peer`, so the module's committed `package-lock.json` is never rewritten. If the lockfile does not match `package.json`, or there is none, `npm install --omit=dev --omit=peer --no-package-lock` is used instead and a warning is shown. Module maintainers: commit a correct `package-lock.json` (`npm ci` requires it to match `package.json`).
- **Hiding an installed module from core's git:** core hides an installed module's folder (`App_frontend/src/core/<id>/`) through its own `.git/info/exclude`, **not** by editing the tracked `.gitignore`.
- **Update repairs a dirty checkout first** (left over from earlier installs):
  - It restores `package-lock.json`. The discarded change is saved as `backups/modules/<id>/discarded-package-lock.json-<ts>.patch`.
  - It moves module lines out of core's tracked `.gitignore` into `.git/info/exclude` (this also happens at backend startup).
  - Any other modified tracked file in the module folder stops the Update with a message. Nothing is discarded.
- **Git cleanliness check:** after Install/Update, `git status --porcelain` is checked for the module folder and for core. It warns only about modified tracked files, or untracked files that the incoming commits add (checked after a best-effort `git fetch`); other untracked paths are not warned about. The result is in the response (`gitCheck`), in the job log, and in `GET /modules-management/job` as `job.gitCheck`: `{ clean, module: { entries }, core: { entries }, message }`.
- **At every backend start**, every installed module folder, plus `CLAUDE.md.bak-*` and `.claude/*.bak-*`, are added to core's `.git/info/exclude`.

## The one requirement: `Restart=always`

The backend restarts itself by exiting, so the process must be restarted by something. It only does this when it can verify, at that moment, that it is a **systemd system service whose unit has `Restart=always`** (checked from `/proc/self/cgroup` and `systemctl show <unit> -p Restart`; the unit name is detected, not configured — `SYSTEMD_SERVICE_NAME` is not needed). `services/mff-backend-prod.service.example` already has `Restart=always`, so a unit created from it works.

If it can't verify that (run from a terminal, a different restart policy such as `on-failure`, a user service), it **never exits** — exiting would just kill the backend. The job instead finishes as `needs_manual_restart`: the frontend is already rebuilt and swapped, and the page shows the one command to run (`sudo systemctl restart <SYSTEMD_SERVICE_NAME>`; `SYSTEMD_SERVICE_NAME` defaults to `mff-core-backend.service`). To get full automation on such a deployment, set `Restart=always` in its unit and `sudo systemctl daemon-reload`.

## State and recovery

The job's state is saved to `App_backend/.module-job.json` (gitignored), so the page can show the final result after the backend restarts itself. On startup the backend marks a job that ended by restarting as done, and one that was still building when the process died as failed.

`GET /api/modules-management/job` returns the latest job (developer/administrator only). Opening the Modules page mid-job resumes following it.

## When Update skips the build

Update skips the build and restart only when all of these hold: the pull says "Already up to date", the module is already running, no database or config value was just created, **and** no file in the module's folder (node_modules and .git excluded) is newer than `App_frontend/build/index.html`. So if an earlier Update pulled new code but stopped before building (e.g. at the database step), clicking Update again builds it even though the second pull has nothing new.

## What this doesn't do

- **Reset** (recovering a stuck, never-activated install) does not rebuild or restart — nothing was ever built or loaded for it.
- It does not run for changes made outside the Modules page (e.g. a manual `git pull`): those still need `npm run build` from the repo root and a backend restart.
