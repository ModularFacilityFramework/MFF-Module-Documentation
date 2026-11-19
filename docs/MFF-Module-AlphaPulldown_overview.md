# MFF-Module-AlphaPulldown: overview

Module `alphapulldown` (mount `/alphapulldown`, table prefix `alphapulldown_`). `requires: ["slurm", "abcfold"]`. It runs bait-vs-candidates protein screens: feature building (AlphaPulldown) on CPU through the Slurm module, structure prediction on GPU through the ABCFold module. Cite AlphaPulldown2: Molodenskiy et al. 2025, Bioinformatics, doi:10.1093/bioinformatics/btaf115.

## Tabs
- Sequences: UniProt search, FASTA paste/upload, picking proteins from a proteome in the shared library, bait/candidate selection, generated baits.txt / candidates.txt.
- Library: the shared proteomes and feature sets (see the library doc).
- Features (CPU): AlphaPulldown `create_individual_features.py --data_pipeline=alphafold3 --compress_features=True` per protein, against the cluster's genetic databases. One Slurm job per run, proteins searched in parallel with xargs -P inside it. The installed AlphaPulldown 2.9 builds features with AlphaFold 3's data pipeline, which searches local databases only (no MMseqs2 mode); pdb_seqres sits in a subfolder, so its path is passed explicitly (Setup: pdb_seqres_path).
- Co-folding (GPU): one ABCFold job per bait-candidate pair, created with the caller's own token through the ABCFold module. Unchanged by the library work.
- Setup: per-user environment (default env `alphafold3_alphapulldown`, micromamba), database folder (default /home/linuxhomes/software/genetic_db), pdb_seqres file, template cut-off date, Slurm resources; staff also set the shared library folder and feature set tag.
- Staff only: Speed statistics, Page Access (page keys: sequences, library, msa = Features (CPU), cofold, setup).

## No background process
A run only advances while its owner has the page open (the tabs refresh Slurm/ABCFold state when loaded). Submitted jobs keep running on the cluster when the page is closed.

## Workspace layout (user workspace)
module-AlphaPulldown/P<n>/features/ (links to shared features for library proteins, real files for the user's own sequences); msa/run-<id>/ (submit.sh, run_one.sh, capture_env.py, msa_input.fasta, msa_names.txt, msa_lengths.txt, msa_targets.txt, sets.txt, timings.tsv, environment.json, provenance.json, logs/).

## Features format and the ABCFold hand-off
Features are AlphaPulldown/AlphaFold 3 input JSON files, `<name>_af3_input.json.xz`, one protein chain each with unpairedMsa and templates filled. The ABCFold module does not accept any precomputed MSA or features today (it builds its own AF3 input from sequences only; confirmed with its session). The proposed hand-off is a per-chain optional `featuresPath` on POST /jobs, read by the job on the cluster and merged into the generated input; it has been put to the ABCFold session/master. Until it exists the Co-folding stage sends sequences only and the "hand over MSAs" option finds nothing to hand over.

## Provenance
Every feature run and every co-folding run stores a provenance record (DB column and, for feature runs, provenance.json in the run folder): module version, settings, each proteome version used (UPID, organism, taxon, UniProt release and date, content, entry count, SHA-256) with its feature set, and for feature runs the software and database versions the job itself recorded (environment.json). GET /alphapulldown/projects/:id/provenance and the Methods panel under the Features and Co-folding tabs write a methods paragraph from the latest finished feature run and latest co-folding run; unrecorded facts are listed as missing, never guessed.

## Speed database
Each finished feature or co-folding job adds one anonymised row to alphapulldown_speed_samples (sizes, method and hardware labels, run and queue seconds, outcome, day; no user, project, protein name or sequence). Feature-run method label: `af3`. Estimates: under 5 finished samples none, 5-29 low, 30+ good. Staff see a read-only Speed statistics tab.

## Database
backend/db/schema.sql is the source of truth, no migrations. New in this release: alphapulldown_library_config, alphapulldown_library_proteomes, alphapulldown_library_feature_sets, columns proteome_id/accession on project_sequences and provenance_json on msa_runs and cofold_runs; the settings table lost msa_use_mmseqs2 and gained pdb_seqres_path (core only adds tables/columns, so an existing install keeps the old column harmlessly).
