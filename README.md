# MFF-Module-Documentation

This repository is two things at once:

1. **The single home of all MFF documentation** (Modular Facility Framework — React, Node.js/Express, MariaDB). Every document for MFF-core and for every MFF module lives in [`docs/`](docs/), and nowhere else.
2. **An MFF module, id `documentation`** (label `Documentation`). For now it is only the renamed module scaffold with no features of its own; it is here so the module can later handle the documentation for everything.

## Layout

```
docs/                    THE DOCUMENTATION - one Markdown file per topic, no subfolders
module.manifest.json     module code: id, label, version, tablePrefix, mountPath
backend/                 module code: Express router (index.js), routes/, config/, db/schema.sql
frontend/                module code: React entry (index.js), components/, styles/
package.json             module code: dependencies and scripts
package-lock.json
.eslintrc.json           module code: lint rules (no imports from an mff-core checkout)
.nvmrc                   module code: Node version
.githooks/               module code: optional pre-commit hook inherited from the template (not enabled by default)
README.md                this file
```

Rule of thumb: **everything under `docs/` is documentation; everything else at the root is the `documentation` module's code.** The two do not depend on each other — editing a document never touches the module, and the module does not read `docs/` yet.

## The documentation (`docs/`)

`docs/` is a real folder in this repository. Every other MFF repository has a `docs` link that points at it (`../MFF-docs/docs`), which is why code and READMEs elsewhere refer to documents as `docs/<file name>`. For those links to resolve, this repository must be checked out in a folder named `MFF-docs`, next to the other MFF repositories.

### File names

Every document is a Markdown file directly in `docs/`. The prefix of the file name says what it documents and who owns its content:

| File name | Documents | Content owned by |
|---|---|---|
| `MFF-Core_<topic>.md` | MFF-core | MFF-core |
| `MFF-Module-<Name>_<topic>.md` | the module `<Name>` | that module |

`MFF-Module_ABCFold_dev-setup.md` predates this pattern and keeps its name, because other repositories refer to it.

Do not rename or delete a document without checking who refers to it: other repositories and code comments point at these files by name.

### Index

**MFF-core — using and running it**

| File | Title |
|---|---|
| [`MFF-Core_user-guide.md`](docs/MFF-Core_user-guide.md) | User Guide |
| [`MFF-Core_admin-guide.md`](docs/MFF-Core_admin-guide.md) | Administrator Guide |
| [`MFF-Core_configuration.md`](docs/MFF-Core_configuration.md) | Configuration Management |
| [`MFF-Core_env-policy.md`](docs/MFF-Core_env-policy.md) | Environment variable policy — the one `.env` rule |
| [`MFF-Core_predetermined-mode.md`](docs/MFF-Core_predetermined-mode.md) | Predetermined deployment mode |
| [`MFF-Core_backup-recovery.md`](docs/MFF-Core_backup-recovery.md) | Backup and Recovery |
| [`MFF-Core_troubleshooting.md`](docs/MFF-Core_troubleshooting.md) | Troubleshooting Guide |

**MFF-core — developing it**

| File | Title |
|---|---|
| [`MFF-Core_dev-setup.md`](docs/MFF-Core_dev-setup.md) | Developer Setup Guide |
| [`MFF-Core_api.md`](docs/MFF-Core_api.md) | API Documentation |
| [`MFF-Core_db-schema.md`](docs/MFF-Core_db-schema.md) | Database Schema and Relationships |
| [`MFF-Core_release-process.md`](docs/MFF-Core_release-process.md) | Release Process and Versioning |
| [`MFF-Core_tasks.md`](docs/MFF-Core_tasks.md) | Tasks |

**MFF-core — the contract with modules**

| File | Title |
|---|---|
| [`MFF-Core_module-core-api.md`](docs/MFF-Core_module-core-api.md) | What core actually gives a module (`coreApi`) |
| [`MFF-Core_module-manifest.md`](docs/MFF-Core_module-manifest.md) | `module.manifest.json` schema |
| [`MFF-Core_module-database-provisioning.md`](docs/MFF-Core_module-database-provisioning.md) | Module Database Provisioning |
| [`MFF-Core_module-install-finalize.md`](docs/MFF-Core_module-install-finalize.md) | Module Install / Update / Remove — automatic rebuild and restart |
| [`MFF-Core_module-access.md`](docs/MFF-Core_module-access.md) | Module access — who can see and use an installed module (and one page within it) |
| [`MFF-Core_module-ui-guide.md`](docs/MFF-Core_module-ui-guide.md) | Module UI Guide |

**Module template**

| File | Title |
|---|---|
| [`MFF-Module-template_dev-setup.md`](docs/MFF-Module-template_dev-setup.md) | Dev setup |
| [`MFF-Module-template_install.md`](docs/MFF-Module-template_install.md) | Why there's no install script in this repo |
| [`MFF-Module-template_module-manifest.md`](docs/MFF-Module-template_module-manifest.md) | Renaming this template into a real module |

**Individual modules**

| File | Title |
|---|---|
| [`MFF-Module_ABCFold_dev-setup.md`](docs/MFF-Module_ABCFold_dev-setup.md) | ABCFold — Dev setup |
| [`MFF-Module-AlphaPulldown_overview.md`](docs/MFF-Module-AlphaPulldown_overview.md) | AlphaPulldown module (`alphapulldown`) — overview |
| [`MFF-Module-AlphaPulldown_library.md`](docs/MFF-Module-AlphaPulldown_library.md) | AlphaPulldown module — shared proteome and feature library |
| [`MFF-Module-Doppio_config.md`](docs/MFF-Module-Doppio_config.md) | doppio module configuration |
| [`MFF-Module-Finance_overview.md`](docs/MFF-Module-Finance_overview.md) | Finance module (`finance`) |
| [`MFF-Module-Pandda2Inspect_config.md`](docs/MFF-Module-Pandda2Inspect_config.md) | pandda2inspect module configuration |
| [`MFF-Module-SLURM_slurm-config.md`](docs/MFF-Module-SLURM_slurm-config.md) | SLURM module configuration |

## The module (`documentation`)

Everything outside `docs/` is the scaffold from MFF-Module-template, renamed to module `documentation`:

- `module.manifest.json` — id `documentation`, label `Documentation`, table prefix `documentation_`, mounted at `/documentation`. The schema of this file is core's contract: see [`docs/MFF-Core_module-manifest.md`](docs/MFF-Core_module-manifest.md).
- `backend/` — the module's Express router and its own tables (`documentation_things`, `documentation_page_access` in `backend/db/schema.sql`). `documentationRoutes.js` and the Overview tab are still the template's placeholder example, to be replaced when the module gets real features; `pageAccessRoutes.js` / `PageAccessTab.js` are the template's reusable per-role page-access pattern.
- `frontend/` — the module's React entry point and components.

The module talks to MFF-core only through `coreApi` — see [`docs/MFF-Core_module-core-api.md`](docs/MFF-Core_module-core-api.md). How a module is installed into an MFF-core checkout is core's own mechanism — see [`docs/MFF-Core_module-database-provisioning.md`](docs/MFF-Core_module-database-provisioning.md).

Known gap: the `dev`, `dev:*` and `lint` scripts in `package.json` refer to a `dev/` harness folder that was not part of the template copy this repository was created from, so they do not run here as they stand.
