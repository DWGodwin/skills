---
name: dispatch
description: Supervise plan → execute → validate as isolated subagents for one high-confidence task, then hand the diff back for review
argument-hint: [#issue | issue-name] [task description, if not a GitHub issue]
user_invocable: true
---

# Dispatch: $ARGUMENTS

You are the **supervisor**, not the implementer. Your only jobs: dispatch each stage as an isolated subagent, read its returned summary, gate on success, and hand the finished diff back to the user. Do NOT plan, write code, or run tests yourself — every stage runs in its own fresh subagent context so this conversation stays a thin control loop. Each stage hands off to the next through files on disk, exactly like the manual rewind/clear loop these skills were built for.

**Run me inside a dedicated worktree session**, normally launched by `claude-dispatch <repo> <issue#> [suffix]` (see `bin/claude-dispatch`), which creates the worktree and a named tmux session (`disp-issue-N`) running:

```
claude --worktree issue-N --model sonnet --permission-mode bypassPermissions "/dispatch #N"
```

Isolated worktree (so changes can't collide with other work), cheap supervisor model (this loop is just orchestration), permissions bypassed so the run is unattended while you check back. The heavy stages spawn on **fable** when available (falling back to **opus** if the Task tool rejects `fable`) regardless of the supervisor model — see below.

This is for **high-confidence work**: a clear-cut, single-session task where you trust the loop to run without your judgement at each step. If you need to learn from the change or the requirements are fuzzy, run the manual loop instead.

## 0. Resolve the task and confirm scope — you cannot ask later

First, get the full task text:

- **If the argument references a GitHub issue** (`#N`, `issue N`, or `owner/repo#N`): fetch it with `gh issue view N --json title,body,comments` — add `--repo owner/repo` only if the argument named a repo, otherwise it resolves from this worktree's remote. The task is title + body **plus the comment thread** — distill any material context from the comments (decisions, added constraints, scope changes, "actually do it this way" — not chatter) into a short **"Context from comments"** section appended to the resolved task text. Comments are often where the real spec lives, and the planning subagent runs in isolation, so anything you leave out here is lost to it. Use `issue-N` as the `{kebab-name}` so artifacts line up with the worktree name.
- **Otherwise** treat the argument as the task description directly and derive a `{kebab-name}` from it.

Subagents run non-interactively and **cannot ask you questions mid-run**, so resolve ambiguity now:

1. Restate the resolved task in 1–2 sentences and its acceptance criteria.
2. If anything material is ambiguous, ask the user **now**. If it's genuinely unambiguous and high-confidence, say so and proceed.

Create `.agents/dispatch/{kebab-name}.md` as a progress log and append to it after every stage (stage, status, one-line summary). This is what the user reads when they check back on the pane.

## 1. Plan — subagent, model: fable (fallback: opus)

Launch a general-purpose subagent (Task tool, `model: fable`; if `fable` is not an accepted model value in this session, use `model: opus`) with this brief — paste the **full resolved task text** from step 0 (for an issue: title, body, and the "Context from comments" section) so the planner needs no other context:

> Read and follow `~/.claude/skills/plan-feature/SKILL.md` to plan this task:
> <resolved task text>
>
> The repo is the current worktree. Do NOT implement anything. You cannot ask questions — make reasonable assumptions and record each one under the plan's Risks. Return ONLY: the plan file path, a 3-line approach summary, and the top risks.

Record the summary in the log. **Gate:** if the agent reports it could not produce a coherent plan (missing context, contradictory requirements), STOP and report to the user. Otherwise note the plan path so the user can peek if they check in, and continue.

## 2. Execute — fresh subagent, model: fable (fallback: opus)

Launch a new general-purpose subagent (`model: fable`, or `model: opus` if fable is not available) — fresh context, sees only the plan file:

> Read and follow `~/.claude/skills/execute/SKILL.md` for the plan at `.agents/plans/{kebab-name}.md`. Read the plan and every file it references before editing. Run each task's validation command and fix before moving on. Return ONLY: files created/modified, per-task validation results, and any deviations from the plan with reasons.

Record the summary in the log. **Gate:** if execute reports unresolved failures or a blocking deviation, STOP — do not validate. Report what happened and ask the user how to proceed (retry, adjust the plan, or hand off).

## 3. Validate — fresh subagent, model: sonnet

Launch a new general-purpose subagent (`model: sonnet`):

> Read and follow `~/.claude/skills/validate/SKILL.md`. Discover the lint, type-check/build, and test commands from CLAUDE.md and project config, run them, and return a PASS/FAIL summary per category plus the failing output for anything that fails.

Record the PASS/FAIL result in the log.

## 4. Hand back to the user

- **If validate FAILED:** present the failures concisely and ask whether to dispatch one fix iteration (a fresh execute subagent scoped to just the failures) or stop. Do not loop automatically more than once without checking in.
- **If validate PASSED:** run `git diff` and `git status --short` (for untracked files), present a scannable summary of what changed, then run the `~/.claude/skills/review-understanding/SKILL.md` flow yourself — confirm it matches the user's understanding, ask 3–5 comprehension questions, surface watch-outs — and end with the natural next step (e.g. `/commit` or open a PR). This human checkpoint is the point of the whole loop; never skip it.

## Rules

- **Sequential only.** One stage at a time; never start a stage before reading the prior stage's summary and passing its gate.
- **Keep your context thin.** Subagents return summaries, not transcripts. Don't re-read their work unless a gate requires it.
- **One source of truth.** Stages read the existing SKILL.md files; you never reimplement their logic here.
- **You don't manage fan-out.** To try an alternative approach, the user opens another worktree session and runs `/dispatch` there — `claude-dispatch <repo> <N> <suffix>` names them `issue-N-<suffix>` so the worktrees don't collide. Each worktree is one independent attempt; comparing and choosing a winner is the user's call.
