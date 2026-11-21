# SLURM module configuration

Per `CLAUDE.md`'s "MFF architecture rules" and `docs/MFF-Core_env-policy.md`, the target is exactly one `.env` in the whole system (`App_backend/.env`, core's own) — no module, this one included, holds its own `.env` file, for anything, ever. There is exactly one config mechanism, and it covers this module's own database credentials and every setting below identically, with no distinction between them: `backend/index.js`'s `register(coreApi)` calls `coreApi.getModuleConfig()` **exactly once** and gets back one object containing `DB_HOST`/`DB_USER`/`DB_PASSWORD`/`DB_NAME` (provisioned automatically - every module gets its own database on Install, nothing to declare for that part) alongside every variable below (each declared in this module's `module.manifest.json`'s `config` array - see `docs/MFF-Core_module-manifest.md`'s `"config"` field - so core auto-provisions them into `App_backend/.env` on Install/Update too, no CLI instructions, no hand-edited file anywhere). That one object is passed to both `backend/config/database.js`'s `createPool()` and `backend/config/slurmConfig.js`'s `init()` - `getModuleConfig()` is never called a second time.

None of the variables below are declared `"required": true` in the manifest - this module's monitoring/detection routes (`GET /nodes`, `/partitions`, `/stats`, `/stats/history`) work with every one of them left at its default. They only matter once a facility wants the optional restart/recovery/external-API features, so an admin sets them via `App_backend/.env` (or whatever settings UI core builds atop `getModuleConfig()`) only if and when they want those features.

| Variable | Default | Purpose |
|---|---|---|
| `SLURM_SSH_USER` | `slurm` | SSH user this module connects as to restart daemons |
| `SLURM_SSH_KEY_PATH` | *(none - required for restart)* | Path to the private key used for SSH restarts |
| `SLURM_CONTROLLER_HOST` | *(none - required for master restart)* | Hostname of the node running `slurmctld` |
| `SLURM_POLL_INTERVAL_MS` | `300000` (5 min) | How often nodes/partitions/queue are snapshotted into this module's history tables |
| `SLURM_RESTART_NODE_CMD` | `sudo systemctl restart slurmd` | Command run over SSH on a compute node |
| `SLURM_RESTART_MASTER_CMD` | `sudo systemctl restart slurmctld` | Command run over SSH on the controller |
| `SLURM_BIN_SQUEUE` | `squeue` | Override if not on PATH |
| `SLURM_BIN_SCONTROL` | `scontrol` | Override if not on PATH |
| `SLURM_CTLD_LOG_PATH` | `/var/log/slurm/slurmctld.log` | Path on the controller the Logs tab's download button tails |
| `SLURM_LOG_TAIL_LINES` | `10000` | How many lines of that log get downloaded - bounded so this can never try to ship a multi-GB file |
| `SLURM_ADMIN_USER` | `root` | OS user the node-recovery resume command runs as, via local `sudo -u` - see "Node recovery" below. Set to a dedicated non-root Slurm operator account instead if preferred, or `""` (empty) to run as the backend's own OS user with no sudo hop at all |
| `SLURM_BIN_SACCTMGR` | `sacctmgr` | Override if not on PATH - used by the external API's Slurm-access gate (see below) |
| `SLURM_BIN_SACCT` | `sacct` | Override if not on PATH - used by the external API's job-status route |
| `SLURM_BIN_SBATCH` | `sbatch` | Override if not on PATH - used by the external API's job-submission route |
| `SLURM_BIN_SCANCEL` | `scancel` | Override if not on PATH - used by the external API's job-cancel route |
| `SLURM_RESTART_MUNGE_CMD` | `sudo -n systemctl restart munge` | Command run over SSH on a node by Recover to restart munge |
| `SLURM_ENABLE_NODE_CMD` | `sudo -n systemctl enable slurmd` | Command run over SSH on a node by Recover to enable slurmd at boot |
| `SLURM_SLURMD_LOG_PATH` | `/var/log/slurm/slurmd.log` | Path on a node whose tail Recover reads when the node is still unhealthy |
| `SLURM_SLURMD_PORT` | `6818` | slurmd port Recover checks with a TCP connection from the backend host |
| `SLURM_RECOVER_TIMEOUT_MS` | `90000` | How long Recover waits for the node to stop being DOWN/DRAIN/UNK/NOT_RESPONDING |
| `SLURM_ALLOC_FINANCE_PATH` | `/gpu-allocations` | Path, relative to the finance module's mount, of the GPU allocations lookup (Allocations tab) |
| `SLURM_BIN_GETENT` | `getent` | Override if not on PATH - used to refuse a new Slurm user that has no Unix account |
| `SLURM_CONF_DIR` | `/etc/slurm` | Directory holding the editable config files on the controller and every node (Config tab) |
| `SLURM_CONF_FILES` | `slurm.conf,gres.conf` | File names the Config tab may edit. slurmdbd.conf is never editable from the app |
| `SLURM_CONF_READ_CMD` | `cat {path}` | Remote command (over SSH) that prints one file; `{path}` = SLURM_CONF_DIR + an allowlisted file name |
| `SLURM_CONF_WRITE_CMD` | `sudo -n /usr/bin/install -m 0644 /dev/stdin {path}` | Remote command (over SSH) that writes stdin to one file |
| `SLURM_HEALTH_TIMEOUT_MS` | `180000` | How long to wait for slurmctld and every previously-healthy node to look healthy after a config change before rolling it back |

## Dependencies

Module version 0.1.37 (Allocations changes in 0.1.38 and 0.1.39; `gpu-capacity` and people display in 0.1.40; `gpu-capacity` made a public route in 0.1.41; page state in 0.1.42). The module `requires` `finance` and sets `actsAsUsers: true` in its manifest (`docs/MFF-Core_module-manifest.md`). Finance must therefore be installed before slurm can be installed or updated, and finance cannot be removed while slurm is installed. `actsAsUsers` is used by the Allocations tab's daily run, which calls finance with no request in flight (`coreApi.callModuleAsUser`, see `docs/MFF-Core_module-core-api.md`).

## What this module does NOT do

- **Provision SSH access.** The SSH key at `SLURM_SSH_KEY_PATH` and the
  corresponding `authorized_keys` entry on every compute node + the
  controller must already exist before restart endpoints will work.
  Detection/stats (`GET /nodes`, `/partitions`, `/stats`, `/stats/history`)
  work without any of this.
- **Provision sudo rights.** The SSH user must be able to run
  `SLURM_RESTART_NODE_CMD`/`SLURM_RESTART_MASTER_CMD` without a password
  prompt (e.g. a `NOPASSWD` sudoers entry scoped to exactly that command) -
  this module has no way to type a password into an SSH session.
- **Provision the local sudo rights `SLURM_ADMIN_USER` needs.** `scontrol
  update NodeName=... State=RESUME` (the Recover action - see below) is a
  Slurm control-plane *write*, not a read, and Slurm enforces its own
  authorization for it independent of the OS: run directly as an arbitrary
  OS user, it fails with `slurm_update error: Invalid user id` - confirmed
  directly against this repo's own dev cluster. `SLURM_ADMIN_USER` (default
  `root`, since Slurm always trusts uid 0 with no `sacctmgr` registration
  needed) tells `resumeNode()` to run the command as that user instead, via
  a local `sudo -u` - which means the backend's own OS user needs
  passwordless sudo rights to run `scontrol` as `SLURM_ADMIN_USER` (a
  `NOPASSWD` sudoers entry). Detection/stats need none of this (those are
  reads).

