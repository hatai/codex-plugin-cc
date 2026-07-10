# Codex Prompt Anti-Patterns

Avoid these when prompting Codex or GPT-5.6.

## Vague task framing

Bad:

```text
Take a look at this and let me know what you think.
```

Better:

```xml
<task>
Review this change for material correctness and regression risks.
</task>
```

## Missing output contract

Bad:

```text
Investigate and report back.
```

Better:

```xml
<output_contract>
Return the root cause, evidence, and smallest safe next step.
</output_contract>
```

## No follow-through default

Bad:

```text
Debug this failure.
```

Better:

```xml
<autonomy>
Inspect relevant material and continue with safe in-scope actions. Ask only when a missing detail changes correctness, safety, or reversibility.
</autonomy>
```

## Asking for more reasoning instead of a better contract

Bad:

```text
Think harder and be very smart.
```

Better:

```xml
<verification>
Before finalizing, verify that the answer matches the observed evidence and task requirements.
</verification>
```

## Mixing unrelated jobs into one run

Bad:

```text
Review this diff, fix the bug you find, update the docs, and suggest a roadmap.
```

Better:
- Run review first.
- Run a separate fix prompt if needed.
- Use a third run for docs or roadmap work.

## Unsupported certainty

Bad:

```text
Tell me exactly why production failed.
```

Better:

```xml
<grounding>
Ground every claim in the provided context or tool outputs, and label inferences.
</grounding>
```
