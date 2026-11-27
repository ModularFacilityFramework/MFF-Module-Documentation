# AI module: safety and scope

Every answer is checked in the module's backend before anything reaches the model; the model's own instructions are never the only guard.

## Who is who
There are two AI levels (ai 0.1.30, the user's decision):
- **Standard:** everyone.
- **Advanced:** core's administrator and developer roles.

Core's roles decide (security stays in core). There is no "facility staff" level any more, and the "roles that count as facility staff" setting on Setup → Safety is gone. Older stored values are read safely: "user" counts as Standard; "staff", "developer" and "developer-admin" count as Advanced.

The guard, scope check and output scan apply to the Standard level. For the Advanced level the scope check and output scan are off, and the guard is off unless switched on.

## The checks
1. Guard model: Qwen3Guard-Gen-0.6B (Qwen's own, 1.52 GB).
   - It runs as `vllm-guard.service` on gpu6: GPU 3, port 8002, `--gpu-memory-utilization 0.06`, `--max-model-len 8192`, beside the main model. A copy of the unit is in the module repo at `tools/gpu6/vllm-guard.service`. It starts 120 s after the controller.
   - Config: `MODULE_AI_GUARD_URL`, for example `http://<gpu6>:8002`. Optional; without it there is no guard check, and Setup → Safety says so.
   - It reads every question at the Standard level. Unsafe → a fixed refusal naming the category (Violent, Non-violent Illegal Acts such as hacking, Jailbreak, ...).
   - It reads every answer: every 2,000 characters while it is written (without waiting) and once at the end. Unsafe → the answer is stopped and replaced by "This answer was stopped by the safety check"; its thinking and any files it made are dropped. While an answer is being written, up to 2,000 characters can show before it is stopped.
   - A question about self-harm (category Suicide & Self-Harm, unsafe or controversial) gets a fixed kind answer, never a bare refusal: call 999 or 112 in danger; Samaritans 116 123 or jo@samaritans.org; the university's wellbeing and support services.
   - If the guard does not answer: refuse (default; Standard level users get "The safety check is not available right now") or answer without it. Chosen on Setup → Safety.
2. Scope check, by the loaded model: one short classification call, thinking off (for DeepSeek, an empty thought).
   - In scope: MFF and its modules; the facility; its servers and computing; structural biology and the science, calculations and programming it needs; scientific writing; greetings and follow-ups.
   - Anything else gets a polite refusal ("I can only help with MFF, the facility and its servers, structural biology ... and scientific writing ..."), saved with the status "refused".
   - If the check itself fails, the question goes on: it is not a safety guard.
3. Level: each tool (Setup → Tools, "Level") and each skill is Standard or Advanced. Direct tools are locked to Advanced.
   - The backend gives each answer only the tools and skills the asker's level allows. Higher levels also get the lower ones.
   - The model is never shown the rest.
4. Output scan, for the Standard level. It hides these, replacing them with "[hidden]", in answers, thinking and tool results, both while streaming and in what is saved:
   - private and loopback IPv4 addresses;
   - host names on internal domains (.local, .lan, .internal, ...);
   - the AI server's own host names, and the names listed on Setup → Safety;
   - absolute server paths with two parts or more, `~/` paths and Windows share paths.

   Text is shown in pieces that end at white space, so a pattern is never split.

## Developers are not slowed down
- This is the Advanced level: no scope check and no output scan.
- Guard off by default; it can be switched on.
- Measured on 26/11/2026, with 15 normal developer tasks:
  - questions alone: none flagged;
  - one canned answer with root commands: flagged 4 of 15;
  - real Qwen answers to those tasks: none flagged.
- Guard cost: 33 to 94 ms per check.

## Added delay per answer (measured on gpu6, 26/11/2026)
| Check | Time |
|---|---|
| Guard on the question | about 50 ms |
| Scope check | about 130 to 150 ms |
| Guard at the end of the answer | about 70 ms |
| Total at the Standard level | about 0.25 s (average 0.17 to 0.20 s) |
| Total at the Advanced level | 0 |

## Setup → Safety (page setup; changes need administrator or developer)
- Guard status (model, answering or not).
- Switches:
  - guard for the Standard level;
  - guard for the Advanced level;
  - what to do when the guard does not answer;
  - scope check;
  - output scan.
- Extra server names to hide.
- Average and longest added delay per answer, by level, over 1, 7 or 30 days.
- Checks by kind.
- The last 50 questions and answers stopped: when, who, which check, result, category, and the first 8 characters of the answer id.
- Page state kept per user: the unsaved settings, the period and the open sections.
- API: `GET /setup/safety?days=`, `PUT /setup/safety`.

## Data
`ai_safety_log` holds one row per check: answer_id, user_id, level, stage, result, label, categories, ms. Values:
- stage: question, scope, answer, answer_stream;
- result: pass, blocked, self_harm, out_of_scope, error.

It never holds any question or answer text.

## Limits
- The guard sees at most the last 12,000 characters of a question or answer.
- Attached files and images are not read by the guard.
- Files the AI makes are not scanned.
