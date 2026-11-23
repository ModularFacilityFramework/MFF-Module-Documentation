# Science module: projects, sharing, materials (API for other modules)

Science owns projects. A module that works in projects lists "science" in its manifest "requires" and calls Science through coreApi.callModule / callModuleAsUser (paths relative to Science's mountPath /science). Access control is at the application level only: Science sets no file permissions and creates no folders.

## Access rules
- Levels: none < read < write < owner < admin.
- read: sees the project, its items and materials. write: read, plus adds items and links materials, and runs jobs in the project. owner: write, plus renames, archives, shares, and hands the project over. admin: administrators and developers have owner rights on every project.
- A project has exactly one owner (who can hand it over). It is shared with users or groups, read by default, raised to write. Grants add up: the highest one wins (read as a person plus write through a group = write).
- A project the caller cannot read answers 404.

## Data
- Project: number P-0001, title, optional text, owner, archived flag, folder_path. folder_path is only a stored path <MODULE_SCIENCE_DATA_BASE_DIR>/projects/P-0001_<short-title>, set when that config value exists; Science does not create the folder.
- Materials (proteins, complexes, ligands): one owner and a private/public flag. Public: others find it and link it into their projects. Private: only linkable into projects owned by its owner; it cannot go private while used in another person's project, and cannot be deleted while linked. One record can be in many projects.
- Project item: module_id, item_type, item_id, title, status, link. The project page lists them; an item from a module the viewer cannot use is greyed out ("No access to <module>").

## API for modules (all need a logged-in user; same project checks as the UI)
- GET /external/projects?scope=mine|shared|all&search=&archived=1: [{id, number, title, owner_label, access, can_write, can_manage, archived, item_count, updated_at}]. Read-only projects are included with can_write false so a picker can grey them out.
- GET /external/projects/:id/access: {project: {id, number, title, archived, folder_path}, access, can_read, can_write, can_manage}; 404 when the user has no access.
- POST /external/projects/:id/items {item_type, item_id, title, status?, link?}: needs write. Upsert by (module, item_type, item_id). module_id is taken from core's delegation (req.delegatedFrom, or req.user.backgroundBy for a background call), never from the body, so a direct call by a user answers 400. link must be an in-app path starting with a single "/". 409 if the project is archived.
- DELETE /external/projects/:id/items/:itemType/:itemId: a module removes its own item; needs write.

A module checks access before it shows or runs anything in a project, and remembers the selected project with coreApi.usePageState.

## Optional config
MODULE_SCIENCE_DATA_BASE_DIR in core's .env: an absolute path; enables folder_path.

## Frontend hook for core's picker
frontend/projectsProvider.js: list({scope, search, includeArchived}), get(id), create({title, description}). Science calls coreApi.registerProjectsProvider(provider) when core offers it.
