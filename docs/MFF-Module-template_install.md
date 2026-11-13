# Why there's no install script in this repo

How installing/updating a module actually works (the one-and-only-checkout model, database provisioning, etc.) is core's own mechanism — see `docs/MFF-Core_module-database-provisioning.md`. This file only covers a decision specific to *this* repo.

There used to be an install script here (`scripts/install-into-core.js`) that copied/symlinked this repo's files into an mff-core checkout. That put the module's code in two places on disk and required this repo to know mff-core's internal directory layout — both wrong. A module's code should exist in exactly one place: wherever it's actually `git clone`d. Where that is, and how it gets there, is mff-core's decision to make (see mff-core's own `modulesManagementRoutes.js`) — not something this repo orchestrates.
