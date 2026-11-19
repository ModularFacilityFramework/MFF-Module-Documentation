# MFF-Module-AlphaPulldown: overview

Module `alphapulldown` (mount `/alphapulldown`, table prefix `alphapulldown_`). It runs AlphaPulldown 2.x (https://github.com/KosinskiLab/AlphaPulldown) with the AlphaFold 3 backend as bait-vs-candidates screens on a Slurm cluster. It requires only the `slurm` module (`requires: ["slurm"]`) and submits every job through `coreApi.callModule(token, 'slurm', ...)` using `POST /external/jobs {workDir, jobName}`, `GET /external/jobs/:id` and the cancel route. Citation for users: AlphaPulldown2, Bioinformatics 2025, doi:10.1093/bioinformatics/btaf115.

## Tabs
- Sequences: UniProt search, FASTA paste/upload, proteome download and browsing, bait/candidate selection, generated baits.txt / candidates.txt.
- Features (CPU): runs `create_individual_features.py --data_pipeline=alphafold3 --compress_features=True --seq_index=N` once per distinct protein. Output `<name>_af3_input.json.xz` in `P<n>/features/`. AF3's data pipeline does not support MMseqs2, so searches use the local databases. `--pdb_seqres_database_path` is passed explicitly. One Slurm job per run; proteins are searched in parallel with `xargs -P` inside it. Existing feature files are skipped unless "search again" is ticked.
- Prediction (GPU): plans a run (complex count, already predicted, held back for size over `warn_max_complex_aa`, estimate, resources, script preview) and refuses to start while features are missing. One Slurm job per complex, running `run_multimer_jobs.py --mode=custom --protein_lists=fold_input.txt --monomer_objects_dir=... --output_path=... --data_dir=<weights> --fold_backend=alphafold3 --num_seeds --num_diffusion_samples --num_cycle --jax_compilation_cache_dir`. Modes: bait vs candidates (pulldown), all vs all (every pair INCLUDING each protein with itself, n(n+1)/2), homo-oligomer (`name:copies`). Options: seeds, diffusion samples (default 5), recycles (default 10), limit, bait filter. Max 5,000 complexes per run.
- Results: reads each finished complex's `<fold>_summary_confidences.json` and stores ipTM, pTM, ranking score, best off-diagonal chain_pair_pae_min, clash flag. Sort/filter, download best model (.cif) and summary, CSV export. Output folders are normal AlphaPulldown output, readable by APLit and AlphaJudge.
- Setup: see the setup doc. Staff also get Speed statistics and Page Access (page keys: sequences, msa=Features (CPU), cofold=Prediction (GPU), results).

## No background process
A run only advances while its owner has the page open: loading the Prediction tab refreshes submitted jobs from Slurm, records finished ones in the speed database (once each), reads scores, and submits queued complexes while fewer than `Prediction jobs in flight` run. Closing the page stops new submissions; submitted jobs keep running. A submit failure is retried 3 times then marked SUBMIT_FAILED. Retry failed requeues and clears scores; Cancel cancels via the Slurm module; deleting a run removes only its DB record, never files.

## Workspace layout (under the user's core workspace)
module-AlphaPulldown/P<n>/features/ ; msa/run-<id>/ (submit.sh, run_one.sh, msa_input.fasta, msa_names.txt, msa_lengths.txt, timings.tsv, logs/) ; predictions/run-<id>/job-<item>/ (submit.sh, fold_input.txt, slurm.out, slurm.err, output/) ; predictions/jax-cache/ (shared by the project).

## Speed database
Every finished feature or prediction job adds one anonymised row to `alphapulldown_speed_samples` (sizes in aa, method label, hardware label, run and queue seconds, outcome, day; no user, project, job, name or sequence). Method labels: `af3` (features), `af3-s{seeds}-d{samples}-r{recycles}` (predictions). Hardware: `<partition>/<n>cpu` or `<partition>/<gres with : as ->` e.g. `gpu/gpu-1`. Estimates: under 5 finished samples none, 5-29 low, 30+ good. Staff see a read-only Speed statistics tab (`GET /alphapulldown/speed/summary`).

## Database
`backend/db/schema.sql` is the single source of truth, no migrations. NOTE: this release changed `alphapulldown_settings`, `cofold_runs` and `cofold_items`; an existing install must drop/recreate those tables (no data worth keeping at this stage).
