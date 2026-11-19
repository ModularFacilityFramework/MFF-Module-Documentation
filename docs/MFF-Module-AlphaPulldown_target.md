# MFF-Module-AlphaPulldown: Target tab

Step 1 of the new workflow (Target -> Baits -> pairing -> prediction -> ranked summary). The Target tab defines the protein or complex a project is about. It is saved as a draft at any time (valid or not) and confirmed explicitly. The Baits tab exists but is locked until the project's target is confirmed. Baits, pairing and prediction are not built yet.

Words: target = the protein or complex of interest (one or more chains). Genome = the UniProt proteome the target comes from (the baits will later be picked from it).

## Chains
- One chain or many (max 20), each with a copy number (1-24) and a name (letters, digits, . _ -; made unique).
- A chain starts from a UniProt ID (the sequence is fetched into the edit box) or from the user's own sequence (pasted FASTA or plain text, or a file). The text in the box is what is used. UniProt's original sequence is kept beside it, so the edit is described in words ("26 residues removed from the start (1-26)"). If UniProt lists a signal peptide the box offers one-click removal. Isoform accessions (P04637-2) are supported. Retired (inactive) UniProt entries are refused.
- UniProt lookups record: entry name, protein, gene, organism and taxon, reviewed or not, entry and sequence version, proteomes the entry belongs to, and the UniProt release and release date from the X-UniProt-Release headers.

## Genome and the check of each chain
- The user picks a UniProt proteome by typing an organism name, taxon id or proteome id (UP...), or takes the app's suggestion from the organisms of the UniProt chains. What was typed and how it was chosen (search, suggestion, typed id) are recorded, plus the proteome record as UniProt had it (organism, taxon, type, strain, genome assembly, BUSCO, modified date).
- Each chain is checked against the genome on every save (and on "Check again"):
  - UniProt chain: confirmed when the entry's proteome list contains the genome; otherwise confirmed when the entry's taxon equals the genome's taxon; otherwise not confirmed (the entry's organism is named).
  - Own sequence: checked against the shared library's copy of that proteome (newest release, "all" preferred over "reviewed"). Confirmed when a protein is identical, contains the sequence, is contained in it, or is >= 90% identical over >= 80% of the query (point mutants, strain variants). The accession, identity and the library release used are recorded. If the proteome is not in the library the chain is "not checked" and the tab offers to add it.
- Results: matched, not_matched, not_checked, each with a basis and a sentence.

## Chimera
A complex with two or more different chain sequences is chimeric when any chain is not confirmed in the genome. It can still be confirmed, but only after the user acknowledges a clear warning ("a huge grain of salt"). The warning text, the named chains, the user and the time are stored with the target. A single chain or a homo-oligomer is never chimeric (it may still be unchecked).

## 3D model (optional)
- Sources: PDB entry (files.rcsb.org, mmCIF), AlphaFold DB (alphafold.ebi.ac.uk, by UniProt accession), a model predicted with ABCFold (uploaded; picking directly from ABCFold jobs waits for an ABCFold API), or any mmCIF/PDB upload.
- The file is read (mmCIF via the ciftools library, PDB via a small reader), its protein chains' sequences are extracted and matched to the target's chains (identical / chain is part of the model chain / model chain shorter / similar / none, with identity and coverage). A fetched or uploaded model becomes the selected model; the user checks the match and accepts it, or removes it.
- Stored in the user's workspace at module-AlphaPulldown/P<n>/target/model.cif|pdb, with SHA-256, size, source, id, version (AlphaFold DB: v<n>; PDB: last revision date), and fetch date. It is not used for folding (target and baits are co-folded).

## Confirming
Needs: at least one chain, a genome, and acceptance of the model if one is loaded. All chains are re-checked first. Changing the chains or genome of a confirmed target, or its model, reopens it as a draft and clears the chimera acknowledgement; a rename does not.

## Who sees it
Only the owner. Developers and administrators can read every target (the tab lists them all, opened read-only) but cannot change one. Anyone else gets 404. Sharing is core's job, later.

## Record for the methods
The tab shows a methods paragraph built from what was recorded (UniProt accessions with release and date, sequence edits, genome and assembly, which chains were confirmed, chimera acknowledgement, model source/id/version/date, confirmation date) and a history. Unrecorded facts are left out, never guessed. Events table: saved, checked_against_genome, reopened, chimera_acknowledged, confirmed, model_loaded, model_accepted, model_removed.

## API (under /api/alphapulldown)
GET/PUT /projects/:id/target; POST /projects/:id/target/validate; POST .../confirm {acknowledgeChimera}; POST .../model/fetch {source: pdb|alphafold_db, id}; POST .../model/upload?filename=&source=upload|abcfold (body = file text, text/plain); POST .../model/accept; DELETE .../model; GET /uniprot/entry/:accession; GET /target/genomes?q=; staff only: GET /targets.

## Tables (schema.sql)
alphapulldown_targets (one per project), alphapulldown_target_chains, alphapulldown_target_events.
