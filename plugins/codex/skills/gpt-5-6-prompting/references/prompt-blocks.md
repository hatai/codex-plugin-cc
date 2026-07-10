# Prompt Blocks

Use these blocks selectively when composing Codex or GPT-5.6 prompts.
Wrap each block in the XML tag shown in its heading.

## Core Wrapper

### `task`

Use in nearly every prompt.

```xml
<task>
Describe the concrete job, the relevant repository or failure context, and the expected end state.
</task>
```

## Output and Format

### `output_contract`

Use when the response shape matters.

```xml
<output_contract>
Lead with the conclusion. Include the evidence, material caveats, and next action required for this task.
</output_contract>
```

### `autonomy`

Use when the task may make local changes or otherwise needs a clear approval boundary.

```xml
<autonomy>
Inspect relevant materials, make in-scope local changes, and run non-destructive validation. Ask before external writes, destructive actions, purchases, or a material scope expansion.
</autonomy>
```

## Follow-through and Completion

### `verification`

Use when the result needs a task-specific check before finalizing.

```xml
<verification>
Before finalizing, check the result against the task requirements and observed repository or tool evidence.
</verification>
```

### `grounding`

Use for review, research, diagnosis, or any claim that must stay evidence-based.

```xml
<grounding>
Base claims on the provided context or inspected tool output. Label any inference.
</grounding>
```

## Grounding and Missing Context

### `citation_rules`

Use when external research or quotes matter.

```xml
<citation_rules>
Back important claims with citations or explicit references to the source material you inspected.
Prefer primary sources.
</citation_rules>
```

## Safety and Scope

### `progress_updates`

Use when the run may take a while.

```xml
<progress_updates>
If you provide progress updates, keep them brief and outcome-based.
Mention only major phase changes or blockers.
</progress_updates>
```
