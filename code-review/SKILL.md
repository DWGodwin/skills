---
description: Technical code review for quality and bugs
---

# Code Review

Run `git diff HEAD` and `git ls-files --others --exclude-standard`. Read each changed/new file in full context.

Review for: logic errors, security issues, performance problems, and violations of codebase patterns (check CLAUDE.md).

Verify issues are real — run tests, confirm type errors, validate with context. No style nitpicks.

Save to `.agents/code-reviews/[name].md`:

```
severity: critical|high|medium|low
file: path/to/file
line: 42
issue: one-line description
suggestion: how to fix
```

If clean: "Code review passed."
