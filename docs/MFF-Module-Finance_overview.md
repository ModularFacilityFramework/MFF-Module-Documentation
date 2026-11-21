# Finance module (`finance`)

The facility's finance section as an MFF module: the costing model that sets the charge-out rates, grant
tracking, quotes, spending (requisitions, SAP transactions, PO reconciliation) and cost recovery, with graphs.
Mounted at `/finance`, table prefix `finance_`, own database `mff_module_finance` (provisioned by Install,
see `MFF-Core_module-database-provisioning.md`). Depends on no other module.

## Financial years

Everything is per financial year = academic year, **1 August to 31 July**, named `2025-2026`. Table
`finance_years` (`name`, `start_date`, `end_date`, `is_current`, `hidden`, `frozen`, `frozen_at`; exactly one row is current, enforced by the
API). Every year-scoped table has a `year_id` pointing at it; child rows (allocations, requisition items,
files, reconciliation links) reach the year through their parent. Each tab has a year selector, defaulting to
the current year, ordered oldest on the left and newest on the right (`GET /finance/years` returns `start_date`
ascending). A "+" (add a year) button sits right of the selector, shared by every tab. It is a placeholder: it only
shows "Adding a year is coming soon" and changes nothing. What a new year copies is still to be decided by the user.

A year with `hidden = 1` keeps all its rows, but the module never lists it, answers `404` for it, and leaves its
requisitions out of the reconcile screens and the REQ picker, so nothing of it shows or counts. `hidden`
(TINYINT, default 0) is set by a one-off script run by the user, not by any screen.

### Frozen years

`finance_years` has `frozen` (TINYINT, default 0) and `frozen_at`. The table `finance_year_freeze_log` (`year_id`, `action` `freeze`|`unfreeze`, `user_id`, `user_name`, `created_at`) records each freeze and unfreeze.

A frozen year cannot be changed. Every write and remove route of the module answers `423` with a message (`lib/freeze.js`, the last step of the write/remove guards). This includes a write in an open year that would touch a frozen one: linking a PO to a REQ of a frozen year, moving a grant into it, or undoing a reconciliation that holds its transactions. Reads, facility profiles and the current-year switch stay allowed.

Endpoints (under `/api/finance`):
- `PUT /years/:id/freeze` and `PUT /years/:id/unfreeze` (roles `developer`, `administrator`, `facility_manager`)
- `GET /years/:id/freeze-log`
- `GET /years` also returns `frozen`, `frozen_at` and `frozen_by`

In the UI, a padlock button in the year bar next to the "+" (confirmation dialog; managers only) freezes the year. A frozen year shows a lock icon in the selector, and every tab's edit controls are off. A year is frozen by a user's click, never by a script.

## Tabs (pages)

Declared in `module.manifest.json` `pages`; granted at Setup -> Preferences -> Access (a page nobody has been
granted is developer/administrator only until an admin grants it):
`grants`, `commercial`, `costing_model`, `quotes`, `spending`, `usage`, `recovery`, `reports_stats`.
Every tab is built except Commercial and Usage, which are placeholders (as in the old app). Reports & Stats holds the graphs. The routes, files and configuration behind the tabs are in `MFF-Module-Finance_api-and-config.md`.

## GPU allocations (Grants tab)

The table `finance_gpu_purchases` (the name is kept) holds dated allocation lines, shown in the Grants tab section "GPU allocations": `group_id` (a core group id), `quantity` (the number of GPUs), `start_date` and `end_date` (both inclusive; NULL = open-ended), `pi_user_id` and `notes`. A group can have many lines with different dates. A line belongs to the year selected in the year bar (`year_id` is kept internally, never shown or edited). The `label` and `purchase_cost` columns are gone from `schema.sql`; an already-installed database keeps them, empty and unused.

Rows from before the dates existed have no `start_date`. The API reads them as the financial year's first and last day (`datesAssumed` true), and the Grants tab flags them "⚠ set dates" until edited.

Writes are manager-only (`developer`, `administrator`, `facility_manager`) and a frozen year refuses them; reading is never stopped by a freeze. Other modules (for example slurm) read the lines through `GET /gpu-allocations`; see `MFF-Module-Finance_api-and-config.md`. A group is shown by its current name, read with `coreApi.listGroups()` (mff-core 0.1.197 or newer); while the running core cannot list groups it reads "Group #<id>".

### GPU total line

The Grants tab's GPU allocations table ends with a total row, which is read-only and so also shows on a frozen year. It shows:
- the GPUs allocated by every line active today (`start <= today <= end`, or no end; all visible years; the same sum Slurm makes);
- the number of GPUs the cluster really has;
- the difference: red "oversubscribed by N GPUs (x%)" when allocated is more than the cluster, green "N free" otherwise.

