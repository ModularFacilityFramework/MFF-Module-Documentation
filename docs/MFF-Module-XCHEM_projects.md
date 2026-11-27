# XCHEM: projects and campaigns

State at XCHEM v0.1.10 (commit 4a2138f). Needs MFF-core 0.1.227 or later and the Science module (manifest `"requires": ["science"]`). XCHEM is for fragment-screening campaigns; so far they can only be listed, not created.

## Screen
- Core's `coreApi.ui.ProjectTabs` (`rememberAs="xchem.dashboard"`, `moduleId="xchem"`) is at the top of the XCHEM page. XCHEM's own tabs (Overview, and Page Access for staff) are shown inside it. See `docs/MFF-Core_module-core-api.md`, "Projects".
- Overview lists the campaigns of the selected project (columns Campaign, Status, Created; sortable). Nothing project-specific is shown until the project tabs have loaded.
- Kept per user: the project tab, the XCHEM tab (`xchem.dashboard.tab`), the sort of the campaign table (`xchem.campaigns.table`) and of the Page Access table (`xchem.pageAccess.table`), and unsaved Page Access edits (`xchem.pageAccess.draft`).

## Backend
- `GET /api/xchem/campaigns?project_id=N`: needs a logged-in user. 200 with the project's campaigns (`id, project_id, title, status, created_at`, newest first) when `coreApi.projects.access` gives `can_read`; 404 `Project not found` otherwise; 400 without a valid `project_id`; 503 when projects are not available; 500 on a database error.
- Table `xchem_campaigns` (`backend/db/schema.sql`): `id, project_id, title, status, created_by, created_at`. `project_id` is Science's project id.
- Database credentials come from `coreApi.getModuleConfig()`. XCHEM has no config keys of its own.

## Science project items
`backend/services/projectItems.js` has `registerCampaignItem` and `removeCampaignItem` (item type `campaign`, link `/dashboard?tab=xchem`), following `docs/MFF-Module-Science_projects.md`. No route calls them yet.

## Rule for future create or change routes
The browser lock is only a screen aid. The server must check `coreApi.projects` (write access; refuse the Project template and archived projects) before creating or changing a campaign, and the forms go inside `coreApi.ui.ProjectTask`.
