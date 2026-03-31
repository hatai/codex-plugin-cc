---
description: Send a Claude Code plan to Codex for review and return feedback
argument-hint: '[--wait|--background] [plan-file-path]'
disable-model-invocation: true
allowed-tools: Read, Glob, Bash(node:*), AskUserQuestion
---

Send a Claude Code implementation plan to Codex for review and return the feedback verbatim.

Raw slash-command arguments:
`$ARGUMENTS`

Core constraint:
- This command is review-only.
- Do not fix issues, apply patches, or suggest that you are about to make changes.
- Your only job is to run the review and return Codex's output verbatim to the user.

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
- The plan content is NOT embedded in the prompt. Codex will read the file itself.
- Store the final prompt string for use in Step 5.

Step 4: Determine execution mode

- If the raw arguments include `--wait`, do not ask. Run in the foreground.
- If the raw arguments include `--background`, do not ask. Run in a Claude background task.
- Otherwise, since plan reviews are typically small, recommend waiting.
- Use `AskUserQuestion` exactly once with two options, putting the recommended option first and suffixing its label with `(Recommended)`:
  - `Wait for results`
  - `Run in background`

Step 5: Execute Codex review

Use a node one-liner to write the interpolated prompt to a temporary file, then invoke `codex-companion.mjs task --prompt-file` in a single chained command.
This ensures the entire command starts with `node`, matching the `Bash(node:*)` allowed-tools pattern.

Foreground flow:
- Run a single chained command:
```bash
node -e "require('fs').writeFileSync('/tmp/codex-review-plan-$$.md', process.argv[1])" '<the prompt string from Step 3>' && node "${CLAUDE_PLUGIN_ROOT}/scripts/codex-companion.mjs" task --prompt-file /tmp/codex-review-plan-$$.md; rm -f /tmp/codex-review-plan-$$.md
```
- The first `node -e` writes the prompt to a temp file.
- The second `node` runs the Codex task, reading the prompt from that file.
- `rm -f` cleans up regardless of task exit status (uses `;` not `&&`).
- Return the command stdout verbatim, exactly as-is.
- Do not paraphrase, summarize, or add commentary before or after it.
- Do not fix any issues mentioned in the review output.

Background flow:
- Launch the review with `Bash` in the background using the same chained command pattern:
```typescript
Bash({
  command: `node -e "require('fs').writeFileSync('/tmp/codex-review-plan-$$.md', process.argv[1])" '<the prompt string from Step 3>' && node "${CLAUDE_PLUGIN_ROOT}/scripts/codex-companion.mjs" task --prompt-file /tmp/codex-review-plan-$$.md; rm -f /tmp/codex-review-plan-$$.md`,
  description: "Codex plan review",
  run_in_background: true
})
```
- Do not call `BashOutput` or wait for completion in this turn.
- After launching the command, tell the user: "Codex plan review started in the background. Check `/codex:status` for progress."

Important: When embedding the prompt string in the `node -e` command, escape any single quotes in the prompt content by replacing `'` with `'\''` to prevent shell interpretation issues.

Argument handling:
- Preserve the user's `--wait` and `--background` flags.
- Do not strip them yourself.
- Do not add extra flags or rewrite the user's intent.
- The first positional argument (if present and not a flag) is the plan file path.
