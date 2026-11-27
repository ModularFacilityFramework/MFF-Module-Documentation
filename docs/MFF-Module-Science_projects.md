# Science module: projects, sharing, materials (API for other modules)

Science owns projects. Other modules never import it. To let a user choose a project they use core's picker
(coreApi.ui.ProjectPicker); to check what a user may do in a project they use coreApi.projects on their backend
(docs/MFF-Core_module-core-api.md, "Projects"). Neither needs "science" in the module's manifest "requires".
A module needs "requires": ["science"] only to call Science's own routes through coreApi.callModule /
callModuleAsUser (paths relative to Science's mountPath /science), in particular to register and remove project
items. Access control is at the application level only: Science sets no file permissions and creates no folders.

## Access rules
- Levels: none < read < write < owner < admin.
- read: sees the project, its items and materials. write: read, plus adds items and links materials, and runs jobs in the project. owner: write, plus renames, archives, shares, and hands the project over. admin: administrators and developers have owner rights on every project.
- A project has exactly one owner (who can hand it over). It is shared with users or groups, read by default, raised to write. Grants add up: the highest one wins (read as a person plus write through a group = write).
- A project the caller cannot read answers 404.
- "Everyone" (Project template only) counts like any other grant: the highest one wins.
- Removing a share, or lowering it from write to read, is always allowed. When it leaves someone with no project, Science first warns the person doing it and names those affected (for a group share, its members).

## Project template
- Science creates one project, "Project template", by itself, once, on a new and on an existing installation: at backend startup, and on the next project list or template call if it was still missing. It needs an administrator or developer to exist. It is found by its row in science_template, never by its title (the owner may rename or archive it; it is not created again). An existing installation needs core's Update (apply schema): new table science_template and new column science_projects.everyone_access.
- Owner: the first (lowest id) active administrator or developer. The owner can edit it and hand it over like any project.
- Shared with everyone (every signed-in user), read only, by default. On its Sharing tab the owner can raise "Everyone" to write, remove it, or add it again. Only the Project template can be shared with everyone.
- It is the example project: a description of how projects work and three example materials (protein, complex, ligand), private to its owner.
- It holds the work modules did before projects existed (see the template routes below). New work does not go there.
- Items in it: each person sees only the items they made (created_by is them), in the list, the item counts and the items read route. Its owner, administrators and developers see all. In every other project, anyone with read access sees all items.
- A module must not widen who can see one of its own records because the record is in the Project template.

## A project before project work
- A person has a project when they can write in at least one that is not archived and is not the Project template (as owner, or shared with them or their group with write; administrators and developers write in every project).
- Someone with no project cannot create one: "+ New project" is hidden in Science, and creating is refused with 403 (Science's page, POST /external/projects, and the provider's create; the provider's canCreate answers false). Administrators and developers can always create. Whether someone who has a project can create more is core's decision on core 0.1.224 or later (coreApi.projects.mayCreate: this rule plus core's setting "people with a project may create more", default yes): Science's page, POST /external/projects, GET /status, GET /external/project-status and the provider's status and requests.create follow it, so with the setting off such a person sends a request instead. On an older core they can create more.
- People can create as many projects as they want: there is no limit on the number, and disk quotas, later, are the only limit planned (not built). So core's setting "people with a project may create more" is yes by default. Someone with no project they can write in still asks for their first one, or gets one at registration approval.
- Modules ask GET /external/project-status before project-based work, and send a person with has_project false to the returned link.

## Request a project
- Science has a third tab, "Project requests". A person who cannot create (no project, or core's setting does not let them create more) fills in: title, a short text (required, at most 2000 characters), optionally a group and a PI (a user whose job position is "PI"). One request can wait at a time; the person can cancel it. A notice on the other Science tabs says they have no project, or that their request is waiting.
- Approvers: administrators, developers, and whoever is granted Science's page "Approve project requests" (key project_approvals; Setup -> Preferences -> Access). They see the requests on the same tab, with a count on its title, and approve or decline with an optional note. Approving creates the project with the requester as owner, with the request's title and text. An approver who is not an administrator or developer cannot decide their own request. A module cannot read or decide requests.
- The requester sees the decision in Science (a notice, and "new" in their list) until they open the tab. On core 0.1.224 or later Science also notifies them with coreApi.notify (title, the decision, the project number or the note, link /dashboard?tab=science). Approvers are not notified of a new request: they see the count in Science.
- A person needs access to the Science module to use this tab.

## Who needs access to the Science module (admin setup)
- Picking a project in another module, and being a member of one, needs no access to Science (core v0.1.218 and
  later, with Science's server provider).
- Adding or removing members, changing their level, handing a project over, renaming, archiving, and the project
  page and materials are in the Science tab only. A project owner without access to Science cannot share their
  project. An administrator must therefore grant Science (Modules -> Access) to everyone who should manage
  projects, for example the role facility_user. With no grants, only developers and administrators can open it.
- A user who creates a project from another module's picker owns it, but needs access to Science to share it.

## Data
- Project: number P-0001, title, optional text, owner, archived flag, folder_path. folder_path is only a stored path <MODULE_SCIENCE_DATA_BASE_DIR>/projects/P-0001_<short-title>, set when that config value exists; Science does not create the folder.
- Materials (proteins, complexes, ligands): one owner and a private/public flag. Public: others find it and link it into their projects. Private: only linkable into projects owned by its owner; it cannot go private while used in another person's project, and cannot be deleted while linked. One record can be in many projects.
- Project item: module_id, item_type, item_id, title, status, link. The project page lists them; an item from a module the viewer cannot use is greyed out ("No access to <module>").
- science_template: one row (slot 1) naming the Project template. science_projects.everyone_access: read, write or empty.
- science_project_requests: one row per request (requester, title, text, group, PI, status pending/approved/declined/cancelled, who decided and when, note, the created project, whether the requester has seen the decision). An existing installation needs core's Update (apply schema) for this table.

## Checking access from another module
coreApi.projects.access(req.user, projectId) answers can_read, can_write, can_manage and the project, and fails
closed. Check it before showing or running anything in a project. Archiving does not lower can_write: before
creating a job or item, also refuse when project.archived is true. Remember the selected project with
coreApi.usePageState (the picker does this through rememberAs).

## Science's /external routes (need "requires": ["science"]; all need a logged-in user; same project checks as the UI)
- POST /external/projects/:id/items {item_type, item_id, title, status?, link?}: needs write. Upsert by
  (module, item_type, item_id). module_id is taken from core's delegation (req.delegatedFrom, or
  req.user.backgroundBy for a background call), never from the body, so a direct call by a user answers 400.
  link must be an in-app path starting with a single "/". 409 if the project is archived.
- DELETE /external/projects/:id/items/:itemType/:itemId: a module removes its own item; needs write.
- GET /external/projects?scope=mine|shared|all&search=&archived=1: [{id, number, title, owner_label, access,
  can_write, can_manage, archived, item_count, updated_at, ...}]. Read-only projects are included with
  can_write false.
- GET /external/projects/:id: one project in the same shape; 404 when absent or not readable.
- POST /external/projects {title, description?}: creates a project owned by the user; 201 with the project.
- GET /external/projects/:id/access: {project: {id, number, title, archived, folder_path}, access, can_read,
  can_write, can_manage}; 404 when the user has no access. folder_path is only available here, not through
  coreApi.projects.
- GET /external/projects/:id/items?module_id=: needs read. [{id, module_id, item_type, item_id, title, status, link, created_by, created_by_label, created_at, updated_at}], newest update first; created_by 0 is shown as "Unknown". Project template: only the caller's own items unless the caller owns it or is an administrator/developer.
- POST /external/projects/:id/items answers 201 {success: true}. item_type max 64, item_id max 128, title max 255, status max 64 characters.
- Project lists and single projects also carry is_template (core's /api/projects does not pass it on).
- GET /external/template: any signed-in user. {id, number, title, archived} of the Project template; creates it if missing; 503 while there is no administrator or developer.
- POST /external/template/items {items: [{item_type, item_id, title, status?, link?, created_by?, created_at?}]}: modules only; needs no access to the template. At most 200 per call. Moves the calling module's project-less items into the template, and updates them later (title, status, link; the first created_by and created_at are kept). Safe to repeat. Answers {project: {id, number, title, archived}, saved, skipped: [{item_type, item_id, reason: "in_another_project" | "invalid", project_id?, error?}]}. "in_another_project": this module already registered the item in that project, so it is not copied. Send created_by = the user id of the person whose work it is; omitted means 0, seen only by the template's owner and administrators/developers.
- DELETE /external/template/items/:itemType/:itemId: modules only. The module removes one of its own items (deleted, or moved to a real project: remove it here, then register it there).
- At module startup there is no request in flight: use callModuleAsUser ("actsAsUsers": true) from a timer that retries (the server may not be listening inside register()), or move the items on the first request with callModule.
- link is an in-app path, for example /dashboard?tab=abcfold.
- GET /external/project-status: any signed-in user. {has_project, project_count, can_create, can_request, link}.
- POST /external/projects answers 403 to someone who cannot create a project.
- Project lists: the Project template comes first, then the other projects by number. Lists and single projects also carry item_counts: {moduleid: n}; in the Project template the own-items rule applies. core's /api/projects (v0.1.223) does not pass item_counts or is_template on.

## Science's own routes for requests (module access needed)
- GET /status: as /external/project-status without link, plus pending_request (or null) and new_decisions.
- GET /request-options: {groups: [{id, label}], pis: [{id, label}]}.
- GET /requests/mine; POST /requests {title, description, group_id?, pi_user_id?} (409 if one is waiting or the person can create); POST /requests/:id/cancel; POST /requests/seen {ids}.
- GET /approvals?status=pending|approved|declined|cancelled|all (default pending) -> {requests, pending}; POST /approvals/:id/approve and /approvals/:id/decline {note?}. Page project_approvals; refused to calls made by a module.
- GET /projects/:id/members/:memberRowId/impact?access=none|read: owner/administrator/developer. {without_project: ["First Last (username)"]}.

## Optional config
MODULE_SCIENCE_DATA_BASE_DIR in core's .env: an absolute path; enables folder_path.

## How core gets Science's projects
- Server provider (core v0.1.218 and later): backend/services/projectsProvider.js, registered in the backend
  register() with coreApi.registerProjectsProvider. list(user, {scope, search, includeArchived}),
  get(user, id) (null when absent or not readable) and create(user, {title, description}) (the user becomes the
  owner; a missing or too long title throws an error with status 400). canCreate and create apply only Science's own rule: canCreate answers false, and create is refused with 403, for someone with no project (administrators and developers excepted; see "A project before project work"). Neither applies core's setting "people with a project may create more": core applies it itself, before calling create, and coreApi.projects.mayCreate calls canCreate, which is why canCreate must not call it. So canCreate can answer true for someone core still does not let create. Core serves it to every signed-in user, so a project member needs no Science grant; Science
  applies its own access rules in each function. The Science tab and its pages (including sharing) stay governed
  by module access.
- For core 0.1.224 or later the server provider also passes is_template and item_counts on each project and offers: items(user, projectId, {moduleId}) (null when not readable), status(user) -> {has_project, can_request, is_approver, pending_request: {id, title, created_at} | null, request_link}, assignment.options(approver, targetUserId) and assignment.apply(approver, targetUserId, {create, grants}) (developers and administrators only; all or nothing), requests.create(user, {title, description, group_id, pi_user_id}), requests.mine(user), requests.cancel(user, id). The signatures are those of core's projects contract (docs/MFF-Core_module-core-api.md, "Projects"). is_approver uses coreApi.hasPage(user, 'project_approvals') (administrators and developers on an older core).
- scope "shared" also lists the Project template for anyone who reads it without owning it.
- Browser provider (fallback, only used when the server has none): frontend/projectsProvider.js, same Project
  shape, over the /external routes. With only this, users without access to Science cannot use the picker.
