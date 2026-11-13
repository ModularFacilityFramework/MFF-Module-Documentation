# Renaming this template into a real module

The manifest schema itself (what each field means, how it's enforced) is core's own contract — see `docs/MFF-Core_module-manifest.md`. This is just the module-template-specific checklist for turning this template's placeholder values into a real module:

1. Pick a real `id` (short, lowercase, matches your `tablePrefix` minus the trailing underscore) and update all fields in `module.manifest.json` accordingly.
2. Rename every `example_*` table in `backend/db/schema.sql` to `<id>_*`.
3. Update `dev/.env.dev.example`'s `DB_NAME` default to `<id>_dev`.
4. `package.json`'s `name` field, for tidiness (not load-bearing anywhere).