Both of the above are the cluster administrator's responsibility, set up
directly on the Slurm hosts - not something this repo can orchestrate (same
reasoning as `docs/MFF-Module-template_install.md`'s "no install script" section: infrastructure
provisioning belongs to whoever owns that infrastructure, not to code that
merely uses it).

## SSH setup (restart, Recover, controller log, Config tab)

Module version 0.1.36. SSH settings can only be set in `App_backend/.env`; there is no settings page. Restart the backend afterwards.

```
MODULE_SLURM_SLURM_SSH_KEY_PATH=<absolute path of the private key of the OS user running the backend, chmod 600>
MODULE_SLURM_SLURM_SSH_USER=<login user on the controller and every node>
MODULE_SLURM_SLURM_CONTROLLER_HOST=<controller name the backend can SSH to, e.g. nusbf = SlurmctldHost>
```

- `SLURM_SSH_USER` must hold the sudoers rules described in this document. The default `slurm` normally cannot log in (nologin, no home), so set this explicitly.
- Create the key as the backend's OS user: `ssh-keygen -t ed25519 -N '' -C mff-slurm -f ~/.ssh/mff_slurm`.
- Authorise it on the controller and on each node: `ssh-copy-id -i ~/.ssh/mff_slurm.pub <ssh-user>@<host>`, or append the `.pub` line to `~<ssh-user>/.ssh/authorized_keys`.
- Test: `ssh -i ~/.ssh/mff_slurm -o BatchMode=yes <ssh-user>@<node> 'hostname && sudo -n -l'`.
- A **changed** host key is refused. After confirming the change is legitimate, fix it with `ssh-keygen -R <node>` as the backend user.

