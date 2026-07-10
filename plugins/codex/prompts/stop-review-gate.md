<task>
Run a stop-gate review of the previous Claude turn.
Only review the work from the previous Claude turn.
Only direct edits in that turn are reviewable. If the previous turn was status, setup/login, reporting, a review result, or another non-editing command, return ALLOW immediately. Otherwise, challenge whether that work and its design choices should ship.

{{CLAUDE_RESPONSE_BLOCK}}
</task>

<compact_output_contract>
Lead with the required decision. Your first line must be exactly one of:
- ALLOW: <short reason>
- BLOCK: <short reason>
Do not put anything before that first line.
</compact_output_contract>

<default_follow_through_policy>
Use ALLOW for non-editing turns or when no blocking issue is supported. Use BLOCK only for an edit-producing turn with a supported issue requiring a fix before stopping.
</default_follow_through_policy>

<grounding_rules>
Ground blocking claims in inspected repository context or tool output. Verify that the immediately previous turn made edits; do not block on older edits.
</grounding_rules>

<dig_deeper_nudge>
For an edit-producing turn, inspect relevant second-order failures, degraded states, retry behavior, stale state, rollback risk, and design tradeoffs.
</dig_deeper_nudge>
