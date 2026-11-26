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
- **Model:** what is loaded, switching (waits for running answers, or stops them), history, the instruction and sampling values per model. A switch takes 1 to 2 minutes, during which the AI answers nobody.
- **Benchmark:** the same questions asked to several models; speed, automatic checks, 1-5 scores with model names hidden, CSV. The model loaded before is loaded again at the end. Chat questions wait meanwhile.
- **Tools:** install, remove, switch on or off, restrict to roles. Installing downloads nothing. Available: Date and time, Calculator, Unit converter, Read attached file, Edit Excel file, Edit Word file, Make a file. None is installed at first. Only models with the `tools` capability use them.
- **Skills:** short written instructions the AI loads for one kind of task. Only administrators and developers can change them; others with the Setup page get a read-only view. See `docs/MFF-Module-AI_skills.md`.
- **Limits:** every quota and limit. Changes apply within a few seconds.
- **Usage:** figures only, no names and no text.
- **Configuration:** which keys are set (never their values).

## Data
- Per person, in their workspace: `module-AI/chats/<id>/` (chat.json, messages.jsonl, files/), `module-AI/trash/`, `module-AI/uploads-tmp/`. Folders 0700, files 0600.
- Database tables: `ai_settings`, `ai_usage`, `ai_storage_users`, `ai_model_switch_log`, `ai_tool_log`, `ai_benchmark_prompts`, `ai_benchmark_runs`, `ai_benchmark_models`, `ai_benchmark_results`. No chat text.
  - `ai_skills`: skills (the whole SKILL.md, name, description, module tag, on/off).
  - `ai_skill_files`: the reference files of each skill.
  - `ai_skill_log`: one row per skill load (answer id, user, skill, file, model keyword; never any question or answer text).
  - `ai_skill_audit`: who created, changed, switched on or off, or deleted which skill.
- Hourly cleanup (`actsAsUsers` in the manifest): abandoned uploads after a day, deleted chats after the limit "Keep deleted chats". Files of people deleted from MFF are not removed.
- If the backend is restarted while an answer is being written, that answer is lost; the page offers "Answer it now".
