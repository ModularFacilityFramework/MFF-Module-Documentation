# AI module: installation and administration

## Configuration (`App_backend/.env`)
Add these by hand, then restart the backend. There is no page for them.

| Key | Required | Meaning |
|---|---|---|
| `MODULE_AI_VLLM_URL` | yes | vLLM server, e.g. `http://<gpu host>:8000` |
| `MODULE_AI_CTL_SSH_HOST` | yes | GPU host with `ai-ctl` |
| `MODULE_AI_CTL_SSH_USER` | yes | account holding the forced-command key |
| `MODULE_AI_CTL_SSH_KEY_PATH` | yes | private key that may only run `ai-ctl`, readable by the backend's user only |
| `MODULE_AI_CTL_SSH_KNOWN_HOSTS` | yes | known_hosts file with the GPU host's key |
| `MODULE_AI_MODELS` | yes | `keyword=Label\|capabilities;...`, capabilities from `vision`, `thinking`, `thinking-r1`, `tools`. Keywords must match the controller |
| `MODULE_AI_GUARD_URL` | no | guard model server, e.g. `http://<gpu6>:8002`; without it there is no guard check (`docs/MFF-Module-AI_safety.md`) |
| `MODULE_AI_DOCS_DIR` | no | folder with the docs, used when core has none (see "MFF docs tools") |
| `MODULE_AI_VLLM_API_KEY` | no | only if vLLM was started with a key |
| `MODULE_AI_CTL_SSH_PORT` | no | default 22 |
| `MODULE_AI_STORAGE_SUBDIR` | no | folder in each person's workspace, default `module-AI` |
| `MODULE_AI_REQUEST_TIMEOUT_S` | no | default 60 |
| `MODULE_AI_SWITCH_TIMEOUT_S` | no | default 600 |

Core's `WORKSPACE_BASE_DIR` must be set, and "workspace storage" ticked for each person who may chat.

## Access
- Chat: core's normal module access.
- Setup: the manifest page `setup` (Setup → Preferences → Access; developer and administrator until granted).
- On Setup, limits, tools, benchmark questions and the switch list can be changed by administrators and developers only. Switching the model and running benchmarks: those two, plus the roles and people added under Model → "Who may switch the model" (they also need the Setup page).

