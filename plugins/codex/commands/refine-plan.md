---
description: Repeatedly review and refine a Claude Code plan via Codex until no critical issues remain (max 5 rounds)
argument-hint: '[--resume] [plan-file-path]'
allowed-tools: Read, Glob, Grep, Edit, Bash(node:*), AskUserQuestion
---

Iteratively review and refine a Claude Code implementation plan by running Codex reviews and applying fixes until no critical issues remain, or the maximum number of rounds is reached.

Raw slash-command arguments:
`$ARGUMENTS`

Execution modes:
- Default: run the refinement loop from Round 1 (Steps 1-4).
- `--resume`: recovery mode. Do not start a new loop from Round 1; follow the Recovery flow at the end of this document instead. Use it when a previous session ended mid-loop (typically after a Codex review completed but before its fixes were applied).

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
- Or specify the plan file path explicitly: `/codex:refine-plan path/to/plan.md`
```

Step 2: Validate the plan file

- Use `Read` to confirm the file exists and is not empty.
- If empty, display an error and stop.

Step 3: Refinement loop (max 5 rounds)

For each round (1 through 5):

1. Display `## Round N/5` to indicate progress.

2. **Collect supporting context**: Read the current plan and build `{{SUPPORTING_CONTEXT}}` for this round before calling Codex:
   - Start with repository-relative paths explicitly named in the plan, including `Files:` blocks, inline code paths, and referenced tests or documentation.
   - Use `Glob` and `Grep` to find the nearest source, test, configuration, and documentation files for plan components when paths are incomplete.
   - Include each selected file as its repository-relative path followed by its content. Prefer direct plan dependencies, their tests, and project guidance/configuration over broad repository dumps.
   - Include at most 12 files or 100,000 characters total. For an oversized file, include only plan-relevant sections and mark the omission; never include secrets, `.env` files, generated artifacts, or binary content.
   - If no related file can be resolved, write `No additional relevant repository context was found.`
   - Interpolate the collected content into the review template's `<supporting_context>` block.

3. **Select model and reasoning effort**: Inspect the current plan before this round and choose values without asking the user:
   - Use `gpt-5.6-luna` with `medium` for a small, conventional plan in one subsystem with established patterns and low change risk.
   - Use `gpt-5.6-terra` with `high` for the normal case: multiple files or components, moderate uncertainty, or meaningful integration and verification work.
   - Use `gpt-5.6-sol` with `xhigh` for a cross-cutting, security- or data-sensitive, migration-heavy, or otherwise high-risk plan whose architectural assumptions need deep scrutiny.
   - Store the values as `<selected-model>` and `<selected-effort>`. Reassess them in every round after the plan changes.

4. **Run Codex review**: Execute the same review flow as `/codex:review-plan`:
   - Load the prompt template at `${CLAUDE_PLUGIN_ROOT}/prompts/review-plan.md` using `Read`.
   - Replace `{{PLAN_PATH}}` with the absolute path to the plan file.
   - Run the Codex review in the foreground:
   ```bash
   node -e "require('fs').writeFileSync('/tmp/codex-refine-plan-$$.md', process.argv[1])" '<the prompt string>' && node "${CLAUDE_PLUGIN_ROOT}/scripts/codex-companion.mjs" task --model <selected-model> --effort <selected-effort> --prompt-file /tmp/codex-refine-plan-$$.md; rm -f /tmp/codex-refine-plan-$$.md
   ```
   - Display the Codex review output verbatim.

5. **Extract Verdict**:
   - Look for the `<!-- VERDICT: ... -->` HTML comment marker in the output.
   - If the marker is not found, search the output text for `**READY**`, `**NEEDS_IMPROVEMENT**`, or `**MAJOR_REVISION**` as a fallback.
   - If the verdict cannot be determined, treat it as NEEDS_IMPROVEMENT and continue.

6. **Evaluate Verdict**:
   - **READY**: Display a completion message and stop the loop. The plan is approved.
   - **NEEDS_IMPROVEMENT** or **MAJOR_REVISION**: Continue to step 7.

7. **Fix the plan**:
   - Read the current plan file using `Read`.
   - For each issue in the Issues section of the review output, apply the Suggestion using the `Edit` tool on the plan file.
   - Only modify the plan file identified in Step 1. Do not edit any other file.
   - After all edits, display a brief bulleted list of changes made.
   - Continue to the next round.

Step 4: Loop exhaustion

If all 5 rounds are completed without reaching READY:
- Display the final review result from the last round.
- Inform the user that the maximum number of refinement rounds has been reached and manual review may be needed.

Important: When embedding the prompt string in the `node -e` command, escape any single quotes in the prompt content by replacing `'` with `'\''` to prevent shell interpretation issues.

Argument handling:
- Preserve the user's `--resume` flag. Do not strip it or add extra flags.
- The first positional argument (if present and not a flag) is the plan file path.

Recovery flow (--resume)

Detect an unapplied review result from a previous session's interrupted loop and continue refining. Never launch a new Codex review during recovery itself.

1. Find the finished review job from the previous session:
   - Run:
   ```bash
   node "${CLAUDE_PLUGIN_ROOT}/scripts/codex-companion.mjs" status --all
   ```
   - `--all` is required: the default status output only lists jobs from the current session, and the interrupted round's job belongs to the previous one.
   - Pick the most recent completed plan-review task job. If no finished job exists, tell the user no recoverable review was found and suggest running `/codex:refine-plan` normally. Stop.

2. Retrieve the stored review output:
   - Run:
   ```bash
   node "${CLAUDE_PLUGIN_ROOT}/scripts/codex-companion.mjs" result <job-id>
   ```
   - Display the output verbatim before any modifications (same rule as the normal loop).

3. Determine where the loop was interrupted:
   - Extract the Verdict using the same rules as Step 3.5. If the verdict cannot be determined, treat it as NEEDS_IMPROVEMENT and continue.
   - If READY, no recovery is needed; the plan was already approved. Stop.
   - Identify the plan file using the Step 1 priority order. The review output usually references the original absolute plan path from the review prompt; prefer that path when present. If `--resume` was given an explicit plan file path, use it.
   - Compare the plan file's last-modified time with the job's completion time. If the plan was modified after the review completed, some fixes from that review may already be applied: verify each issue against the current plan content and skip any that are no longer present.

4. Resume the loop:
   - Apply any remaining Suggestions from the recovered review with the `Edit` tool on the plan file only, and report the changes as a brief bulleted list (same as Step 3.7).
   - Then continue with Step 3's refinement loop for the remaining rounds. The recovered review counts as one round: if every issue was already applied (i.e., the previous session completed the fix step before interruption), start the next review round immediately and count it as the following round; if fixes were applied in this session, the round of the recovered review is consumed.

Recovery limitations:
- Recovery depends on the companion job state for this workspace, which is stored outside the repository and capped at the most recent 50 jobs. Very old reviews may no longer be recoverable; in that case rerun `/codex:refine-plan` normally.
- The recovered round number is not recorded in the review output, so the loop conservatively counts the recovered review as one full round when resuming. This can reduce the remaining refinement budget by one compared with an uninterrupted run.
- If the Codex session ID is present in the result output, the user can also inspect the original review interactively with `codex resume <session-id>`.
