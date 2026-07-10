<role>
Perform an implementation-plan review and provide actionable feedback.
</role>

<task>
Read the implementation plan file at the following path and review it thoroughly:
Plan file: {{PLAN_PATH}}

Read the file before evaluating it against the criteria below.
</task>

<supporting_context>
{{SUPPORTING_CONTEXT}}
</supporting_context>

<review_criteria>
Evaluate the plan on these dimensions:

1. **Feasibility**: Is the plan technically executable? Are the proposed approaches realistic given the codebase and constraints?
2. **Completeness**: Are all necessary steps covered? Are there missing steps, edge cases, or integration points that should be addressed?
3. **Risk**: What are the potential failure modes? Are there assumptions that could break under real-world conditions?
4. **Dependencies**: Are prerequisites and ordering constraints correctly identified? Are there implicit dependencies that should be made explicit?
5. **Improvements**: Are there simpler, safer, or more efficient approaches that achieve the same goal?
</review_criteria>

<review_method>
- Cross-reference the codebase to verify paths, symbols, and architectural assumptions.
- Identify material gaps, dependency issues, and verification blind spots.
- Check that the plan scope matches its goal.
</review_method>

<output_format>
Structure your review as:

## Plan Review

### Summary
State readiness and the evidence that matters.

### Issues
For each issue found (max 5, ordered by severity):
- **Issue**: Clear description of the problem
- **Impact**: Why this matters
- **Suggestion**: Concrete improvement

### Strengths
Bullet points of what the plan does well.

### Verdict
One of:
- **READY**: Plan is solid and can be executed as-is
- **NEEDS_IMPROVEMENT**: Plan has issues that should be addressed before execution
- **MAJOR_REVISION**: Plan has fundamental problems that require rethinking

Immediately after the verdict line, emit a machine-readable HTML comment marker on its own line:
`<!-- VERDICT: READY -->`, `<!-- VERDICT: NEEDS_IMPROVEMENT -->`, or `<!-- VERDICT: MAJOR_REVISION -->`

### Recommended Next Steps
Specific actions to take based on the verdict.
</output_format>

<grounding_rules>
Every observation must be traceable to the plan or inspected codebase. Do not invent files, functions, or behavior; label assumptions.
</grounding_rules>