The cluster size is asked of Slurm at runtime: `coreApi.callModule(token, 'slurm', 'GET', '/external/gpu-capacity')`, which answers `{ totalGpus, availableGpus, ... }` (slurm 0.1.40; `totalGpus` is all the GPUs in the GRES of every node Slurm defines). Finance does **not** list `slurm` in `requires`, because slurm requires finance and that would be circular. Finance forwards the user's token through `coreApi.callModule`. Slurm 0.1.41 lists `GET /external/gpu-capacity` in its manifest `publicRoutes`, so core skips slurm's module gate for it and no slurm grant is needed; slurm's own handler still checks the login and the role (`developer`, `administrator`, `facility_manager`, `facility_staff`). A user of another role, a slurm older than 0.1.41, or an unreachable Slurm gives "cluster size unknown", and the row then shows the allocated total. Slurm must be updated and the backend restarted (core reads the manifest at startup). A good answer is cached for 60 s, a failure never. Core logs a warning for the undeclared dependency.

## People pickers

The pickers for PI, supervisors, PhD student and "prepared by" show users as "First Last (username)", sorted by first name, then last name (case-insensitive).
- The PI picker lists only users that core marks `is_pi` (core 0.1.199 or newer; on an older core it falls back to the users listed as a PI of a group). It keeps an already chosen PI even if that user is no longer a PI.
- The supervisor, student and prepared-by pickers keep all users.
- `GET /people` returns the users sorted that way, each with `job_position` and `is_pi`, plus `pi_ids` (see `MFF-Module-Finance_api-and-config.md`).

Tables of people are sorted by first name, then last name (case-insensitive), like the pickers: the Recovery PhD entries and by-staff table, the Grants "PhD students using EM IT" table, and the Costing Model salary rows (`ORDER BY LOWER(staff_name)`). Money and date tables keep their natural order.

## Totals and rounding

Totals (the Costing Model charge total, the Recovery totals, the Summary PhD consumables) are the exact sum rounded once, not the sum of rounded rows. Room cost and yearly depreciation are no longer rounded before summing, so the figures match the old app to the penny. The costing API returns a `totals` object next to the rates.

## Tables (all `finance_`)

- Years: `years`.
- Costing model: `services` (annual quantity = rate denominator, unit, VAT status), `salary`, `non_salary`
  (category, e.g. Consumables / IT / Maintenance / Operating / Training), `depreciation` (asset name, cost,
  life, purchase date), `space_types`, `rooms`, and one allocations table per cost kind
  (`salary_allocations`, `non_salary_allocations`, `depreciation_allocations`, `room_allocations`:
  cost row x service -> fraction 0..1). Charge-out rates are computed, never stored: direct rate =
  (salary + non-salary allocated to the service) / service quantity; FEC rate adds depreciation and estates.
- Recovery: `phd_recoveries` (PhD student, title, up to three supervisors, funder), `phd_recovery_services`
  (quantity per service).
- Grants: `gpu_purchases` (GPU allocations: see "GPU allocations" below), `grants` (costed / active / archived, direct or FEC), `grant_services`.
- Spending: `classification_rules`, `transactions` (university SAP export, unique by MD5 hash),
  `requisitions` (+ `requisition_items`, `requisition_files`), `po_reconciliations`, `po_rec_transactions`
  (a transaction is reconciled at most once), `po_req_links` (one REQ can serve many POs; one PO has one REQ,
  enforced by the API).
- There is no product-category table: a requisition item's `prod_cat` is the 4-character university procurement
  code parsed from the REQ PDF (plain text); the "Category" of a transaction is its costing item
  (`non_salary`, per year).
- Quotes: `facility_profiles` (letterhead), `quotes`, `quote_items`.

Users and groups are core's: every user/group column (`pi_user_id`, `phd_student_user_id`, `supervisor_*_id`,
`reconciled_by`, `created_by`, `group_id`, ...) holds a core id, with no foreign key (another database).

## Access

- Reading a route: logged in and granted the page (`coreApi.requirePage`). Routes shared by all tabs (the year
  list) accept any of the module's pages.
- Writing: the page plus one of `developer`, `administrator`, `facility_manager`, `facility_staff`.
  Deleting: `developer`, `administrator` or `facility_manager`.
- Salary and costing data are only as visible as the page grants make them; grant `costing_model`,
  `recovery` and `spending` only to roles that may see them.

## Config

Module-specific config key: `MODULE_FINANCE_FILES_DIR` (optional; root folder of the requisition PDFs, with a default; see `MFF-Module-Finance_api-and-config.md`). Database credentials: `MODULE_FINANCE_DB_*` in core's `App_backend/.env`,
read through `coreApi.getModuleConfig()`.

## Status

All tabs are built except Commercial and Usage, which are placeholders as in the old app. The "+" copy-year button is not built yet. The SQL has been exercised only on a SQLite stand-in, not yet on MariaDB, and the screens have not yet been checked in a browser. Requisition files default to /home/local/nusbf/MFF-core/module-finance/files (see `MFF-Module-Finance_api-and-config.md`).
