# Codex Prompt Recipes

Use these as starting templates for Codex task prompts or other Codex/GPT-5.6 prompt construction.
Copy the smallest recipe that fits the task, then trim anything you do not need.
In `codex:codex-rescue`, run diagnosis and fix-oriented recipes in write mode by default unless the user explicitly asked for read-only behavior.

## Diagnosis

```xml
<task>
Diagnose why the failing test or command is breaking in this repository.
Use the available repository context and tools to identify the most likely root cause.
</task>

<output_contract>
Return the most likely root cause, supporting evidence, and the smallest safe next step.
</output_contract>

<verification>
Before finalizing, verify that the proposed root cause matches the observed evidence.
</verification>

<grounding>
Inspect missing repository facts rather than guessing. State what remains unknown if evidence is unavailable.
</grounding>
```

## Narrow Fix

```xml
<task>
Implement the smallest safe fix for the identified issue in this repository.
Preserve existing behavior outside the failing path.
</task>

<autonomy>
Make in-scope local edits and run non-destructive validation. Ask before destructive actions, external writes, or material scope expansion.
</autonomy>

<output_contract>
Return the fix summary, touched files, verification, and any material residual risk.
</output_contract>

<verification>
Before finalizing, verify that the fix matches the task requirements and that the changed code is coherent.
</verification>
```

## Root-Cause Review

```xml
<task>
Analyze this change for the most likely correctness or regression issues.
Focus on the provided repository context only.
</task>

<output_contract>
Return material findings ordered by severity, supporting evidence, and brief next steps.
</output_contract>

<grounding>
Ground every finding in repository context or tool output. Label inferences and inspect relevant second-order failure paths when warranted.
</grounding>

<verification>
Before finalizing, verify that each finding is material and actionable.
</verification>
```

## Research Or Recommendation

```xml
<task>
Research the available options and recommend the best path for this task.
</task>

<output_contract>
Return observed facts, the recommendation, material tradeoffs, and open questions.
</output_contract>

<grounding>
Separate facts from inferences. Cite material claims with the primary sources inspected.
</grounding>
```

## Prompt-Patching

```xml
<task>
Diagnose why this existing prompt is underperforming and propose the smallest high-leverage changes to improve it for Codex or GPT-5.6.
</task>

<output_contract>
Return the supported failure modes, root causes, revised prompt, and why the revision addresses them.
</output_contract>

<grounding>
Base your diagnosis on the prompt text and the failure examples provided.
Do not invent failure modes that are not supported by the examples.
</grounding>

<verification>
Before finalizing, make sure the revised prompt resolves the cited failure modes without adding contradictory instructions.
</verification>
```
