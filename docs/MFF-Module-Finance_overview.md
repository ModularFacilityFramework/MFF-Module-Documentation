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
Every tab is built except Commercial, which is a placeholder (as in the old app). The Usage tab is built, for MX only for now (see "Usage tab" below). Reports & Stats holds the graphs. The routes, files and configuration behind the tabs are in `MFF-Module-Finance_api-and-config.md`.

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

## Usage tab (MX only for now)

**Data:**
- `finance_usage_mx`: `year_id`, `user_id` (a core user), `login` (the username the robot's usage data carries), `group_id` (a fallback only), `tips` and `rate_basis` (VARCHAR(10) NOT NULL, default `excl`; `excl` = excluding salaries, `direct` = direct cost with salaries included, `ukri` = UKRI PhD). Unique per year and login. `rate_basis` is a column with a DEFAULT on an existing table, so core's schema diffing adds it on Update / Apply Schema.
- `finance_usage_settings`: `year_id` and `mx_tips_per_plate` (default 104).
- `finance_usage_mx_moves`: tips moved off a user's row into another group. Columns: `year_id`, `user_id`, `login`, `group_id` (the target group), `tips` and `rate_basis`. Unique per year, login, group and rate; moving more tops it up.
- `finance_usage_mx_move_log`: an insert-only audit of moves: the year, the user, the login, the from and to group id and name, the tips, the rate, who moved (id and name) and when.
- Both move tables are new `CREATE TABLE`s, picked up by Update / Apply Schema; no existing table changed.

**Rules:**
- A group is never typed or stored as a fact. A user's own row is in the user's current core group (`listUsers().group_id` named by `listGroups()`). A user core no longer lists keeps the group saved with the row. A moved part (see "Move" below) is in the group it was moved to, so a row's group is not always the user's core group.
- `plates` = tips / tips per plate.
- Each row has a Rate drop-down (`rate_basis`). The unit price is the year's Costing Model MX cost per plate of the chosen rate, from the service named "MX": `excl` = non-salary cost / quantity; `direct` = (salary + non-salary) / quantity. With no MX service the prices are empty and a warning is shown.
- A third choice, "UKRI PhD" (`rate_basis` = `ukri`), gives a unit price of £0.00 and a total of £0.00 whatever the Costing Model says, and needs no MX service. Its plates and its % of group still count. Group and overall totals treat it as 0; they stay empty only if another row cannot be priced because the year has no MX service. The same rules apply as for the other choices: a manager-only write, refused (`423`) on a frozen year.
- Total = plates x the unrounded unit price. Group and overall totals add the unrounded row totals and round once.
- "% of group" = the row's plates / the group's plates.
- Groups are sorted by name ("no group" last); people inside a group are sorted by first, then last name.

**Move.** When a user did plates for someone else, each row's "Move" button (manager roles; not on a frozen year) takes a quantity of tips off the row and puts it under another core group.
- The quantity is in tips (tips are the stored unit, with the plates shown beside it); "All" is allowed. The target is picked from a drop-down of the active core groups by name, at a chosen rate (the row's own rate by default).
- The moved part appears in the target group's table, for example "Jean-Lou Reyre (for Josts)", on a tinted row. It counts in that group's plates, totals and % of group. The user's own row keeps the rest. An own row left with 0 tips because everything was moved away is not returned by `GET /usage/year/:id`: it is not in the rows, the group tables, the totals or the summary, and a group with no remaining row disappears. The row is still stored and comes back when tips are moved back, and adding a row for that login reuses the hidden row instead of answering `409`. A row added by hand with 0 tips and never moved stays listed.
- Undo is Move again: a moved row moved to the user's own group goes back onto the user's own row, at that row's rate (a moved row at 0 tips disappears). It can also go on to a third group. If the user's own row is missing, moving back creates it again (same group, at the moved row's rate).
- Every Usage row keeps Move, Edit and Delete: the user's own row and the moved rows ("for <group>"). Editing a moved row can change its tips (a whole number, 1 or more) and/or its rate. If the new rate matches another moved row of the same user, group and rate, the two merge. Deleting a moved row removes it, and its tips are not given back to the user's own row.
- The user's own row is not locked while tips are moved away.
- Edit and delete never touch the move log (`finance_usage_mx_move_log`), which stays as history. These writes are manager-only and refused (`423`) on a frozen year.
- "Move history", collapsible at the bottom of the tab, lists the move log.

**UI:**
- A "Tips per plate" box at the top (manager roles).
- "+ Add row": a user picker ("First Last (username)"), the usage login (defaulting to the username) and the tips. Edit and delete (manager roles). Changing the rate is a manager-only write, refused (`423`) on a frozen year.
- One bordered table per group (a title bar with the group name and its PI(s), the user rows, and a shaded bold total line), separated by space. The columns are User, Usage login, Tips, Plates, Rate, Unit price, Total and % of group. Then a summary table with the columns Group, Users, Tips, Plates, Total and % of total, plus the grand total.
- A "Generate IPR" button in each group's title bar (see "Generate IPR" below).
- A disabled "Upload raw data" placeholder: not implemented, the module has no import.
- A frozen year is read-only. Reading follows the page `usage`.

The 2025-2026 table was loaded once by a throw-away script kept outside the repository (`master/migration/finance`); it is not a module feature.

**Generate IPR.** Each group's title bar has a "Generate IPR" button. It calls `GET /usage/year/:yearId/ipr?groupId=N[&date=YYYY-MM-DD]` and returns the group's prefilled Internal Purchasing Requisition as a `.docx` (no schema step).
- It is read-only: Usage page access only, and it works on a frozen year. The browser sends its own date; otherwise the server's day is used.
- The form is `backend/templates/IPR_Structural_Biology_Template.docx` (converted once from the facility's old Word `.doc`).
- Only the click date (dd/mm/yyyy) and one line per row of the group's Usage table are written. The date goes in the DATE box of the right-hand (selling school) block, the DATE on the REQUESTED BY (NAME) row, above ENTERED BY. The DATE box beside OWN REF NUMBER in the left block stays empty (it was filled in 0.1.41 and 0.1.42). The cell is found by the labels REQUESTED BY (NAME) / DATE. Moved "for <group>" rows are included.
- Line columns: Order Reference = "MX" (fixed); Description = "MX – First Last"; Order Unit = the unit of the year's Costing Model service "MX"; Quantity = the row's plates; Agreed Price = the row's unit price, so it follows the row's selected rate (0 for UKRI PhD); Total Amount = the row's Total.
- After the group's lines there is one more row. Only its Total Amount cell is filled, with the sum of the lines' Total Amounts (2 decimals, added in pence). Every other cell in that row is empty.
- Left blank: Date Goods/Services Required, Account to be Charged and Cost Element. Everything else is the template's.
- The cells are found by their printed labels, so a re-saved copy of the form can replace the template file.

## Page state

Every tab keeps its state when the user leaves and comes back, reloads, or logs out. This is done by one hook, `usePageState(key, default, validate)`, in `frontend/persist.js` (keys are named `finance.<page>.<what>`).

**What is kept:**
- the selected tab (`finance.tab`) and year (`finance.year`);
- the sub-tabs of Costing Model, Recovery, Quotes and Spending;
- in Spending: the Requisitions/Transactions filter and search text, the open requisition, and the expanded reconciliation;
- in Quotes: the status filter;
- in Grants: "show all";
- in Usage: the tips-per-plate draft;
- unsaved form drafts (quote, grant, GPU line, PhD entry, usage row, costing items and rooms, rules, facility profile, requisition notes), cleared on Save or Cancel.

A saved value that no longer exists (a hidden year, a removed sub-tab, a deleted requisition or quote) is ignored silently.

**Where it is stored:** the hook uses mff-core's `coreApi.usePageState` (per user, server side) when the core has it. Until then it uses a `localStorage` stand-in per user id with the same signature (`index.js` calls `configurePersist(coreApi)` once).

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
- Usage: `usage_mx`, `usage_settings` (see "Usage tab" above).

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

All tabs are built except Commercial, which is a placeholder as in the old app. The Usage tab is built, MX only for now. The "+" copy-year button is not built yet. The SQL has been exercised only on a SQLite stand-in, not yet on MariaDB, and the screens have not yet been checked in a browser. Requisition files default to /home/local/nusbf/MFF-core/module-finance/files (see `MFF-Module-Finance_api-and-config.md`).
