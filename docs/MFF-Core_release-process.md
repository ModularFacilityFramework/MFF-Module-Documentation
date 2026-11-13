# Release Process and Versioning

This document defines how we version, tag, and release the application.

## Versioning Policy

We use Semantic Versioning (SemVer): MAJOR.MINOR.PATCH
- MAJOR: incompatible API changes or significant functionality shifts
- MINOR: backwards-compatible new functionality
- PATCH: backwards-compatible bug fixes and small improvements

## Changelog

We maintain a human-friendly changelog at the repository root: CHANGELOG.md.
Entries follow the Keep a Changelog format.

## Tooling

We use `standard-version` to cut releases and update the changelog and package versions automatically.
- Backend script: `npm run release` in `App_backend`
- Frontend script: `npm run release` in `App_frontend`

## Release Steps

There is no CI pipeline defined in this repo yet — run tests and lint locally before releasing (`npm run lint` at the root, `npm test` in each of `App_backend`/`App_frontend`).

1. Ensure main branch is up to date, tests/lint pass locally, and docs are updated.
2. Choose the appropriate bump:
   - Patch: `npm run release -- --release-as patch`
   - Minor: `npm run release -- --release-as minor`
   - Major: `npm run release -- --release-as major`
3. Run the release in backend and frontend folders:
   - `cd App_backend && npm run release`
   - `cd ../App_frontend && npm run release`
4. Review generated CHANGELOG.md and version bumps.
5. Commit and push tags:
   - `git add . && git commit -m "chore(release): cut new version"`
   - `git push && git push --follow-tags`
6. Deploy the backend and frontend according to your environment (systemd/Apache2/nginx — see the root README).

## Hotfixes

- Branch off from the latest tag (e.g., `hotfix/v1.0.1`)
- Apply the fix and tests
- Create a patch release as above (`--release-as patch`)

## Notes

- Tag names follow `v<version>` (e.g., `v1.0.0`).
- This repo has no migrations — `App_backend/install/schema.sql` is the single source of truth for the schema. If a release changes it, note the change in the release, since operators apply it by re-running the install/upgrade process, not an incremental migration.
- Ensure any new or changed environment variables are documented in `App_backend/.env.example` and `docs/MFF-Core_configuration.md` alongside the release notes.
