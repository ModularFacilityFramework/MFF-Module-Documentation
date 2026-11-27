# MFF-Module-AlphaPulldown: overview

Module `alphapulldown` (mount `/alphapulldown`, table prefix `alphapulldown_`). `requires: ["slurm", "abcfold", "science"]`. It runs bait-vs-candidates protein screens: feature building (AlphaPulldown) on CPU through the Slurm module, structure prediction on GPU through the ABCFold module. Cite AlphaPulldown2: Molodenskiy et al. 2025, Bioinformatics, doi:10.1093/bioinformatics/btaf115.

## Tabs
Tab list: Target, Baits, AlphaPulldown, Modelling (ABCFold), Summary, Library, Setup; staff also get Speed statistics and Page Access. Only Target works so far. Baits opens once the selected target is confirmed. AlphaPulldown, Modelling (ABCFold) and Summary are shown but locked ("not built yet").
- Target: define the target (one or more chains from UniProt IDs or own sequences), choose its genome, optionally attach a 3D model, confirm. A project can hold several targets. See the Target doc.
- Baits: locked until the selected target is confirmed; a shell for now (picking baits from the genome comes next).
- Library: the shared proteomes and feature sets (see the library doc).
- Setup: per-user environment (default env `alphafold3_alphapulldown`, micromamba), database folder (default /home/linuxhomes/software/genetic_db), pdb_seqres file, template cut-off date, Slurm resources; staff also set the shared library folder and feature set tag.
- Staff only: Speed statistics, Page Access (page keys: target, baits, alphapulldown, modelling, summary, library, setup).

### Removed tabs
The old Sequences, MSA (CPU) and Co-folding (GPU) tabs were removed from the UI; their code and backend routes remain in the repo but are unreachable from the interface. The page-access list no longer has the keys sequences, msa and cofold. What they did:
- Sequences: UniProt search, FASTA paste/upload, picking proteins from a proteome in the shared library, bait/candidate selection, generated baits.txt / candidates.txt.
- Features (CPU): AlphaPulldown `create_individual_features.py --data_pipeline=alphafold3 --compress_features=True` per protein, against the cluster's genetic databases. One Slurm job per run, proteins searched in parallel with xargs -P inside it. The installed AlphaPulldown 2.9 builds features with AlphaFold 3's data pipeline, which searches local databases only (no MMseqs2 mode); pdb_seqres sits in a subfolder, so its path is passed explicitly (Setup: pdb_seqres_path).
- Co-folding (GPU): one ABCFold job per bait-candidate pair, created with the caller's own token through the ABCFold module. Unchanged by the library work.

## Projects
The module has no projects of its own. Targets, baits and runs belong to a Science project. Manifest: `requires` `["slurm", "abcfold", "science"]`, `actsAsUsers` true.

- Targets and runs are registered as project items (`POST /external/projects/:id/items`; `item_type` `target`, `msa_run`, `cofold_run`; link `/dashboard?tab=alphapulldown`), updated on state changes and removed on delete. If Science cannot be told, the work is still saved and a warning is shown.
- The routes that listed, created and deleted local projects are removed.
- `alphapulldown_projects` is now only the link between one user's rows in one Science project and that user's workspace folder `P<n>` (new columns `science_project_id`, `items_pending`; `alphapulldown_targets.model_user_id`).
- Old local projects are moved in the background after module start. Each becomes a Science project created as its owner (same name), and its targets and runs are listed on it. Runs whose old project no longer exists go to Science's Project template with `created_by` = their owner. A project that cannot be moved (its owner cannot sign in or may not use AlphaPulldown, or Science refuses because the owner has no writable project and so may not create one) stays where it is, shown under the Project template tab to its owner and staff, unchangeable, and is retried for a few hours (9 tries, further apart each time) and again at every module start (the reason is in the backend log). The move is safe to repeat.
- Core's project tabs (`coreApi.ui.ProjectTabs`, core 0.1.227 or later) are at the top of the page; everything below shows only the open project's targets and runs. The new-target form is inside core's `ProjectTask`.
- The server refuses any change in the Project template (even for administrators) and in archived projects. Adding a library proteome needs a writable project of one's own (`coreApi.projects.hasWritableProject`); developers and administrators are exempt.
- The Project template tab shows older work with no Science project (not moved yet, or its old project is gone), read-only: each user their own; developers and administrators everyone's.
- `GET /alphapulldown/projects/:id/runs` lists a project's feature and co-folding runs (all users'), as last seen; read access is enough. The Target tab shows them in a sortable Runs table.
- The staff all-users target list and its route `GET /alphapulldown/targets` are removed.

## Background work
A run only advances while the user who started it has its page open (the tabs refresh Slurm/ABCFold state when loaded), and queued co-folding pairs are submitted only while that user has write access to the Science project. Submitted jobs keep running on the cluster when the page is closed. The module's only background task is the move of old local projects into Science after it starts (see Projects).

## Workspace layout (user workspace)
module-AlphaPulldown/P<n>/features/ (links to shared features for library proteins, real files for the user's own sequences); msa/run-<id>/ (submit.sh, run_one.sh, capture_env.py, msa_input.fasta, msa_names.txt, msa_lengths.txt, msa_targets.txt, sets.txt, timings.tsv, environment.json, provenance.json, logs/).

## Features format and the ABCFold hand-off
Features are AlphaPulldown/AlphaFold 3 input JSON files, `<name>_af3_input.json.xz`, one protein chain each with unpairedMsa and templates filled. The ABCFold module does not accept any precomputed MSA or features today (it builds its own AF3 input from sequences only; confirmed with its session). The proposed hand-off is a per-chain optional `featuresPath` on POST /jobs, read by the job on the cluster and merged into the generated input; it has been put to the ABCFold session/master. Until it exists the Co-folding stage sends sequences only and the "hand over MSAs" option finds nothing to hand over.

## Provenance
Every feature run and every co-folding run stores a provenance record (DB column and, for feature runs, provenance.json in the run folder): module version, settings, each proteome version used (UPID, organism, taxon, UniProt release and date, content, entry count, SHA-256) with its feature set, and for feature runs the software and database versions the job itself recorded (environment.json). GET /alphapulldown/projects/:id/provenance (`:id` is the Science project id; read access to the project is enough, and it reads the caller's own runs in it) and the Methods panel under the Features and Co-folding tabs write a methods paragraph from the latest finished feature run and latest co-folding run; unrecorded facts are listed as missing, never guessed.

## Speed database
Each finished feature or co-folding job adds one anonymised row to alphapulldown_speed_samples (sizes, method and hardware labels, run and queue seconds, outcome, day; no user, project, protein name or sequence). Feature-run method label: `af3`. Estimates: under 5 finished samples none, 5-29 low, 30+ good. Staff see a read-only Speed statistics tab.

## Database
backend/db/schema.sql is the source of truth, no migrations. New in this release: alphapulldown_library_config, alphapulldown_library_proteomes, alphapulldown_library_feature_sets, columns proteome_id/accession on project_sequences and provenance_json on msa_runs and cofold_runs; the settings table lost msa_use_mmseqs2 and gained pdb_seqres_path (core only adds tables/columns, so an existing install keeps the old column harmlessly).

Target tables: alphapulldown_targets, alphapulldown_target_chains, alphapulldown_target_events.

Added for Science projects (v0.1.23): columns science_project_id and items_pending on alphapulldown_projects, and model_user_id on alphapulldown_targets.

## Dependencies
ciftools 0.2.5 (mmCIF reading) was added to the module's package.json.
