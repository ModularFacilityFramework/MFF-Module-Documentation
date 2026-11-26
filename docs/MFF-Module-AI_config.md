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
- **Tools:** install, remove, switch on or off, restrict to roles. Installing downloads nothing. Available: Date and time, Calculator, Unit converter, Read attached file, Edit Excel file, Edit Word file, Make a file. None is installed at first. Only models with the `tools` capability use them.
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
- Where the docs come from: `coreApi.docs.list()` and `coreApi.docs.read(name)` when core offers them; otherwise the folder in `MODULE_AI_DOCS_DIR` (optional config key). Only top-level `*.md` files are read, and the list is cached for 60 s.
- Who may read which doc: `docs/INDEX.json`, `{ "files": { "<name>.md": { "audience": "user"|"staff"|"developer", "summary": "..." } } }`. Without an entry, `*_user-guide.md` files are for everyone and every other doc is for developers only. The check is made in the backend for the asker; a doc above the asker's level is never listed, searched or read.
- Setup → Tools: "MFF docs the AI can read" lists each doc with its area, audience and summary. "Tools used in answers" lists the last 50 tool uses.
- Data: `ai_tool_use` has one row per tool call (answer_id, user_id, level, tool, target, ok, keyword). `target` is a doc or skill name only, never the question, the arguments or the result.
- Also new on Setup → Tools and Skills: "What the AI can do" lists each tool and skill with a status: tested (tried with real questions on the AI server), done (built), or planned.

## Data
- Per person, in their workspace: `module-AI/chats/<id>/` (chat.json, messages.jsonl, files/), `module-AI/trash/`, `module-AI/uploads-tmp/`. Folders 0700, files 0600.
- Database tables: `ai_settings`, `ai_usage`, `ai_storage_users`, `ai_model_switch_log`, `ai_tool_log`, `ai_benchmark_prompts`, `ai_benchmark_runs`, `ai_benchmark_models`, `ai_benchmark_results`. No chat text.
  - `ai_skills`: skills (the whole SKILL.md, name, description, module tag, on/off).
  - `ai_skill_files`: the reference files of each skill.
  - `ai_skill_log`: one row per skill load (answer id, user, skill, file, model keyword; never any question or answer text).
  - `ai_skill_audit`: who created, changed, switched on or off, or deleted which skill.
  - `ai_skills.audience` (new column): who the skill is for (everyone, facility staff or developers).
  - `ai_safety_log`: one row per safety check; never any question or answer text.
- Hourly cleanup (`actsAsUsers` in the manifest): abandoned uploads after a day, deleted chats after the limit "Keep deleted chats". Files of people deleted from MFF are not removed.
- If the backend is restarted while an answer is being written, that answer is lost; the page offers "Answer it now".
