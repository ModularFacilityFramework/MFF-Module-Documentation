# Renaming this template into a real module

The manifest schema itself (what each field means, how it's enforced) is core's own contract — see `docs/MFF-Core_module-manifest.md`. This is just the module-template-specific checklist for turning this template's placeholder values into a real module:

1. Pick a real `id` (short, lowercase, matches your `tablePrefix` minus the trailing underscore) and update all fields in `module.manifest.json` accordingly.
2. Rename every `example_*` table in `backend/db/schema.sql` to `<id>_*`.
3. Replace the hard-coded `example` names in the code to match the new id:
   - `backend/routes/exampleRoutes.js` and `backend/routes/pageAccessRoutes.js`: the table names in the SQL (`example_things`, `example_page_access`) and the `/api/example/...` paths in the comments.
   - `frontend/components/Dashboard.js`, `frontend/components/OverviewTab.js`, `frontend/components/PageAccessTab.js`: the API paths `/example/things` and `/example/page-access`. Their first segment must equal the manifest's `mountPath` — core mounts the module's router at `<apiBasePath><mountPath>` (see `docs/MFF-Core_module-manifest.md`), so it is `mountPath` that decides the path, not `id` as such. With the usual `mountPath` of `/<id>` the two are the same.
   - `frontend/components/OverviewTab.js`: the heading text "Example Module".
   - Optional tidy-up: the file and identifier names `exampleRoutes.js` / `buildExampleRouter` (required in `backend/index.js`).
4. `package.json`'s `name` field, for tidiness (not load-bearing anywhere).
