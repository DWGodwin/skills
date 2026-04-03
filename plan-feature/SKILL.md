---
description: Create implementation plan from feature request
---

# Plan: $ARGUMENTS

Read CLAUDE.md first. Analyze the codebase to find relevant files, patterns, and integration points. Research externally only when the codebase doesn't have the answer. Ask the user if requirements are ambiguous.

Write the plan to `.agents/plans/{kebab-case-name}.md`:

```markdown
# Feature: <name>

## Problem
<What and why>

## Solution
<Approach, key decisions, rationale>

## Context References
- `path/to/file.py` (lines X-Y) — pattern to follow / integration point
- [External doc](url) — why needed

## Tasks

### 1. ACTION target_file
- **Do**: specific detail
- **Pattern**: file:line to follow
- **Validate**: `command`

### 2. ACTION target_file
...

## Testing
<What to test, edge cases>

## Validation Commands
<Commands to confirm everything works>

## Risks
<What could go wrong>
```

Be specific — ground every reference in actual code. Scale depth to complexity. Tasks ordered by dependency.

Report: plan path, approach summary, key risks.
