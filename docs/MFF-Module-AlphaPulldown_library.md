# MFF-Module-AlphaPulldown: shared proteome and feature library

One copy of every UniProt proteome and of the features built from it, shared by all MFF-core users, documented so a job's methods can be written from it. Folder: Setup field "Library folder" (staff only; suggested /home/linuxhomes/software/alphapulldown_library/) plus a "feature set tag".

## Layout
<library>/README.txt
<library>/proteomes/<UPID>/<UniProt release>/<all|reviewed>/proteome.fasta and metadata.json
<library>/features/<UPID>_<release>_<subset>_<tag>_<config hash>/set.json, <name>_af3_input.json.xz per protein, .locks/

## Proteome versions (immutable)
A UniProt reference proteome at one UniProt release, all entries or reviewed (Swiss-Prot) only. Downloaded from rest.uniprot.org (uniprotkb/stream) by anyone in the Library tab; if the library already holds that release nothing is downloaded. A newer release is added next to the old one, never over it. metadata.json and the DB row (alphapulldown_library_proteomes) record: organism, taxon id, UniProt proteome id and type, strain, genome assembly, BUSCO completeness, UniProt release and release date (from the X-UniProt-Release headers), content subset, entry count, file size, SHA-256 of proteome.fasta, source, download date, module version. Staff can remove a proteome only when no project protein points at it.

## Feature sets
The input features of ONE proteome version built under ONE configuration: format (af3_json), template cut-off date, database folder and the staff tag; the folder name carries a hash of these. A different Setup value therefore gives a different folder, so features built two ways never mix. Features are built lazily: the first project that needs a protein builds it into the set (under a per-protein lock), later projects only link to it. In the project, `features/<name>_af3_input.json.xz` is a symlink into the set. "Search again" never rebuilds a shared feature.

## What a feature set records
set.json holds the configuration and, written by the first job that builds into it, the environment: AlphaPulldown, AlphaFold 3 and JAX versions, Python, host, and name/size/date of each database file the pipeline reads (uniref90_*.fa, mgy_clusters_*.fa, uniprot_all_*.fa, bfd-first_non_consensus_sequences.fasta, pdb_*_mmcif_files.tar, pdb_seqres, mmcif_files). Every later job (capture_env.py) compares its own environment fingerprint (AlphaPulldown + AlphaFold 3 versions, database file names and sizes) with the recorded one and stops with exit 3 if different (differences printed in slurm.err). Then staff change the feature set tag in Setup: a new feature set starts, the old one is kept.

## Permissions
The folder must be writable by the account that runs mff-core (downloads, set folders) and by the accounts that run users' Slurm jobs (they add feature files and lock files); group-writable with the group bit inherited (the module creates folders as mode 2775 and runs jobs with umask 002).

## Operations
Add a proteome: Library tab, search UniProt, "Add to library". Browse it and add proteins to a project: Sequences tab, Proteome source. Update after a database or software change: bump the tag in Setup. Remove: Library tab, staff, only when unused.
