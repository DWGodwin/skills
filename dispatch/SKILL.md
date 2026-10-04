---
name: dispatch
description: Supervise plan → execute → validate as isolated subagents for one high-confidence task, then hand the diff back for review
argument-hint: [#issue | issue-name] [--light | --full] [task description, if not a GitHub issue]
user_invocable: true
---

# Dispatch: $ARGUMENTS

You are the **supervisor**, not the implementer. Your only jobs: dispatch each stage as an isolated subagent, read its returned summary, gate on success, and hand the finished diff back to the user. Do NOT plan, write code, or run tests yourself — every stage runs in its own fresh subagent context so this conversation stays a thin control loop. Each stage hands off to the next through files on disk, exactly like the manual rewind/clear loop these skills were built for.

**Run me inside a dedicated worktree window**, normally launched by `claude-dispatch new <repo> <issue#> [suffix] [--light|--full]` (see `bin/claude-dispatch`), which creates the worktree at `<repo>/.git/wt/issue-N` on branch `issue-N` and opens a tmux window named `issue-N` in the repo's project session running:

```
claude --remote-control <repo>/issue-N --name issue-N --model sonnet --permission-mode bypassPermissions "/dispatch #N"
```

Isolated worktree (so changes can't collide with other work), cheap supervisor model (this loop is just orchestration), permissions bypassed so the run is unattended while you check back, and Remote Control on so the run can be watched or steered from claude.ai/code or the mobile app. The stages run as the `dispatch-*` agents below, whose definitions pin each stage's model and reasoning effort regardless of the supervisor model.

This is for **high-confidence work**: a clear-cut, single-session task where you trust the loop to run without your judgement at each step. If you need to learn from the change or the requirements are fuzzy, run the manual loop instead.

## Stage agents and tiers

| Stage | **full** tier | **light** tier |
|---|---|---|
| Plan | `dispatch-planner` (fable, high effort) | skipped |
| Execute | `dispatch-executor` (fable, high effort) | `dispatch-executor-light` (sonnet, medium effort) — plans inline |
| Validate | `dispatch-validator` (sonnet, medium effort) | `dispatch-validator` with `model: haiku` |

Spawn each stage with the Agent tool, `subagent_type: <agent name>`. The definitions (`~/.claude/skills/agents/`, symlinked to `~/.claude/agents`) own model and effort — do not pass a `model` override, except the light-tier validate above and, if a spawn reports `fable` is unavailable, re-spawning that stage with `model: opus`.

If the `dispatch-*` agent types are not available in this session, STOP before step 1 and tell the user to run `ln -s ~/.claude/skills/agents ~/.claude/agents` and restart the session.

## 0. Resolve the task, pick the tier, and confirm scope — you cannot ask later

First, get the full task text:

- **If the argument references a GitHub issue** (`#N`, `issue N`, or `owner/repo#N`): fetch it with `gh issue view N --json title,body,comments,labels` — add `--repo owner/repo` only if the argument named a repo, otherwise it resolves from this worktree's remote. The task is title + body **plus the comment thread** — distill any material context from the comments (decisions, added constraints, scope changes, "actually do it this way" — not chatter) into a short **"Context from comments"** section appended to the resolved task text. Comments are often where the real spec lives, and the planning subagent runs in isolation, so anything you leave out here is lost to it. Use `issue-N` as the `{kebab-name}` so artifacts line up with the worktree name.
- **Otherwise** treat the argument as the task description directly and derive a `{kebab-name}` from it.

Then pick the **tier**:

- `--light` or `--full` in the arguments, or a `dispatch:light` / `dispatch:full` label on the issue, decides it (the argument wins over the label).
- Otherwise triage. **light** only if ALL hold: the change is confined to about 1–3 files; the "how" is obvious from the task or an existing pattern (no design decision with real trade-offs); the acceptance criteria are concrete; and it touches no schema, public API, dependency, migration, or security-sensitive code. Anything else — and anything you are unsure about — is **full**.

Subagents run non-interactively and **cannot ask you questions mid-run**, so resolve ambiguity now:

1. Restate the resolved task in 1–2 sentences and its acceptance criteria, plus the tier and a one-line reason for it.
2. If anything material is ambiguous, ask the user **now**. If it's genuinely unambiguous and high-confidence, say so and proceed.

Create `.agents/dispatch/{kebab-name}.md` as a progress log, starting with the tier and why, and append to it after every stage (stage, agent, status, one-line summary). This is what the user reads when they check back on the pane.

## 1. Plan — `dispatch-planner` (full tier only; light skips to step 2)

Spawn `dispatch-planner` with this brief — paste the **full resolved task text** from step 0 (for an issue: title, body, and the "Context from comments" section) so the planner needs no other context:

> Plan this task as `{kebab-name}`:
> <resolved task text>

Record the summary in the log. **Gate:** if the agent reports it could not produce a coherent plan (missing context, contradictory requirements), STOP and report to the user. Otherwise note the plan path so the user can peek if they check in, and continue.

## 2. Execute — fresh `dispatch-executor` (full) or `dispatch-executor-light` (light)

**Full:** spawn `dispatch-executor` — fresh context, sees only the plan file:

> Execute the plan at `.agents/plans/{kebab-name}.md`.

**Light:** spawn `dispatch-executor-light` with the full resolved task text from step 0 as the brief.

Record the summary in the log. **Gate:** if a light executor returns `ESCALATE`, follow **Escalation** below. If execute reports unresolved failures or a blocking deviation, STOP — do not validate. Report what happened and ask the user how to proceed (retry, adjust the plan, or hand off).

## 3. Validate — fresh `dispatch-validator`

Spawn `dispatch-validator` (light tier: pass `model: haiku`):

> Validate the current worktree.

Record the PASS/FAIL result in the log.

## 4. Hand back to the user

- **If validate FAILED on the light tier:** follow **Escalation** below.
- **If validate FAILED on the full tier:** present the failures concisely and ask whether to dispatch one fix iteration (a fresh `dispatch-executor` scoped to just the failures) or stop. Do not loop automatically more than once without checking in.
- **If validate PASSED:** run `git diff` and `git status --short` (for untracked files), present a scannable summary of what changed, then run the `~/.claude/skills/review-understanding/SKILL.md` flow yourself — confirm it matches the user's understanding, ask 3–5 comprehension questions, surface watch-outs — and end with the natural next step (e.g. `/commit` or open a PR). This human checkpoint is the point of the whole loop; never skip it.

## Escalation (light tier only)

A light run that hits trouble moves up a tier instead of fix-iterating on the cheap model. If `dispatch-executor-light` returns `ESCALATE`, or light-tier validate FAILS, switch the run to **full** — once, automatically: log the escalation and its reason, then go to step 1 and append what the light attempt found to the planner's brief (the `ESCALATE` reason; or the failing output plus a note that the worktree holds a light-tier attempt to keep, fix, or replace). From there the full-tier gates apply; never drop back to light.

## Rules

- **Sequential only.** One stage at a time; never start a stage before reading the prior stage's summary and passing its gate.
- **Keep your context thin.** Subagents return summaries, not transcripts. Don't re-read their work unless a gate requires it.
- **One source of truth.** The agents preload the existing stage SKILL.md files; you never reimplement their logic here.
- **The tier is a cost decision, not a quality one.** The review-understanding checkpoint and every gate apply to both tiers.
- **You don't manage fan-out.** To try an alternative approach, the user opens another worktree window and runs `/dispatch` there — `claude-dispatch new <repo> <N> <suffix>` names them `issue-N-<suffix>` so the worktrees and windows don't collide. Each worktree is one independent attempt; comparing and choosing a winner is the user's call.
