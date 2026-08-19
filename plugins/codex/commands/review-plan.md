---
description: Review a Claude Code plan via Codex and fix issues found
argument-hint: '[--wait|--resume] [plan-file-path]'
allowed-tools: Read, Glob, Grep, Edit, Bash(node:*)
---

Send a Claude Code implementation plan to Codex for review. If critical issues are found, fix the plan based on the feedback.

Raw slash-command arguments:
`$ARGUMENTS`

Core constraint:
- Your job is to run the Codex review and, if the verdict is not READY, fix the plan file based on the issues found.
- Do not fix issues in any file other than the plan file itself.
- Always display the Codex review output verbatim before any modifications.

Execution modes:
- Default: launch the review in the background, then check results and apply fixes automatically on completion (Steps 1-6).
- `--wait`: run the review in the foreground, then fix immediately.
- `--resume`: recovery mode. Skip Steps 1-5 entirely and follow the Recovery flow at the end of this document. Use it when a previous session ended before a background review completed or before its fixes were applied.

Step 1: Identify the plan file

Follow this priority order to locate the plan file:

1. If the raw arguments contain a file path (a positional argument that is not a flag), use that path.
2. If no path was provided, look for the plan file path in the current conversation context. Plan mode includes the path in the form `.claude/plans/<name>.md` in system messages. Extract that path if present.
3. If still not found, use `Glob` to search for `.claude/plans/*.md` and pick the most recently modified file.
4. If no plan file is found by any method, display this error and stop:

```
## Error: No plan file found

No plan files found in `.claude/plans/`.

Resolution:
- Create a plan using Claude Code's plan mode first
- Or specify the plan file path explicitly: `/codex:review-plan path/to/plan.md`
```

Step 2: Validate the plan file

- Use `Read` to confirm the file exists and is not empty.
- If empty, display an error and stop.

Step 3: Build the review prompt

- Load the prompt template at `${CLAUDE_PLUGIN_ROOT}/prompts/review-plan.md` using `Read`.
- Replace `{{PLAN_PATH}}` with the absolute path to the plan file.
- Build `{{SUPPORTING_CONTEXT}}` before interpolation. Read the plan and collect related source, test, configuration, and documentation files:
  - Start with repository-relative paths explicitly named in the plan, including `Files:` blocks, inline code paths, and referenced tests or documentation.
  - Use `Glob` and `Grep` to find the nearest implementation, test, configuration, and documentation files for the plan's named components when paths are incomplete.
  - Include each selected file as its repository-relative path followed by its content. Prefer direct plan dependencies, their tests, and project guidance/configuration over broad repository dumps.
  - Include at most 12 files or 100,000 characters total. If a selected file would exceed the budget, include only the sections relevant to the plan and mark the omission; never include secrets, `.env` files, generated artifacts, or binary content.
  - If no related file can be resolved, write `No additional relevant repository context was found.`
- Insert the collected content as `<supporting_context>` in the final prompt so Codex receives it with the plan path.
- Store the final prompt string for use in Step 5.

Step 4: Determine execution mode

- Always run in a Claude background task by default. Do not ask the user.
- If the raw arguments include `--wait`, run in the foreground instead (explicit opt-in for synchronous review-and-fix).
- If the raw arguments include `--resume`, this step does not apply; follow the Recovery flow instead.
- Never use `AskUserQuestion` to select the execution mode.

Step 4a: Select the Codex model and reasoning effort

Inspect the plan and choose values before launching Codex. Do not ask the user for this routine selection.

- Use `gpt-5.6-luna` with `medium` for a small, conventional plan in one subsystem with established patterns and low change risk.
- Use `gpt-5.6-terra` with `high` for the normal case: multiple files or components, moderate uncertainty, or meaningful integration and verification work.
- Use `gpt-5.6-sol` with `xhigh` for a cross-cutting, security- or data-sensitive, migration-heavy, or otherwise high-risk plan whose architectural assumptions need deep scrutiny.

Store the chosen values as `<selected-model>` and `<selected-effort>`. Reassess them if the plan changes materially before a later review invocation.

Step 5: Execute Codex review

Use a node one-liner to write the interpolated prompt to a temporary file, then invoke `codex-companion.mjs task --prompt-file` in a single chained command.
This ensures the entire command starts with `node`, matching the `Bash(node:*)` allowed-tools pattern.

Foreground flow:
- Run a single chained command:
```bash
node -e "require('fs').writeFileSync('/tmp/codex-review-plan-$$.md', process.argv[1])" '<the prompt string from Step 3>' && node "${CLAUDE_PLUGIN_ROOT}/scripts/codex-companion.mjs" task --model <selected-model> --effort <selected-effort> --prompt-file /tmp/codex-review-plan-$$.md; rm -f /tmp/codex-review-plan-$$.md
```
- The first `node -e` writes the prompt to a temp file.
- The second `node` runs the Codex task, reading the prompt from that file.
- `rm -f` cleans up regardless of task exit status (uses `;` not `&&`).
- Display the command stdout verbatim to the user.
- Do not paraphrase or add commentary before the review output.