## Setup tabs
- **Model:** what is loaded, switching (waits for running answers, or stops them), history, the instruction and sampling values per model. A switch takes about 2 to 3 minutes (about 10 minutes the first time a mode starts at a new context length), during which the AI answers nobody.
- **Benchmark:** the same questions asked to several models; speed, automatic checks, 1-5 scores with model names hidden, CSV. The model loaded before is loaded again at the end. Chat questions wait meanwhile.
- **Tools:** install, remove, switch on or off. Each installed tool has two limits: For (everyone, facility staff, or developers and administrators; the asker's level from Setup → Safety), and optionally roles (tick core roles to restrict it further). A tool is offered only when both allow the asker. Installing downloads nothing. Available: Date and time, Calculator, Unit converter, Read attached file, Edit Excel file, Edit Word file, Make a file. None is installed at first. Only models with the `tools` capability use them.
- **Skills:** short written instructions the AI loads for one kind of task. Only administrators and developers can change them; others with the Setup page get a read-only view. See `docs/MFF-Module-AI_skills.md`.
- **Safety:** the safety and scope checks, which roles count as facility staff, extra server names to hide, added delay and the questions and answers stopped. Changes need administrator or developer. See `docs/MFF-Module-AI_safety.md`.
- **Limits:** every quota and limit. Changes apply within a few seconds.
- **Usage:** figures only, no names and no text.
- **Configuration:** which keys are set (never their values).

## Qwen precision and context
Qwen3.6-27B can run in three modes. Each mode is its own keyword in the vLLM controller on gpu6, and all of them are served under the model name "qwen", so the chat sees the same model with the same capabilities. All three fit the full 262,144 tokens the model was trained for.

| Mode | Controller keyword | Weights | KV cache | Context | KV cache size | Full-length requests at once | Decode, 1 request | Reading a long prompt |
|---|---|---|---|---|---|---|---|---|
| bf16 (default) | `qwen` | bf16, `/home/ai/models/llm/Qwen3.6-27B`, 51.1 GiB | bf16 | 262,144 | 504,123 tokens | 1.92 | 26.6 tokens/s | 3,171 tokens/s |
| bf16 + FP8 KV | `qwen-fp8kv` | bf16 (same folder) | FP8 | 262,144 | 994,920 tokens | 3.80 | 26.8 tokens/s | 1,462 tokens/s |
| FP8 + FP8 KV | `qwen-fp8` | FP8, Qwen's official Qwen/Qwen3.6-27B-FP8 (revision a56219b8, 30.89 GB) in `/home/ai/models/llm/Qwen3.6-27B-FP8`, 28.5 GiB on the GPU | FP8 | 262,144 | 1,715,433 tokens | 6.54 | 46.7 tokens/s | 1,501 tokens/s |

- Measured on 26/11/2026 on gpu6 GPU 3 (RTX PRO 6000 Blackwell, 96 GB), vLLM 0.26.0, `--gpu-memory-utilization 0.9`. In every mode, a 260,530-token prompt with a fact hidden near the start was answered correctly.
- Going past 262,144 tokens would need YaRN rope scaling, which is not used.
- The two FP8 KV modes use `--attention-backend TRITON_ATTN`. FlashInfer cannot build its FP8-KV kernels in the gpu6 vLLM environment (nvcc 13.3 against CUDA 13.0 headers). That is also why a long prompt is read about half as fast in those modes.
- The first start of a mode at a new context length compiles for about 10 minutes. After that the compile cache is used, and a switch takes about 2 to 3 minutes.
- Before 26/11/2026 every model was served with `--max-model-len 32768`. Mistral and DeepSeek still are.
- The controller's table (keyword, model folder, `--max-model-len`, flags) lives in `/home/ai/vllm_controller.cpp`. A copy is kept in the module repo at `tools/gpu6/vllm_controller.cpp`. Changing it means rebuilding the controller (`g++ -std=c++17 -O2 -pthread`) and restarting `vllm-controller.service`, which reloads the default model.
- `ai-ctl info` now also returns `controller_context` (keyword -> `--max-model-len`) and each model folder's `quant_method`.

Setup -> Model (switch permission as before):
- Switching: the model choice lists Qwen once per mode ("Qwen3.6 27B: FP8 weights, FP8 KV cache, up to 262,144 tokens"), next to Mistral and DeepSeek. A mode is offered only when the controller knows its keyword.
- What is loaded: the header shows the loaded mode. The "Loaded model" table has a Precision row (for example "FP8 weights, FP8 KV cache"), read from the running `vllm serve` arguments and the model folder's config.
- History: the switch history names the mode of both sides of each switch.
- API: `POST <api>/ai/setup/switch` `{keyword:'qwen', variant:'qwen-fp8'|'qwen-fp8kv'|'qwen' (optional), mode}`. A variant that belongs to another model, or that the server does not offer, gives `400`.
- Page state: the chosen target persists per user (`ai.setup.switchTarget`).

## MFF docs tools
- Tools: `docs_index` (list of docs with title, area and summary, with an optional filter), `search_docs` (best matching sections, with extracts) and `read_doc` (one section by heading, or the whole doc in 8,000-character parts).
- They are installed, switched on and given an audience on Setup → Tools like any tool. The AI is told to look in the docs first for questions about MFF, a module, procedures or servers, and to name the doc it used.
- Where the docs come from: `coreApi.docs` (core 0.1.219: `available`, `list`, `read`) when core has docs. Core has docs only where an MFF-docs checkout sits beside the core checkout; production has none today. Otherwise the folder in `MODULE_AI_DOCS_DIR`. Where neither has docs, the docs tools are not offered, the AI says the docs are not available on this server, and Setup → Tools shows a warning.
- Who may read which doc (the user's rule, 26/11/2026): `*_user-guide.md` files are for everyone; every other doc (install, admin, security, server, deployment, API) is for facility staff and developers. `INDEX.json` in `MODULE_AI_DOCS_DIR`, if it ever exists, can set an audience per file. The check is in the backend: a doc above the asker's level is never listed, searched or read. A user guide that names another doc only shows its name, never its text.
- Setup → Tools: "MFF docs the AI can read" lists each doc with its area, audience and summary. "Tools used in answers" lists the last 50 tool uses.
- Data: `ai_tool_use` has one row per tool call (answer_id, user_id, level, tool, target, ok, keyword). `target` is a doc or skill name only, never the question, the arguments or the result.
- Also new on Setup → Tools and Skills: "What the AI can do" lists each tool and skill with a status: tested (tried with real questions on the AI server), done (built), or planned.

## Slurm tools
- Tools: `slurm_queue` (your own jobs, with id, name, state, reason, partition, GPUs, CPUs, memory and times, plus anonymous cluster totals; facility staff and developers can ask for every job), `slurm_job` (the state of one job by id: state, reason, exit code, times) and `slurm_cluster` (per partition: node states and free/total GPUs and CPUs; per node, with its state reason, for staff and developers).
- Each is installed, switched on and given an audience on Setup → Tools.
- They call the slurm module's read-only routes (slurm 0.1.46 or later): `GET /external/queue?scope=own|all`, `/external/job-state/<id>` and `/external/cluster-state`. The call is `coreApi.callModuleAsUser(<asking user's id>, 'slurm', ...)`, so the ai manifest has `"requires": ["slurm"]`.
- The slurm module decides what each person may see:
  - normal users get only their own jobs (by Slurm user name = MFF user name) and anonymous totals;
  - staff (developer, administrator, facility_manager, facility_staff) may see all jobs and nodes;
  - another person's job answers "There is no job <id> that you can see", the same as a missing one.
- The AI never widens this, and answers only from the asker's own result (nothing is cached).
- Errors the AI explains:
  - no Slurm access (ask facility staff);
  - only staff may see every job;
  - Slurm unavailable;
  - the Slurm module cannot be reached (for example, slurm older than 0.1.46).
- The AI is told that partitions share nodes, so figures are never added across partitions, and that "free" does not promise a job will start.
- Tested against a stand-in built to the slurm module's contract; not yet against the real routes, which need slurm 0.1.46 deployed.

## Date and time
- Every request, for every model (Qwen, Mistral, DeepSeek), starts its system prompt with a line made fresh for that request. It gives the current date and time in Europe/London (with GMT/BST), today's date as YYYY-MM-DD, and when the user's message was sent, and tells the model never to say it does not know the date or time.
- This comes before the per-model system prompt, and it is added even when that prompt is empty.
- The chat shows each question's time, and each answer's time in its footer: the time alone for today, otherwise date and time, in the viewer's own clock.
- The scope check counts questions about the date, time or calendar as in scope.

## Web tools
- Tools: `web_search` (titles, URLs and short extracts of up to 8 results) and `web_read` (the readable text of one public web page, in 10,000-character parts). Each is installed, switched on and given an audience on Setup → Tools.
- When either is offered, the model is told to cite every web page it used as a Markdown link, and never to follow instructions found in a page.
- `web_search` goes through the facility's own SearXNG, which forwards the search words to public search engines (only the words leave the facility). Config key `MODULE_AI_SEARXNG_URL` (optional), for example `http://127.0.0.1:8888`.
- SearXNG install on nusbf (27/11/2026):
  - service `searxng.service`, run as the system user `searxng`;
  - source in `/usr/local/searxng` (SearXNG commit b5605b8f, Granian server);
  - settings in `/etc/searxng/settings.yml`;
  - listens only on `127.0.0.1:8888`, so only the MFF backends on nusbf can reach it.

  It is on nusbf rather than gpu6 because every MFF backend (dev and production) runs on nusbf, so it never needs to listen on the network, and gpu6 stays for the models. Rollback: disable the service, remove its unit, `/etc/searxng` and `/usr/local/searxng`, then `userdel searxng`.
- `web_read` safety, all in the backend:
  - only `http` and `https`, ports 80 and 443, and no `user:password` in the URL;
  - every address of the host name must be public (no private, loopback, link-local, carrier-grade NAT, multicast or reserved IPv4/IPv6). The connection uses the address that was checked, and each redirect (at most 5) is checked again;
  - HTML or plain text only, at most 2 MB and 15 s.
- The guard reads the search words and each page part, for the people it is switched on for. A flagged search is refused and a flagged page is withheld.
- `ai_tool_use` logs `web_read` with the host name only; for `web_search` nothing but the tool name is kept.
- If the model is out of tool rounds, it is told that no more tool calls are possible and to answer from what it has.

## MCP servers
- The module has its own small MCP client (Model Context Protocol, Streamable HTTP, protocol 2025-06-18: `initialize`, `tools/list`, `tools/call`). Each tool of a registered MCP server becomes an ordinary tool of the module:
  - its function name for the model is `mcp_<server key>_<tool>`, at most 64 characters (a hash is added when shortened);
  - it is installed, switched on and given an audience (and optionally roles) on Setup → Tools like any tool;
  - the guard reads the arguments the AI sends and the answer that comes back (an unsafe request is stopped, an unsafe answer withheld);
  - every use is logged per answer in `ai_tool_use` (tool, server key).
- Setup → Tools → "MCP servers" (changes need administrator or developer): add a server (key: lowercase, digits, `_`; name; `http(s)` address), switch it off or on, or remove it. The table shows each server's number of tools or its error. A "Source" filter shows the tools of one server. Saved in `ai_settings` `'mcp.servers'`; tool lists are cached 5 minutes.
- The facility's servers run on the backend host (nusbf), `127.0.0.1` only, as the system user `aimcp`. Install and rollback: `tools/nusbf/mcp-install.sh` and `mcp-rollback.sh` in the module repo.
  - `ai-mcp-pdbe.service`: the official PDBe MCP servers from EMBL-EBI (pdbe-mcp-server 2.0.1, Apache-2.0), at `http://127.0.0.1:8901/api/` (46 PDBe API tools: entries, molecules, ligands, validation, UniProt mappings, best structures, AI summaries, ...) and `http://127.0.0.1:8901/search/` (2 tools: PDBe Solr search and its schema).
  - `ai-mcp-biomcp.service`: BioMCP (biomcp-cli 0.9.3, MIT, GenomOncology; not the unrelated PyPI "biomcp"), at `http://127.0.0.1:8902/mcp`. It has 7 read-only tools over PubMed, Europe PMC, PubTator3, UniProt, InterPro, STRING, PDB, AlphaFold and more. No API keys are set, and each data source keeps its own terms of use.
- After a deploy, add the three servers on Setup → Tools and install the tools wanted. Not all 46 PDBe API tools need be on: each tool's description takes room in every question.

## Code sandbox and gemmi
- `run_python` runs a Python 3 program written by the AI.
  - Packages: numpy, scipy, pandas, matplotlib, biopython, gemmi, rdkit.
  - Chat files named in the call are at `/work/in/<name>`, read-only.
  - Files the program saves in `/work/out` (at most 10, 20 MB in all) are given to the person as new chat files, and images are shown in the answer.
  - stdout and stderr go back to the AI.
- Isolation: each run is a bubblewrap sandbox (no network; its own process tree; no `/home`, `/etc` or other files; only `/usr` read-only and a fresh work folder) inside a systemd user scope: 2 GB memory with no swap, 64 processes, 2 CPUs, stopped after 60 s, plus prlimit for CPU seconds and a 50 MB file size. At most 2 runs at once; the work folder is deleted afterwards.
- Runtime: `/usr/local/ai-sandbox` on the backend host (Python venv, root-owned, read-only for the sandbox; package versions in `requirements.lock`). Installed by `tools/nusbf/sandbox-install.sh`, which also enables lingering for the backend's user (`namlb`) so its systemd user manager is always there; rollback: `sandbox-rollback.sh`.
- `read_structure_file` summarises an attached file with gemmi 0.7.5, in the same sandbox:
  - MTZ: space group, cell, resolution range, reflections, columns, datasets;
  - mmCIF/PDB: space group, cell, chains, ligands, waters, entities, and R factors, resolution and method from the header;
  - CCP4/MRC maps.
- Uploads now accept MTZ files. The model sees their gemmi summary, or a short note if the sandbox is not available.

## OCR
- Scanned PDFs: a PDF with no text layer is read at upload with ocrmypdf 16.7 (Tesseract 5.5, English), first 30 pages, in the code sandbox (4 CPUs, 4 GB, 2 minutes). The model gets the text marked "read by OCR; may contain errors". If even OCR finds nothing, the upload is refused with a message.
- Images: every uploaded image is also read with Tesseract in the sandbox. The text is kept only when it has real words. A model that can see images (Qwen, Mistral) reads the image itself; a model that cannot (DeepSeek) gets the OCR text instead of "an image this model cannot see".
- Installed from Debian on the backend host by `tools/nusbf/ocr-install.sh` (`ocrmypdf`, `tesseract-ocr`, `tesseract-ocr-eng` and dependencies). `ocr-rollback.sh` removes exactly the packages that script added. More languages: one `tesseract-ocr-<lang>` package each.

## Speech to text
- What the user sees: a 🎤 Speak button in the chat. The browser records (MediaRecorder; it needs `https`) and the module sends the recording to Whisper on the AI server. The text is added to the question box for the person to check and send. When sent, the question goes through the guard like any other.
- The "Speech to text (microphone)" entry on Setup → Tools switches the button on, with an audience and roles. It is never offered to the model as a tool. `GET /ai/config` returns `speech: true/false` for the person.
- Route: `POST <api>/ai/transcribe` with the recording as the body. Limits: 25 MB, 10 minutes, 30 recordings per 10 minutes per person.
- Errors: `403` not switched on for you; `400` nothing recorded; `413` too long; `422` unreadable; `503` Whisper not available.
- Log: `ai_transcribe_log` has one row per recording (user, seconds of audio, ms, ok) and no text. The recording is not kept.
- Whisper service on gpu6:
  - model: Whisper large-v3-turbo (`dropbox-dash/faster-whisper-large-v3-turbo`, MIT), with faster-whisper 1.2.1 and 8-bit weights (`int8_float16`);
  - placement: GPU 3, next to Qwen and the guard, using 1.6 GB;
  - service: `ai-whisper.service` on `192.168.1.16:8003` (private address), started 150 s after the controller and guard;
  - a facility vocabulary hint (Krios, apoferritin, HEPES, ...) helps with technical words;
  - speed: about 0.2–0.7 s for 8 s of speech;
  - files: `tools/gpu6/whisper_server.py`, `whisper-install.sh`, `whisper-rollback.sh` (`--all` also removes the venv and model).
- Config key `MODULE_AI_WHISPER_URL` (optional), for example `http://192.168.1.16:8003`.

## Database tool (developers only)
- `db_query_mariadb` lets the AI read MFF's MariaDB through a dedicated read-only account created by the user. Config keys (all optional): `MODULE_AI_MARIADB_RO_HOST` (default `127.0.0.1`), `MODULE_AI_MARIADB_RO_PORT` (default 3306), `MODULE_AI_MARIADB_RO_USER`, `MODULE_AI_MARIADB_RO_PASSWORD`. What the account may read is decided by its grants. The user creates it with `SELECT` only, on the databases and tables chosen, never on tables with password hashes or tokens.
- Only developers and administrators can use it. The module enforces this in code, so Setup → Tools refuses a wider audience.
- Each call is limited in the backend:
  - one statement: `SELECT`, `WITH`, `SHOW`, `DESCRIBE` or `EXPLAIN`. Refused: `INTO OUTFILE`/`DUMPFILE`, `LOAD_FILE`, `SLEEP`, `BENCHMARK`, locks, `FOR UPDATE` and `/*!` version comments;
  - run in a read-only transaction that is always rolled back;
  - `max_statement_time` 10 s;
  - at most 200 rows, with long cells shortened.
- Every query is logged in `ai_db_query_log`: answer id, user, database, the query text, rows returned, ms, ok, error.
- The MSSQL server on nusbf is not connected yet: it waits for its own read-only account.

## Data
- Per person, in their workspace: `module-AI/chats/<id>/` (chat.json, messages.jsonl, files/), `module-AI/trash/`, `module-AI/uploads-tmp/`. Folders 0700, files 0600.
- Database tables: `ai_settings`, `ai_usage`, `ai_storage_users`, `ai_model_switch_log`, `ai_tool_log`, `ai_benchmark_prompts`, `ai_benchmark_runs`, `ai_benchmark_models`, `ai_benchmark_results`. No chat text.
  - `ai_skills`: skills (the whole SKILL.md, name, description, module tag, on/off).
  - `ai_skill_files`: the reference files of each skill.
  - `ai_skill_log`: one row per skill load (answer id, user, skill, file, model keyword; never any question or answer text).
  - `ai_skill_audit`: who created, changed, switched on or off, or deleted which skill.
  - `ai_skills.audience` (new column): who the skill is for (everyone, facility staff or developers).
  - `ai_safety_log`: one row per safety check; never any question or answer text.
  - `ai_tool_use`: one row per tool call (answer id, user, level, tool, target, ok, model keyword); `target` is a doc or skill name only.
- Hourly cleanup (`actsAsUsers` in the manifest): abandoned uploads after a day, deleted chats after the limit "Keep deleted chats". Files of people deleted from MFF are not removed.
- If the backend is restarted while an answer is being written, that answer is lost; the page offers "Answer it now".
