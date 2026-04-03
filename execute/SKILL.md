---
description: Execute an implementation plan
argument-hint: [path-to-plan]
---

# Execute: $ARGUMENTS

1. Read the entire plan
2. Execute tasks in order — read referenced files before modifying, run each task's validation command, fix before moving on
3. Run all validation commands from the plan
4. If deviating from the plan, explain why

Report: files created/modified, validation results, any deviations, ready for `/commit`.