Background flow (default):
- Launch the review with `Bash` in the background using the same chained command pattern:
```typescript
Bash({
  command: `node -e "require('fs').writeFileSync('/tmp/codex-review-plan-$$.md', process.argv[1])" '<the prompt string from Step 3>' && node "${CLAUDE_PLUGIN_ROOT}/scripts/codex-companion.mjs" task --model <selected-model> --effort <selected-effort> --prompt-file /tmp/codex-review-plan-$$.md; rm -f /tmp/codex-review-plan-$$.md`,
  description: "Codex plan review",
  run_in_background: true
})
```
- Do not call `BashOutput` or poll for completion in this turn. Claude Code will notify you when the background task finishes.
- Keep the plan file path from Step 1 in context; you will need it when applying fixes after completion.
- After launching the command, tell the user: "Codex plan review started in the background. Results will be checked and fixes applied automatically once it completes. If this session ends before then, run `/codex:review-plan --resume` in your next session to recover the result and continue fixing."
- When the background task completion notification arrives:
  1. Retrieve the full review output using `BashOutput` for that background task.
  2. Display the Codex review output verbatim before any modifications (same rule as the foreground flow).
  3. Continue to Step 6 and apply fixes if the Verdict is not READY.
- If the background task failed or its output contains no determinable Verdict, display the raw output and stop without editing the plan file.

Important: When embedding the prompt string in the `node -e` command, escape any single quotes in the prompt content by replacing `'` with `'\''` to prevent shell interpretation issues.

Step 6: Fix the plan if needed

After displaying the Codex review output (immediately in foreground mode, or upon the background completion notification in background mode):

1. Extract the Verdict from the output:
   - Look for the `<!-- VERDICT: ... -->` HTML comment marker.
   - If the marker is not found, search the output text for `**READY**`, `**NEEDS_IMPROVEMENT**`, or `**MAJOR_REVISION**` as a fallback.

2. If the Verdict is **READY**:
   - No modifications needed. Stop here.

3. If the Verdict is **NEEDS_IMPROVEMENT** or **MAJOR_REVISION**:
   - Read the plan file using `Read`.
   - For each issue in the Issues section of the review output:
     - Apply the Suggestion using the `Edit` tool on the plan file.
     - Only modify the plan file identified in Step 1. Do not edit any other file.
   - After all edits, report the changes as a brief bulleted list.

Argument handling:
- Preserve the user's `--wait` and `--resume` flags.  (default: `--background`)
- Do not strip them yourself.
- Do not add extra flags or rewrite the user's intent.
- The first positional argument (if present and not a flag) is the plan file path.

Recovery flow (--resume)

Detect an unapplied review result from a previous session and resume fixing the plan. Never launch a new Codex review in this mode.

1. Find the finished review job from the previous session:
   - Run:
   ```bash
   node "${CLAUDE_PLUGIN_ROOT}/scripts/codex-companion.mjs" status --all
   ```
   - `--all` is required: the default status output only lists jobs from the current session, and the job you need belongs to the previous one.
   - Pick the most recent completed plan-review task job. If no finished job exists, tell the user no recoverable review was found and suggest running `/codex:review-plan` normally. Stop.

2. Retrieve the stored review output:
   - Run:
   ```bash
   node "${CLAUDE_PLUGIN_ROOT}/scripts/codex-companion.mjs" result <job-id>
   ```
   - Display the output verbatim before any modifications (same rule as Step 6).

3. Determine whether fixes are still needed:
   - Extract the Verdict using the same rules as Step 6.
   - If READY or undeterminable, no recovery is needed. Stop.
   - Identify the plan file using the Step 1 priority order. The review output usually references the original absolute plan path from the review prompt; prefer that path when present. If `--resume` was given an explicit plan file path, use it.
   - Compare the plan file's last-modified time with the job's completion time. If the plan was modified after the review completed, some fixes may already be applied: verify each issue against the current plan content and skip any that are no longer present.

4. Resume the fixes:
   - Follow Step 6 exactly: apply each remaining issue's Suggestion with the `Edit` tool on the plan file only.
   - After all edits, report the changes as a brief bulleted list.

Recovery limitations:
- Recovery depends on the companion job state for this workspace, which is stored outside the repository and capped at the most recent 50 jobs. Very old reviews may no longer be recoverable; in that case rerun `/codex:review-plan` normally.
- If the Codex session ID is present in the result output, the user can also inspect the original review interactively with `codex resume <session-id>`.
