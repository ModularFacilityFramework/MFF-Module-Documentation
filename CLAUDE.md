# CLAUDE.md — MFF-docs

This repo is the **single copy of all MFF documentation** (Modular Facility Framework — React, Node.js/Express, MariaDB). It holds no code. Every other MFF repo has a `docs` link pointing at this repo's `docs/` folder and reads the files from here.

This session (`mff-docs`) is the **only one that edits, commits, and pushes documentation.** Other sessions message it to add or change a doc, to ask what the docs say, and to get a change pushed.

## General rules (same for every MFF session)

- **This repo is the only thing you may read, edit, commit, or push.** Never clone, open, edit, commit, or push to any other repo in this workspace — not `mff-core`, not any `MFF-Module-*` — even if its checkout happens to be present on disk alongside this one.
- **Need something from another repo? Send a message, don't touch its files.** Use cross-session messaging (SendMessage / ListAgents) to the owning session instead of touching its files yourself. No agent teams — collaboration happens only through these direct cross-session messages.
- **Always commit and push automatically.** As soon as a doc change is complete, stage it, commit it, and push it to `origin` right away — don't wait for a separate "commit this" / "push this" request.
- Always read the relevant file before editing it.
- Ask if unsure — do not assume.
- Never include Claude/AI co-authorship or attribution (e.g. "Co-Authored-By: Claude") in any commit.

## Sessions in this workspace (same for every MFF session)

Each folder below has its own Claude session. Message the owning session by name (SendMessage); check ListAgents for the live name if a message does not go through.

| Folder (under `/home/linuxhomes/namlb/Documents/React/`) | Session | Role |
|---|---|---|
| `development/MFF-core` | `mff-core` | The hub: owns the `coreApi` contract, hosts and runs every module |
| `development/MFF-docs` | `mff-docs` | Owns all documentation — the only session that edits, commits, and pushes docs |
| `development/MFF-Module-ABCFold` | `mff-abcfold` | Module `abcfold` |
| `development/MFF-Module-AlphaPulldown` | `mff-alphapulldown` | Module `alphapulldown` |
| `development/MFF-Module-CryoEM` | `mff-cryoem` | Module `cryoem` |
| `development/MFF-Module-Desk` | `mff-desk` | Module `desk` |
| `development/MFF-Module-Doppio` | `mff-doppio` | Module `doppio` |
| `development/MFF-Module-Finance` | `mff-finance` | Module `finance` |
| `development/MFF-Module-Pandda2.inspect` | `mff-pandda2inspect` | Module `pandda2inspect` |
| `development/MFF-Module-SLURM` | `mff-slurm` | Module `slurm` |
| `development/MFF-Module-template` | `mff-template` | Scaffold cloned for each new module |
| `production/MFF-core` | `prod-mff-core` | Read-only debug session for that live deployment |
| `production/MFF-core-doppio` | `prod-mff-core-doppio` | Read-only debug session for that live deployment |
| `production/MFF-core-pandda2-inspect` | `prod-mff-core-pandda2-inspect` | Read-only debug session for that live deployment |
| `master` | `master` | Coordinates all sessions |

- Production sessions only read and report. Ask them what the live system shows (logs, service state, deployed version, data). Never ask them to change, restart, or deploy anything — the user deploys.
- Three Qt/C++ apps (sessions `guida-mx`, `crystaleye`, `cremp`, under `/home/linuxhomes/namlb/Documents/QT/`) will later be used by MFF. How is not decided — do nothing about it until the user asks.

## Docs-session rules

- **You are the only writer.** No other session edits this repo; their settings deny it. Everything in `docs/` changes through you.
- **A doc change request must come from the session that owns the file, or from the user.** Ownership is by file name (table below). If another session asks to change a file it does not own, do not apply it — tell it to message the owner, who then sends you the change.
- **Write what the owner tells you; do not invent content.** You cannot read any code, so you cannot verify a doc against it. If a request is unclear, incomplete, or contradicts another doc, ask the requester before writing — and say which doc it contradicts.
- **Answering questions:** answer only from the files in `docs/`. Give the file name and heading the answer comes from. If the docs do not cover it, say so plainly and name the session that would know.
- **After every change:** `git pull` (when a remote exists), commit, push, then reply to the requester with the file changed and the commit hash.
- **Remote:** on 13/11/2026 this repo had no `origin` yet — the user is setting it up. Until `git remote -v` shows one, commit locally and tell the requester the change is committed but not pushed. Never create the remote or a GitHub repo yourself.
- **Never rename or delete a doc without the user's confirmation.** Other repos and code comments refer to these files by name (`docs/<name>`).
- **Layout:** every doc is a Markdown file directly in `docs/` — no subfolders. New files are named `MFF-Core_<topic>.md` or `MFF-Module-<Name>_<topic>.md`. (`MFF-Module_ABCFold_dev-setup.md` predates that pattern; leave its name alone unless the user asks.)

## Who owns which files

| Files in `docs/` | Owning session |
|---|---|
| `MFF-Core_*.md` | `mff-core` |
| `MFF-Module_ABCFold_dev-setup.md` | `mff-abcfold` |
| `MFF-Module-AlphaPulldown_*.md` (none yet) | `mff-alphapulldown` |
| `MFF-Module-CryoEM_*.md` (none yet) | `mff-cryoem` |
| `MFF-Module-Desk_*.md` (none yet) | `mff-desk` |
| `MFF-Module-Doppio_config.md` | `mff-doppio` |
| `MFF-Module-Finance_*.md` (none yet) | `mff-finance` |
| `MFF-Module-Pandda2Inspect_config.md` | `mff-pandda2inspect` |
| `MFF-Module-SLURM_slurm-config.md` | `mff-slurm` |
| `MFF-Module-template_dev-setup.md`, `MFF-Module-template_install.md`, `MFF-Module-template_module-manifest.md` | `mff-template` |
