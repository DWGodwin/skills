---
name: prime
description: Prime agent with codebase understanding
---

# Prime: Load Project Context

1. Read CLAUDE.md and any README files
2. Run `git ls-files` and `tree -L 3 -I 'node_modules|__pycache__|.git|dist|build'`
3. Identify and read main entry points, config files, and key modules
4. Run `git log -10 --oneline` and `git status`

## Output

Concise bullet-point summary:
- **Purpose** — what the project does
- **Architecture** — structure, key directories, patterns
- **Tech stack** — languages, frameworks, tools
- **Current state** — active branch, recent focus
