# Finance module (`finance`)

The facility's finance section as an MFF module: the costing model that sets the charge-out rates, grant
tracking, quotes, spending (requisitions, SAP transactions, PO reconciliation) and cost recovery, with graphs.
Mounted at `/finance`, table prefix `finance_`, own database `mff_module_finance` (provisioned by Install,
see `MFF-Core_module-database-provisioning.md`). Depends on no other module.

## Financial years

Everything is per financial year = academic year, **1 August to 31 July**, named `2025-2026`. Table
`finance_years` (`name`, `start_date`, `end_date`, `is_current`; exactly one row is current, enforced by the
API). Every year-scoped table has a `year_id` pointing at it; child rows (allocations, requisition items,
files, reconciliation links) reach the year through their parent. Each tab has a year selector, defaulting to
the current year. A "+" button, not built yet, is meant to copy the current year's costing model, PhD allocations
and classification rules as the template for the next year; transactions, requisitions,
reconciliations, grants and quotes are running data and are not copied.

## Tabs (pages)

Declared in `module.manifest.json` `pages`; granted at Setup -> Preferences -> Access (a page nobody has been
granted is developer/administrator only until an admin grants it):
`grants`, `commercial`, `costing_model`, `quotes`, `spending`, `usage`, `recovery`, `reports_stats`.
Every tab is built except Commercial and Usage, which are placeholders (as in the old app). Reports & Stats holds the graphs. The routes, files and configuration behind the tabs are in `MFF-Module-Finance_api-and-config.md`.

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
- Grants: `gpu_purchases`, `grants` (costed / active / archived, direct or FEC), `grant_services`.
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

Module-specific config key: `MODULE_FINANCE_FILES_DIR` (root folder of the requisition PDFs; see `MFF-Module-Finance_api-and-config.md`). Database credentials: `MODULE_FINANCE_DB_*` in core's `App_backend/.env`,
read through `coreApi.getModuleConfig()`.

## Status

Stage 1: schema, financial-year selector and access in place; tab contents are built tab by tab.
