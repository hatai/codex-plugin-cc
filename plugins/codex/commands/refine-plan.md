---
description: Repeatedly review and refine a Claude Code plan via Codex until no critical issues remain (max 5 rounds)
argument-hint: '[plan-file-path]'
allowed-tools: Read, Glob, Edit, Bash(node:*), AskUserQuestion
---

Iteratively review and refine a Claude Code implementation plan by running Codex reviews and applying fixes until no critical issues remain, or the maximum number of rounds is reached.

Raw slash-command arguments:
`$ARGUMENTS`

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

2. **Run Codex review**: Execute the same review flow as `/codex:review-plan`:
   - Load the prompt template at `${CLAUDE_PLUGIN_ROOT}/prompts/review-plan.md` using `Read`.
   - Replace `{{PLAN_PATH}}` with the absolute path to the plan file.
   - Run the Codex review in the foreground:
   ```bash
   node -e "require('fs').writeFileSync('/tmp/codex-refine-plan-$$.md', process.argv[1])" '<the prompt string>' && node "${CLAUDE_PLUGIN_ROOT}/scripts/codex-companion.mjs" task --prompt-file /tmp/codex-refine-plan-$$.md; rm -f /tmp/codex-refine-plan-$$.md
   ```
   - Display the Codex review output verbatim.

3. **Extract Verdict**:
   - Look for the `<!-- VERDICT: ... -->` HTML comment marker in the output.
   - If the marker is not found, search the output text for `**READY**`, `**NEEDS_IMPROVEMENT**`, or `**MAJOR_REVISION**` as a fallback.
   - If the verdict cannot be determined, treat it as NEEDS_IMPROVEMENT and continue.

4. **Evaluate Verdict**:
   - **READY**: Display a completion message and stop the loop. The plan is approved.
   - **NEEDS_IMPROVEMENT** or **MAJOR_REVISION**: Continue to step 5.

5. **Fix the plan**:
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