When `SLURM_SSH_KEY_PATH` (or `SLURM_CONTROLLER_HOST`, where needed) is unset, or the key file is missing or unreadable by the backend user, Restart node/controller, Recover, the controller log download and the Config tab all return the same message, which names those `.env` variables. In Recover the `ssh` step is then `failed` (not `skipped`), the run snapshot has `setup: { code: 'no_key' | 'key_unreadable', message }`, and the UI shows a highlighted "Setup needed" box; the Slurm resume and wait steps still run. This replaces the earlier wording that Recover "degrades to resume plus wait".

## How detection works

`backend/services/slurmCli.js` shells out to `scontrol show node --json`,
`scontrol show partition --json`, and `squeue --json` on whatever host this
module's backend process runs on - so that host needs the Slurm client
binaries and Munge authentication already configured to reach the cluster's
`slurmctld` (same as any other Slurm client). No `slurmrestd`, no separate
REST layer.

## Restart audit trail

Every restart attempt (single node, all nodes, master, all-nodes+master, or
a node resume) - success or failure - is logged to this module's
`slurm_restart_log` table (who triggered it, what target, and the
command's output/error) - see `backend/db/schema.sql`.

## Node recovery

Module version 0.1.35. The "Recover" button next to a problem node (`NodeTable.js` / `NodeRecovery.js`) runs a guided recovery of that node. A node counts as a problem when it is DOWN, DRAIN or UNK, or NOT_RESPONDING. This now includes a node that is IDLE but NOT_RESPONDING (red chip and Recover button in the node table); previously Recover reported such a node as "recovered".

**API** (administrator only):
- `POST /api/slurm/nodes/:nodeName/recover` starts a background run and answers `202` with the run snapshot: `{ runId, node, status: 'running'|'done', before, after, recovered, failureReason, steps: [{ id, title, status, detail }], logs: { slurmd, controller }, alreadyRunning }`.
- `GET /api/slurm/nodes/:nodeName/recover/:runId` returns the same snapshot (`404` if unknown, for example after a backend restart).
- One run per node at a time: a second POST joins the running one. The node name must be in the live Slurm node list.
- The finished run is written to the Audit tab (category `runtime`, action `node_recover`); the node's previous config content is kept in `before_state`.

**Steps.** Each step has a status `ok`, `fixed`, `warn`, `failed`, `skipped` or `info`:
- `ssh`: reach the node as `SLURM_SSH_USER`. A changed host key gives a clear message: run `ssh-keygen -R <node>` as the backend's OS user.
- `clock`: `warn` if the node's clock differs by more than 30 s.
- `munge`: `systemctl is-active munge`; if not active, restart it with `SLURM_RESTART_MUNGE_CMD`. A credential made on the controller must decode on the node.
- `config`: the files in `SLURM_CONF_FILES` are compared by sha256 with the controller's. A differing or missing copy on the node is replaced with the controller's through `SLURM_CONF_WRITE_CMD` and read back. A file that exists only on the node (for example a node-local gres.conf) is left alone.
- `slurmd`: `systemctl is-active slurmd`; started or restarted with `SLURM_RESTART_NODE_CMD` when inactive, when slurmctld says the node is NOT_RESPONDING, or after a config push or munge restart.
- `boot`: if `systemctl is-enabled slurmd` says disabled, `SLURM_ENABLE_NODE_CMD` enables it. A failure here is only a warning.
- `resume`: `scontrol update State=RESUME` as `SLURM_ADMIN_USER`, only if the state is DOWN/DRAIN/UNK. Skipped when slurmd failed to start.
- `wait`: polls every 3 s, up to `SLURM_RECOVER_TIMEOUT_MS`, until the node is no longer DOWN/DRAIN/UNK/NOT_RESPONDING.
- `why`: only if the node is still unhealthy: slurmctld's reason, a TCP check of the slurmd port (`SLURM_SLURMD_PORT`) from the backend host, the slurmd journal and the tail of `SLURM_SLURMD_LOG_PATH` from the node, and the slurmctld log lines that mention the node, read via the controller.

**Setup.** The `resume` step needs the local sudo rights for `SLURM_ADMIN_USER` described above. The other steps need SSH (see "SSH setup"; without a usable key the `ssh` step fails with a "Setup needed" message and only resume and wait still run). On **every compute node**, the SSH user needs this sudoers entry: `slurm ALL=(root) NOPASSWD: /usr/bin/systemctl restart slurmd, /usr/bin/systemctl restart munge, /usr/bin/systemctl enable slurmd, /usr/bin/install -m 0644 /dev/stdin /etc/slurm/slurm.conf, /usr/bin/install -m 0644 /dev/stdin /etc/slurm/gres.conf, /usr/bin/journalctl -u slurmd -n 40 --no-pager`. The journalctl entry is optional: without it the log file tail is still tried. The SSH user must also be able to run `munge -n` and `unmunge`, and to read `/var/log/slurm/slurmctld.log` on the controller; the controller needs nothing more than read access to its log.

## Logs tab

Downloads the tail of `SLURM_CTLD_LOG_PATH` from `SLURM_CONTROLLER_HOST`
over SSH - same SSH configuration requirement as restarts.

## Admin tabs

Added in module version 0.1.33 (Users-tab changes in 0.1.34). Users, Accounts, Runtime, Config and Audit are for administrator/developer only. They are not governed by Page Access, and the backend routes require the administrator role.

- **Users:** every Slurm user with accounts, default account, GPU/CPU limits (GrpTRES, MaxTRES, MaxJobs, MaxSubmitJobs, GrpJobs), QOS, admin level and a blocked flag. It cross-checks against MFF users: MFF users with no Slurm user (with a one-click "Add to Slurm"), and Slurm users with no MFF user. Writes: add a user to an account (refused if the name has no Unix account according to `getent passwd`; see "Adding a user" below), change the default account, set limits and QOS, block/unblock. Block sets MaxSubmitJobs=0 on the user's associations and stores the previous values in `slurm_user_block`, so Unblock restores them. Nothing is ever deleted: there is no user, account or association deletion anywhere.
- **Accounts:** list and create accounts, set account limits and QOS. QOS are listed, not created.
- **Runtime:** `scontrol update` of partition State/MaxTime/Nodes, and node drain (a reason is required) / resume. This is runtime only: it is lost on a slurmctld restart unless slurm.conf is changed too.
- **Audit:** table `slurm_admin_audit`: who, when, the exact commands, the state before, and the result, for every change in the four tabs above.

Adding a user: `POST /api/slurm/admin/users/add-to-account` takes an optional `gpuQuota` (a whole number, 0 to 100000).
- It sets `GrpTRES=gres/gpu=N` in the same single command: `sacctmgr -i add user <u> Account=<a> GrpTRES=gres/gpu=N`. 0 is accepted (no GPUs).
- For a user who is already a member of the account, `gpuQuota` is applied with `sacctmgr modify user where name=<u> account=<a> set GrpTRES=gres/gpu=N`.
- Without `gpuQuota` and without `makeDefault`, an existing member is still refused ("already belongs to account").
- An invalid value (negative, fractional, non-numeric, boolean) answers `400` "GPU quota must be a whole number of GPUs (0 or more)".

In the Users tab, "Add to Slurm" is one form: User name, Account (defaults to the most common default account), GPU quota (default 1) and a single "Add user" button. Add user, change default account and edit limits (buttons "Add user" / "Save") apply directly: one request, no `dryRun`, no review dialog. The result is shown, or the error inside the form, which stays open, and the `slurm_admin_audit` entry is still written. Block/Unblock keep their confirmation dialog. The Accounts, Runtime and Config tabs are unchanged.

Every write:
- the request body is validated against allowlists: names match `^[A-Za-z0-9][A-Za-z0-9_.-]{0,63}$`; TRES are `key=value` with key in `cpu|mem|node|billing|energy|gres/<n>|license/<n>`; the node, partition, account and QOS must exist;
- it runs with `execFile` argv (never a shell string) as `SLURM_ADMIN_USER` via `sudo -n -u`;
- `dryRun: true` is supported on all admin POST routes: it returns the exact commands and the before-state without running anything or changing anything. The Users tab no longer uses it for add, default account and limits.

HTTP routes (all under `/api/slurm`, administrator only):
- `GET /admin/users`, `/admin/accounts`, `/admin/runtime`, `/admin/audit`
- `POST /admin/users/{add-to-account,default-account,limits,block,unblock}`, `/admin/accounts`, `/admin/accounts/limits`, `/admin/partitions/update`, `/admin/nodes/{drain,resume}`
- `GET /config/files`, `/config/changes`, `/config/changes/:id`
- `POST /config/preview`, `/config/apply`, `/config/changes/:id/rollback`

### How people are shown

In the Users tab, people are shown as "First Last (username)" and sorted by first name, then last name, case-insensitively. Slurm-only accounts without an MFF user are shown by their user name. The Audit tab and the config history show the acting user the same way.

## Editing the Slurm configuration (Config tab)

Edits `slurm.conf` and `gres.conf` (`SLURM_CONF_FILES`). Flow: edit, then Review, then explicit confirmation, then a backup of the previous content in table `slurm_config_change`, then write to the controller first and then every node (5 at a time, read back and hash-verified) over SSH, then make it effective, then a health check, then automatic rollback if anything fails.

**Review** consists of:
- the module's own semantic validation: partitions naming unknown nodes, duplicate nodes/partitions, Gres not in GresTypes, two default partitions, missing SlurmctldHost;
- Slurm's own parser, via `SLURM_CONF=<candidate> scontrol show config`;
- a diff, and what applying will take.

**How it takes effect** (automatic):
- only comments or whitespace changed: nothing;
- a partition or other scalar change: `scontrol reconfigure`;
- a node definition, GresTypes, SelectType, TaskPlugin, AuthType, accounting storage, gres.conf and similar: restart slurmctld, wait for `scontrol ping`, then restart slurmd on every node (rolling, 5 at a time).

The user can force "restart".

**Health check:** slurmctld answers, and every node that was healthy before (and is still defined) is not DOWN/DRAIN/UNK/NOT_RESPONDING within `SLURM_HEALTH_TIMEOUT_MS`. On any failure (write, reconfigure, restart, health) the previous files are restored on every touched host, the daemons are restarted or reconfigured again, and the change ends as `rolled_back`. If even that does not recover the cluster, it ends as `rollback_failed`, with a red banner in the tab, and the previous file stays in the history. Any past change can be restored from the history. Status `interrupted` means the backend stopped mid-change: check by hand.

**Safety rules:**
- one change at a time;
- the file name is allowlisted and the path is always `SLURM_CONF_DIR` + that name;
- an unreachable healthy node means nothing is changed; an unreachable node that is already DOWN is skipped and reported (it keeps the old file);
- settings that run programs or plugins as root (Prolog, Epilog, PrologSlurmctld, EpilogSlurmctld, TaskProlog, TaskEpilog, SrunProlog, SrunEpilog, HealthCheckProgram, ResumeProgram, SuspendProgram, ResumeFailProgram, MailProg, RebootProgram, UnkillableStepProgram, JobSubmitPlugins, Include, and others) cannot be added, changed or removed from the app;
- slurmdbd.conf (it holds the database password) is not editable, so slurmdbd is never restarted by the app (nothing in slurm.conf or gres.conf is read by it).

**What this module does not provision** (the administrator or sysadmin does, like the existing restart setup):
- **Slurm writes (Users/Accounts/Runtime):** passwordless sudo for the backend's OS user, for example `/etc/sudoers.d/mff-slurm-admin`: `<backend-user> ALL=(root) NOPASSWD: /usr/bin/sacctmgr -i *, /usr/bin/scontrol update *, /usr/bin/scontrol reconfigure`. Alternatively point `SLURM_ADMIN_USER` at a dedicated Slurm operator account. This gives that account full Slurm-admin power.
- **Config tab:** the same SSH key and user as the restart buttons (`SLURM_SSH_KEY_PATH`, `SLURM_CONTROLLER_HOST`, the key authorised on the controller and every node, node names resolvable as SSH hosts). On the controller and every node, the SSH user (default `slurm`) also needs: `slurm ALL=(root) NOPASSWD: /usr/bin/install -m 0644 /dev/stdin /etc/slurm/slurm.conf, /usr/bin/install -m 0644 /dev/stdin /etc/slurm/gres.conf` (adjust the directory if `SLURM_CONF_DIR` differs), plus the existing `/usr/bin/systemctl restart slurmd` (nodes) and `/usr/bin/systemctl restart slurmctld` (controller) rules.

**Tests:** `npm test` (node:test, Node 20; it uses stub sacctmgr/scontrol and a simulated cluster for the apply/rollback flow).

## Allocations (GPU quota from finance)

The finance module records GPU allocations: a core group, a number of GPUs, start and end dates, many lines per group (`docs/MFF-Module-Finance_overview.md`, "GPU allocations"). Slurm reads them and sets each mapped group's Slurm account GPU quota, `GrpTRES` `gres/gpu`, to the total of the lines active today (`startDate <= today <= endDate`, both inclusive, `endDate` null = open-ended).
- Only the `gres/gpu` entry of the account's `GrpTRES` is written; other entries (for example `cpu=...`) are kept.
- An ended allocation sets the quota to 0, so access ends on its own. A later start date sets the quota up when it arrives.

**Contract with finance.** Slurm calls finance's `GET /api/finance/gpu-allocations` with `coreApi.callModuleAsUser(adminUserId, 'finance', 'GET', path, { params: { date } })`. `date` is today in the server's local calendar. The path is `SLURM_ALLOC_FINANCE_PATH` (in `App_backend/.env`: `MODULE_SLURM_SLURM_ALLOC_FINANCE_PATH`), default `/gpu-allocations`. The response shape is finance's: see `docs/MFF-Module-Finance_api-and-config.md`. Slurm reads these fields of each entry of `lines`: `id`, `groupId` (the core `setup_groups.id`), `groupName`, `groupNameKnown`, `gpus` (an integer from 1 to 100000), `startDate`, `endDate` (`YYYY-MM-DD`; `endDate` may be null) and `datesAssumed`. Everything else is ignored: since finance 0.1.27 `label` is always null and `purchaseCost`, `yearId` and `yearName` are gone from finance's lines.
- Slurm recomputes each group's active total from the lines and cross-checks it with finance's `groups[].activeGpus` for the same date. A difference leaves that group's account untouched and flags it ("finance reports N GPUs active today ... but its lines add up to M").
- A line with `groupId` null is reported and ignored.
- A line with a bad `gpus` or date value leaves that group's account untouched for the run, and flags it.
- A finance error, a refusal by core, a malformed answer, or an empty `lines` array changes nothing. An empty list is never turned into "0 GPUs".

**Group to Slurm account mapping** (table `slurm_group_account`):
- By default the Slurm account with the same name as the group (case-insensitive, exact).
- An administrator can pick another account, or "Ignore this group", in the Allocations tab.
- A group with no matching account is flagged "No Slurm account" and skipped, never guessed. A mapped account that no longer exists is flagged.
- Several groups mapped to one account add up.
- Finance sends `groupNameKnown`. While it is false (core cannot list groups yet; finance sends the placeholder "Group #<id>" as `groupName`), slurm treats the name as unknown: the group is shown as "group <id>" (marked "name not known yet"), is never matched to a Slurm account by name, and its placeholder name is not stored. An administrator maps it by id in the Allocations tab. When `groupNameKnown` is true, `groupName` is the group's current core name and same-name matching to a Slurm account works.
- A line with `datesAssumed` true (a legacy finance row with no dates, read as its financial year's first and last day) is used as a normal line and flagged "dates assumed" in the tab. The fix is to correct the dates in finance.

**When it runs:**
- The admin-only Allocations tab has "Preview" (shows what would change, changes nothing) and "Sync now" buttons (`POST /api/slurm/admin/allocations/sync`, body `{ dryRun }`).
- A daily run, **off** until an administrator enables it in the tab. Settings: auto on/off and an hour 0-23 in server time (table `slurm_allocation_settings`). It runs once per day after that hour; an errored run is retried after an hour; the scheduler checks every 10 minutes.
- The daily run acts as the last administrator who saved the settings or pressed "Sync now" (`callModuleAsUser` needs a user). If that user is no longer an active administrator, the run log reports the refusal: open the tab and press "Sync now" to set a new one.

**Routes** (all `/api/slurm`, administrator only):
- `GET /admin/allocations`: the view (groups, lines with active/upcoming/ended state, next change date, the account's current quota, mapping status, the finance error if any, last runs)
- `POST /admin/allocations/mapping` `{ groupId, groupName, account|null, ignored }`
- `POST /admin/allocations/settings` `{ autoEnabled, syncHour }`
- `POST /admin/allocations/sync` `{ dryRun }`

**Logging.** Every account change is one row in the Audit tab (category `allocation`, action `allocation_apply`: the GPU quota before and after, the lines used, the exact `sacctmgr` command, the result). Mapping and settings changes are logged too (`allocation_mapping`, `allocation_settings`). Every sync attempt is also a row in `slurm_allocation_run`, shown as "Last run" in the tab. New tables: `slurm_group_account`, `slurm_allocation_settings`, `slurm_allocation_run` (installed by the module Update).

## Page state

Module version 0.1.42 (storage changed in 0.1.43). Every tab keeps its state, per user, when the user leaves the tab or reloads the page. What is kept:
- the selected tab;
- Monitor: the restart-node choice, and the node Recover run (shown again until it is 1 h old or the backend has forgotten it);
- Stats: the granularity;
- Users: the filter, and the open form with its typed values;
- Accounts: the create and limits forms;
- Allocations: the expanded groups;
- Runtime: the partition and drain forms;
- Config: the selected file, the apply mode and the unsaved edit (restored only while the file on the cluster is still the version the edit was started from);
- Audit: the category and the expanded row;
- Page Access: the unsaved grid;
- Job Profiles: the unsaved add rows.

A saved value for something that no longer exists falls back quietly. Cancel and Save forget a form's draft.

**Where it is stored:** by mff-core's shared helper `coreApi.usePageState` (core 0.1.205 or newer: per user, in the browser and on the server; it survives a reload and a logout), with keys of the form `slurm.<page.what>` (`docs/MFF-Core_module-core-api.md`, "`usePageState`"). On an older core without the helper, slurm falls back to the browser's `localStorage` (per user; it survives a reload only). Everything goes through `frontend/components/usePageState.js`. The Config tab's unsaved edits are one saved object for both files; an edit over 60 KB is not saved (core's limit is 64 KB per key).

## External API for other modules

Other modules (e.g. ABCFold) reach this module's Slurm capabilities via
core's generic module-to-module bridge -
`coreApi.callModule(token, 'slurm', method, path, opts)` (see
`docs/MFF-Core_module-core-api.md`) - never a Slurm-specific named wrapper.
Core deliberately doesn't build per-module wrapper functions; `callModule`
is the one proxy every module uses to reach every other module. This
section documents the actual HTTP routes that proxy calls into
(`backend/routes/slurmApiRoutes.js`, mounted at `/api/slurm/external/*` -
i.e. `path` above is `/external/...` - a router deliberately separate from
this module's own UI-facing routes in `backend/routes/slurmRoutes.js`).

`callModule` forwards the calling user's own bearer token on every
request, so `authenticateToken`/`requireSlurmAccess` see the real
originating user, not a service identity - access below is scoped per that
specific person, both for the Slurm-registration gate and for job
ownership. Per `callModule`'s own "never throws" contract, it always
resolves to `{ ok, status, error }` - a caller should check `ok`/`status`
rather than assume a thrown exception ever happens on their end.

### Access gate

Every route below requires (in order):

1. A valid mff-core session (`coreApi.authenticateToken` - the usual
   bearer-token check).
2. **The calling user must be an actual registered Slurm user** - checked
   against `sacctmgr show user --json` (`backend/services/slurmUsers.js`),
   *not* against any mff-core role. A real app user can easily not be a
   real Slurm user, and vice versa. A rejection here looks like:
   ```json
   { "error": "No Slurm access", "reason": "not_slurm_user", "username": "jdoe",
     "message": "MFF user 'jdoe' is not a Slurm user." }
   ```
   status `403`. The answer names the user that was checked, in both `message` and a separate `username` field, plus a `reason`, one of:
   - `not_slurm_user`: no Slurm user of that name. The MFF user name is the Slurm/Unix login, compared case-insensitively.
   - `no_association`: the Slurm user exists but belongs to no account.
   - `blocked`: all the user's associations have MaxSubmitJobs=0. This is enforced only for `POST /jobs`; status and cancel still work.

   If `sacctmgr` itself fails, the answer is `502 { "error": "Failed to verify Slurm access", "username": "...", "message": "..." }`. Callers can show `message` unchanged.

   This is a different rejection from the per-job ownership
   check below - "you're not a Slurm user at all" vs. "this isn't your job".

### `GET /gpu-capacity`

Added in module version 0.1.40. Used by the finance module for its "GPUs allocated vs the cluster's real GPUs" total line. Reached as `/api/slurm/external/gpu-capacity` (`coreApi.callModule(userToken, 'slurm', 'GET', '/external/gpu-capacity')`).

- **Access:** a valid session and one of the roles `developer`, `administrator`, `facility_manager`, `facility_staff`. The caller need **not** be a Slurm user, and there is no page gate (so the Slurm-access gate above does not apply).
- **Response `200`:** `{ totalGpus, availableGpus, basis, nodes, nodesWithGpus, generatedAt }`.
  - `totalGpus` is the sum of the GPUs in the GRES of every node Slurm defines. Typed (`gpu:model:N`) and untyped (`gpu:N`) entries are counted; other GRES are ignored.
  - `availableGpus` is the same sum excluding nodes that are DOWN, DRAIN, UNK or NOT_RESPONDING.
- A Slurm failure answers `502 { success: false, message }`.
- **Module access (since version 0.1.41):** the route is listed in slurm's manifest `publicRoutes` (`{ GET, /external/gpu-capacity }`, the only public route in the manifest; see `docs/MFF-Core_module-manifest.md`), so core's module-level access gate does not apply: a user needs **no** grant on the slurm module. Core lets the request through untouched, so the handler authenticates itself: a valid bearer token (`401` otherwise) and one of the roles above (`403` otherwise). It is never anonymous. This is how finance reads it with `callModule(userToken, 'slurm', 'GET', '/external/gpu-capacity')`, although it cannot list slurm in `requires` (slurm requires finance, which would be circular).

### `GET /resource-profile?gpuCount=N`

The admin-configured `{ gpuCount, partition, gres, cpusPerTask, mem, time }`
policy row for a job requesting `N` GPUs (managed via this module's own
"Job Profiles" admin tab, table `slurm_resource_profile`). Returns JSON
`null` (200) if no profile has been configured for that exact count -
policy is genuinely undefined at that point, not an error.

### `GET /gpu-partitions`

Array of `{ name, nodes, gpusPerNode, gpuModel, gres, gpuMemoryGB }` - which
partitions actually have GPUs, and what they are. `nodes` is this
partition's GPU node count (this route already only returns partitions
that have GPU nodes at all) - named to match ABCFold's own expected
partition-listing shape exactly, so it needs no reshaping on ABCFold's
side. `gpuMemoryGB` is `null` until an admin enters that model's memory
size on the Job Profiles tab (table
`slurm_gpu_model`) - Slurm itself never reports GPU memory size anywhere.

### `POST /jobs`

Submits a job on the calling user's behalf. Body:

```json
{
  "workDir": "/abs/path/to/an/already-prepared/job/directory",
  "jobName": "optional label"
}
```

**This module never creates, writes into, or reads any file or directory
itself.** `workDir` must already exist and already contain a `submit.sh`
script (`backend/services/slurmJobs.js`'s `SCRIPT_FILENAME`) plus any input
files the job needs - written there by the caller (ultimately core's
responsibility, same "requires, never provisions" boundary as
`SLURM_SSH_KEY_PATH` above) *before* calling this route. This module treats
`workDir` as an opaque string, passed straight through to `sbatch --chdir`
- exactly like a node name or job ID, never composed, created, or read by
this module. `workDir` must be readable by every compute node too (the
job's script/input files need to exist wherever it actually runs).

The job's stdout/stderr are always written as fixed filenames - `slurm.out`
/ `slurm.err` - inside `workDir` (via `sbatch --output=... --error=...`),
regardless of what the submitted script itself does or doesn't set, so
whoever owns `workDir` always knows exactly where to find them. Response:
`{ "jobId": 12345 }`. Every submission is recorded in `slurm_submitted_job`
(who submitted it, the `workDir` string it was given) - audit/ownership
metadata only, never read back off disk by this module.

**This module provides no file-listing/download routes** - it never reads
`workDir`'s contents, so it can't serve them back either. Whoever
provisions `workDir` (core) owns serving its contents to a caller, if
that's needed.

### `GET /jobs/:jobId`

Current status: `{ jobId, state, reason, exitCode, submittedAt, startedAt,
endedAt }`, from `sacct -j <jobId> --json` (covers active *and*
completed/failed jobs, unlike `squeue`). Scoped to that job's original
submitter or an mff-core admin - `403 { error: 'Not your job', ... }`
otherwise, `404 { error: 'Unknown job', ... }` if `jobId` was never
submitted through this API at all.

### `POST /jobs/:jobId/cancel`

Cancels a previously-submitted job by running `scancel <jobId>` (binary
overridable via `SLURM_BIN_SCANCEL`). No request body. Same access checks,
in the same order, as `GET /jobs/:jobId`: valid session, then the Slurm
access gate (`403 { error: 'No Slurm access', ... }`), then ownership -
the job's original submitter or an mff-core admin (`403 { error: 'Not your
job', ... }`), `404 { error: 'Unknown job', ... }` if it was never
submitted through this API.

- Success: `200 { "jobId": 12345, "state": "CANCELLED" }` - this is the
  requested outcome; Slurm may take a moment to update, so
  `GET /jobs/:jobId` reports the real state afterwards.
- Job already in a terminal state (`COMPLETED`, `FAILED`, `CANCELLED`,
  `TIMEOUT`, `NODE_FAIL`, `OUT_OF_MEMORY`, `PREEMPTED`, `BOOT_FAIL`,
  `DEADLINE`, `REVOKED`): `409 { "error": "Job already finished", "state":
  "<state>" }`, and `scancel` is not run.
- `scancel` (or the preceding `sacct` state lookup) fails:
  `502 { "error": "Failed to cancel job", "message": "..." }`.

No filesystem access is involved, consistent with this module never
reading or writing `workDir`.
