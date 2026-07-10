---
name: gpt-5-6-prompting
description: Internal guidance for composing Codex and GPT-5.6 prompts for coding, review, diagnosis, and research tasks inside the Codex Claude Code plugin
user-invocable: false
---

# GPT-5.6 Prompting

Use this skill when `codex:codex-rescue` needs to shape a task for Codex or another GPT-5.6 workflow.

Start with the smallest prompt that reliably completes the task. State the task, authorization, outcome, and only the verification or output requirements that the workflow cannot infer. Use XML tags only when they separate a material contract or preserve a machine-readable response.

Core rules:
- Prefer one clear task per Codex run. Split unrelated asks into separate runs.
- State the expected end state, approval boundary, and any non-negotiable constraint.
- Add grounding or verification only when unsupported claims or an unverified result would be harmful.
- Do not repeat generic brevity, persistence, or friendliness instructions.
- Lead output contracts with the conclusion, required evidence, material caveats, and next action.

Default prompt recipe:
- `<task>`: the concrete job and the relevant repository or failure context.
- `<output_contract>`: only when response shape matters.
- `<autonomy>`: safe local actions and confirmation boundaries.
- `<verification>`: for implementation, diagnosis, or risk-sensitive conclusions.
- `<grounding>`: for review, research, or root-cause analysis.

When to add blocks:
- Coding or debugging: add `autonomy` and `verification`.
- Review: add `grounding` and the required structured output contract.
- Research: add `grounding` and citations when sources are material to the answer.
- Write-capable tasks: authorize in-scope local edits and require confirmation only for destructive, external, or expanded-scope actions.

How to choose prompt shape:
- Use built-in `review` or `adversarial-review` commands when the job is reviewing local git changes. Those prompts already carry the review contract.
- Use `task` when the task is diagnosis, planning, research, or implementation and you need to control the prompt more directly.
- Use `task --resume-last` for follow-up instructions on the same Codex thread. Send only the delta instruction instead of restating the whole prompt unless the direction changed materially.

Working rules:
- Prefer direct requirements over motivational nudges.
- Keep progress updates limited to material phase changes or blockers.
- Keep claims anchored to observed evidence; label inferences.

Prompt assembly checklist:
1. Define the exact task and scope in `<task>`.
2. Choose the smallest output contract that still makes the answer usable.
3. State which local actions are authorized and when confirmation is required.
4. Add verification and grounding only where the task needs them.
5. Remove redundant instructions before sending the prompt.

Reusable blocks live in [references/prompt-blocks.md](references/prompt-blocks.md).
Concrete end-to-end templates live in [references/codex-prompt-recipes.md](references/codex-prompt-recipes.md).
Common failure modes to avoid live in [references/codex-prompt-antipatterns.md](references/codex-prompt-antipatterns.md).
