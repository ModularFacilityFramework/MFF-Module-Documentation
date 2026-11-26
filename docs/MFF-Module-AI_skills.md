# AI module: skills

A skill is a short set of written instructions for one kind of task, for example how the facility wants a buffer recipe written. Skills are built like Claude Code skills.

## What a skill is
- SKILL.md: a frontmatter block, then the steps:
  ```
  ---
  name: buffer-recipe
  description: Use when someone asks how to make a buffer from stock solutions (what it does and exactly when to use it).
  ---
  1. First step ...
  ```
- name: lowercase letters, digits and hyphens; at most 64 characters; unique.
- description: required; at most 1024 characters. It is what the AI reads to decide when to use the skill.
- Steps: required. The whole SKILL.md is at most 8,000 characters.
- Reference files (optional): at most 10, plain text, at most 50,000 characters each. A skill and its files together are at most 250,000 characters. File names use letters, digits, ".", "-" and "_".
- Module tag (optional): a label such as slurm or bookings, used to filter the list.
- For: everyone, facility staff, or developers (default everyone).

## How the AI uses skills
- Every answer from a model that can use tools gets, in its system prompt, the name and description of each skill that is on and meant for the asker's level (a staff skill is offered to staff and developers, a developer skill to developers only). At most 40 skills are listed (by name); each description is cut to 300 characters there.
- When a question matches a description, the AI calls the tool load_skill(name), gets the steps, and follows them. If the steps say so, it reads a reference file with read_skill_file(skill, file, start).
- These two tools come with skills. They need no install on the Tools tab, and they work even if no other tool is installed.
- Models that cannot use tools (Mistral, DeepSeek) never see skills and answer as before.
- The chat shows a skill load like any other tool use ("Used 2 tools: Load skill, Read skill file").
- A skill loaded in an earlier answer shows up shortened in later questions; the AI is told to load it again when it needs the steps.

## Setup -> Skills (core page setup)
- Who can see it: anyone with access to the Setup page. Only administrators and developers can create, edit, switch on or off, or delete; everyone else gets a read-only view.
- What it shows:
  - the list, with search, a module tag filter and On/Off;
  - "New skill", which opens an editor with a template; tabs for SKILL.md and each reference file, "+ Reference file", and a For choice;
  - "Skills loaded in answers": the last 50 loads, with when, person ("First Last (username)"), skill, file (SKILL.md or the reference file), model, and the first 8 characters of the answer id;
  - "History of changes": create, edit, rename, enable, disable, delete; who did it and when.
- Page state kept per user: the sub-tab, the filter, the open sections, the file open in the editor, and the unsaved draft. Unchanged reference files are not copied into the draft. A draft larger than about 60 KB is kept only while the page is open, and the page says so.
- If the skill being edited is deleted meanwhile, the draft is kept as a new skill, with a notice.

## Storage
Skills live in the module's own database:
- `ai_skills`: the whole SKILL.md, with name and description copied from its frontmatter, plus the tag and on/off.
- `ai_skill_files`: the reference files.
- `ai_skill_log`: one row per load. It holds the answer id, user_id, skill, file_name (NULL for SKILL.md) and model keyword, but never any question or answer text.
- `ai_skill_audit`: the changes.

## API (all under `<api>/ai`, page setup; writes need administrator or developer)
- `GET /setup/skills`: the list, limits, template, recent loads and recent changes.
- `GET /setup/skills/:id`
- `POST /setup/skills` `{content, tag, audience: 'user'|'staff'|'developer', enabled, files:[{name, content}]}`
- `PUT /setup/skills/:id` with the same body. It replaces all files of the skill.
- `POST /setup/skills/:id/enabled` `{enabled}`
- `DELETE /setup/skills/:id`
