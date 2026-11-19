# MFF-Module-AlphaPulldown: setup and cluster defaults

Setup values live in `alphapulldown_settings` and are edited in the Setup tab (retrieved by the module itself; nothing is read from an .env).

## Installed stack the defaults are written for (Newcastle cluster)
- AlphaPulldown 2.9.0 + AlphaFold 3 3.0.4 in micromamba env `alphafold3_alphapulldown`; no containers, no Snakemake, so plain scripts under Slurm.
- Weights: /home/linuxhomes/software/alphafold3/models (passed as `--data_dir` to run_multimer_jobs.py).
- Databases: /home/linuxhomes/software/genetic_db (`--data_dir` for feature creation). `pdb_seqres.txt` is in pdb_seqres/ so Setup holds `pdb_seqres_path` = /home/linuxhomes/software/genetic_db/pdb_seqres/pdb_seqres.txt.
- GPU partition `gpu`, gres `gpu:1`; login node has no GPU.

## Setup fields
Environment: manager (micromamba/conda), env name (default alphafold3_alphapulldown). Databases and weights: data_dir, pdb_seqres_path, weights_dir. Features resources: partition, CPUs, memory and time per protein, proteins searched at once. Prediction resources: fold_partition (gpu), fold_gres (gpu:1), fold_cpus_per_task (8), fold_mem (64G), fold_time (24:00:00), fold_unified_memory (on: exports XLA_PYTHON_CLIENT_PREALLOCATE=false, TF_FORCE_UNIFIED_MEMORY=true, XLA_CLIENT_MEM_FRACTION=3.2 so large complexes can spill into host RAM). Limits: warn_max_complex_aa, prediction jobs in flight.

## Notes / troubleshooting
- `run_multimer_jobs.py --num_cycle` is the wrapper's name for AF3 `--num_recycles`; the wrapper default is 3 but AF3's is 10, the module defaults to 10.
- AlphaPulldown's AF3 data pipeline refuses `--use_mmseqs2`; the MMseqs2 option was removed.
- AF3 output per complex: `<fold>_model.cif`, `<fold>_summary_confidences.json`, `<fold>_confidences.json`, `<fold>_ranking_scores.csv`, `seed-N_sample-M/`. Fold folder names are AF3's lower-cased job names.
- Python scripts cannot be run with --help on the login node (no GPU); check flags in site-packages/alphapulldown/scripts/.
- A prediction stuck PENDING usually means no free GPU in the partition; the module only submits, Slurm decides.
- Scores missing in Results: the module gives up looking for the summary file 10 minutes after the job completes; check `output/` and slurm.err in the job folder (Log button in the Prediction tab).
