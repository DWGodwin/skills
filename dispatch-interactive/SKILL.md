---
name: dispatch-interactive
description: Supervise plan → execute → validate as isolated subagents, pausing for the user's input at every stage (approve / edit / redo) before moving on
argument-hint: [#issue | issue-name] [task description, if not a GitHub issue]
user_invocable: true
---

# Dispatch (interactive): $ARGUMENTS

You are the **supervisor**, not the implementer. Same loop as `/dispatch` — dispatch each stage as an isolated subagent, hand off through files on disk, keep your own context thin — **except you stop after every stage and wait for the user's input before continuing.** Do NOT plan, write code, or run tests yourself.

The subagents are non-interactive and cannot ask questions mid-run. That is fine: **all of the user's judgement happens here, at the stage boundaries between dispatches.** Where autonomous `/dispatch` gates only on failure, you gate after *every* stage — success included — present the artifact, and thread the user's feedback into the next dispatch (or a re-dispatch of the same stage).

**Run me inside a dedicated worktree session**, normally launched by `claude-dispatch-i <repo> <issue#> [suffix]`. Isolated worktree (changes can't collide), cheap supervisor model (this loop is just orchestration), permissions bypassed so a stage's subagent never blocks on a per-edit prompt — the user reviews at stage boundaries, not per edit. The heavy stages spawn on **fable** when available (falling back to **opus** if the Task tool rejects `fable`) regardless of the supervisor model.

Use this variant when you want a hand on the wheel — fuzzy requirements, a plan you want to shape, or a change you want to learn from. For clear-cut high-confidence work that can run unattended, use autonomous `/dispatch` instead.

## 0. Resolve the task and confirm scope

Get the full task text:

- **If the argument references a GitHub issue** (`#N`, `issue N`, or `owner/repo#N`): fetch it with `gh issue view N --json title,body,comments` — add `--repo owner/repo` only if the argument named a repo, otherwise it resolves from this worktree's remote. The task is title + body **plus the comment thread** — distill any material context from the comments (decisions, added constraints, scope changes, "actually do it this way" — not chatter) into a short **"Context from comments"** section appended to the resolved task text. Comments are often where the real spec lives, and the planning subagent runs in isolation, so anything you leave out here is lost to it. Use `issue-N` as the `{kebab-name}` so artifacts line up with the worktree.
- **Otherwise** treat the argument as the task description directly and derive a `{kebab-name}`.

Restate the resolved task in 1–2 sentences with its acceptance criteria, and **ask the user to confirm scope before you dispatch anything.** Unlike autonomous dispatch, you can — and should — surface even minor ambiguities here; the user is present.

Create `.agents/dispatch/{kebab-name}.md` as a progress log and append after every stage (stage, status, one-line summary, and the user's decision at that gate).

## 1. Plan — subagent, model: fable (fallback: opus)

Launch a general-purpose subagent (Task tool, `model: fable`; if `fable` is not an accepted model value in this session, use `model: opus`) — paste the **full resolved task text** from step 0 (title, body, and the "Context from comments" section) so the planner needs no other context:

> Read and follow `~/.claude/skills/plan-feature/SKILL.md` to plan this task:
> <resolved task text>
>
> The repo is the current worktree. Do NOT implement anything. You cannot ask questions — make reasonable assumptions and record each under the plan's Risks. Return ONLY: the plan file path, a 3-line approach summary, and the top risks.

**Gate — always stop here.** Present the approach summary and top risks, and point the user at the plan file so they can open it. Then ask which way to go:

- **approve** → continue to Execute.
- **edit** → the user gives changes; either apply small edits to the plan file yourself, or re-dispatch a fresh planning subagent with their feedback appended to the brief. Re-present and ask again.
- **stop** → record and end.

Do not proceed until the user approves.

## 2. Execute — fresh subagent, model: fable (fallback: opus)

Only after plan approval. Launch a new general-purpose subagent (`model: fable`, or `model: opus` if fable is not available) — fresh context, sees only the plan file:

> Read and follow `~/.claude/skills/execute/SKILL.md` for the plan at `.agents/plans/{kebab-name}.md`. Read the plan and every file it references before editing. Run each task's validation command and fix before moving on. Return ONLY: files created/modified, per-task validation results, and any deviations from the plan with reasons.

**Gate — always stop here.** Run `git diff` and `git status --short` (for untracked files) and present a scannable summary of what changed, plus any deviations the agent reported. Then ask:

- **approve** → continue to Validate.
- **request changes** → the user describes what to fix; dispatch a fresh execute subagent scoped to just those changes (give it the diff context and the feedback). Re-present the diff and ask again.
- **abort** → record and end (the worktree keeps the work for inspection).

Do not proceed until the user approves the diff.

## 3. Validate — fresh subagent, model: sonnet

Launch a new general-purpose subagent (`model: sonnet`):

> Read and follow `~/.claude/skills/validate/SKILL.md`. Discover the lint, type-check/build, and test commands from CLAUDE.md and project config, run them, and return a PASS/FAIL summary per category plus the failing output for anything that fails.

Record the PASS/FAIL result in the log and present it.

- **If it FAILED:** show the failures and ask whether to dispatch a fix iteration (a fresh execute subagent scoped to just the failures), edit the plan, or stop. Loop back through Execute → Validate as the user directs.
- **If it PASSED:** ask the user to confirm they're ready for the final review, then continue.

## 4. Hand back to the user

Run the `~/.claude/skills/review-understanding/SKILL.md` flow yourself — confirm the change matches the user's understanding, ask 3–5 comprehension questions, surface watch-outs — and end with the natural next step (e.g. `/commit` or open a PR). This human checkpoint is the point of the whole loop; never skip it.

## Rules

- **Stop at every gate.** Steps 0, 1, 2, and 3 each end by waiting for the user. Never carry one stage's output into the next without an explicit go.
- **Sequential only.** One stage at a time; read the prior stage's summary before the next dispatch.
- **Keep your context thin.** Subagents return summaries, not transcripts. Don't re-read their work unless a gate (or the user's feedback) requires it.
- **One source of truth.** Stages read the existing SKILL.md files; you never reimplement their logic here.
- **Thread feedback, don't override.** When the user asks for changes, fold their words into a fresh subagent brief rather than editing code yourself.
- **You don't manage fan-out.** To try an alternative approach, the user opens another worktree session — `claude-dispatch-i <repo> <N> <suffix>` names them `issue-N-<suffix>` so worktrees don't collide. Comparing attempts is the user's call.
