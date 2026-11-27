# CrystalEye module: projects

Module `crystaleye`. Needs core v0.1.227 or later and `"requires": ["science"]`. There are no plates yet: plates will belong to a Science project, and today the list is an empty placeholder. Not deployed.

## Page
- `coreApi.ui.ProjectTabs` (`rememberAs="crystaleye.dashboard"`, `moduleId="crystaleye"`) is at the top, with the module's sub-tabs (Overview, and Page Access for staff) below it. The chosen project and sub-tab are kept per user.
- Overview lists the plates of the chosen project ("No plates yet in ..."). Nothing project-specific shows until the tabs are ready. Page Access is not per project.
- With an older core the page says "Project tabs need core v0.1.227 or later."
- There are no create forms yet. When there are, they go inside `coreApi.ui.ProjectTask` and follow `selection.canCreate`.

## Backend
- `GET /api/crystaleye/plates?projectId=N` (signed in) returns `{ projectId, project, canWrite, plates: [] }`. 400 for a bad id; 404 when the user cannot read the project (`coreApi.projects.access`); 503 when projects are not available.
- When plate creation is added, the server must check: `access.can_write`, the project is neither `is_template` nor archived, and `coreApi.projects.hasWritableProject(req.user)`. In the Project template a person must see only their own plates, unless they own it or are an administrator or developer.

## Project items
- `backend/services/projectItems.js` registers (`item_type` `plate`, link `/dashboard?tab=crystaleye`) and removes plate items through Science's `/external` routes (`docs/MFF-Module-Science_projects.md`). Nothing calls it yet.
