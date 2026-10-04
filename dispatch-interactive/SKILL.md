---
name: dispatch-interactive
description: Supervise plan → execute → validate as isolated subagents, pausing for the user's input at every stage (approve / edit / redo) before moving on
argument-hint: [#issue | issue-name] [--light | --full] [task description, if not a GitHub issue]
user_invocable: true
---

# Dispatch (interactive): $ARGUMENTS

You are the **supervisor**, not the implementer. Same loop as `/dispatch` — dispatch each stage as an isolated subagent, hand off through files on disk, keep your own context thin — **except you stop after every stage and wait for the user's input before continuing.** Do NOT plan, write code, or run tests yourself.

The subagents are non-interactive and cannot ask questions mid-run. That is fine: **all of the user's judgement happens here, at the stage boundaries between dispatches.** Where autonomous `/dispatch` gates only on failure, you gate after *every* stage — success included — present the artifact, and thread the user's feedback into the next dispatch (or a re-dispatch of the same stage).

**Run me inside a dedicated worktree window**, normally launched by `claude-dispatch new <repo> <issue#> [suffix] -i [--light|--full]`, which creates the worktree at `<repo>/.git/wt/issue-N` and opens a tmux window named `issue-N` in the repo's project session with Remote Control on. Isolated worktree (changes can't collide), cheap supervisor model (this loop is just orchestration), permissions bypassed so a stage's subagent never blocks on a per-edit prompt — the user reviews at stage boundaries, not per edit. The stages run as the `dispatch-*` agents below, whose definitions pin each stage's model and reasoning effort regardless of the supervisor model.

Use this variant when you want a hand on the wheel — fuzzy requirements, a plan you want to shape, or a change you want to learn from. For clear-cut high-confidence work that can run unattended, use autonomous `/dispatch` instead.

## Stage agents and tiers

| Stage | **full** tier | **light** tier |
|---|---|---|
| Plan | `dispatch-planner` (fable, high effort) | skipped |
| Execute | `dispatch-executor` (fable, high effort) | `dispatch-executor-light` (sonnet, medium effort) — plans inline |
| Validate | `dispatch-validator` (sonnet, medium effort) | `dispatch-validator` with `model: haiku` |

Spawn each stage with the Agent tool, `subagent_type: <agent name>`. The definitions (`~/.claude/skills/agents/`, symlinked to `~/.claude/agents`) own model and effort — do not pass a `model` override, except the light-tier validate above and, if a spawn reports `fable` is unavailable, re-spawning that stage with `model: opus`.

If the `dispatch-*` agent types are not available in this session, STOP before step 1 and tell the user to run `ln -s ~/.claude/skills/agents ~/.claude/agents` and restart the session.

## 0. Resolve the task, pick the tier, and confirm scope

Get the full task text:

- **If the argument references a GitHub issue** (`#N`, `issue N`, or `owner/repo#N`): fetch it with `gh issue view N --json title,body,comments,labels` — add `--repo owner/repo` only if the argument named a repo, otherwise it resolves from this worktree's remote. The task is title + body **plus the comment thread** — distill any material context from the comments (decisions, added constraints, scope changes, "actually do it this way" — not chatter) into a short **"Context from comments"** section appended to the resolved task text. Comments are often where the real spec lives, and the planning subagent runs in isolation, so anything you leave out here is lost to it. Use `issue-N` as the `{kebab-name}` so artifacts line up with the worktree.
- **Otherwise** treat the argument as the task description directly and derive a `{kebab-name}`.

Then propose the **tier**:

- `--light` or `--full` in the arguments, or a `dispatch:light` / `dispatch:full` label on the issue, decides it (the argument wins over the label).
- Otherwise triage. **light** only if ALL hold: the change is confined to about 1–3 files; the "how" is obvious from the task or an existing pattern (no design decision with real trade-offs); the acceptance criteria are concrete; and it touches no schema, public API, dependency, migration, or security-sensitive code. Anything else — and anything you are unsure about — is **full**.

Restate the resolved task in 1–2 sentences with its acceptance criteria, plus the tier and a one-line reason for it, and **ask the user to confirm scope and tier before you dispatch anything.** Unlike autonomous dispatch, you can — and should — surface even minor ambiguities here; the user is present.

Create `.agents/dispatch/{kebab-name}.md` as a progress log, starting with the tier and why, and append after every stage (stage, agent, status, one-line summary, and the user's decision at that gate).

## 1. Plan — `dispatch-planner` (full tier only; light skips to step 2)

Spawn `dispatch-planner` — paste the **full resolved task text** from step 0 (title, body, and the "Context from comments" section) so the planner needs no other context:

> Plan this task as `{kebab-name}`:
> <resolved task text>

**Gate — always stop here.** Present the approach summary and top risks, and point the user at the plan file so they can open it. Then ask which way to go:

- **approve** → continue to Execute.
- **edit** → the user gives changes; either apply small edits to the plan file yourself, or re-dispatch a fresh `dispatch-planner` with their feedback appended to the brief. Re-present and ask again.
- **stop** → record and end.

Do not proceed until the user approves.

## 2. Execute — fresh `dispatch-executor` (full) or `dispatch-executor-light` (light)

**Full:** only after plan approval. Spawn `dispatch-executor` — fresh context, sees only the plan file:

> Execute the plan at `.agents/plans/{kebab-name}.md`.

**Light:** spawn `dispatch-executor-light` with the full resolved task text from step 0 as the brief. If it returns `ESCALATE`, follow **Escalation** below instead of the gate.

**Gate — always stop here.** Run `git diff` and `git status --short` (for untracked files) and present a scannable summary of what changed, plus any deviations the agent reported. Then ask:

- **approve** → continue to Validate.
- **request changes** → the user describes what to fix; dispatch a fresh executor of the same tier scoped to just those changes (give it the diff context and the feedback). Re-present the diff and ask again.
- **abort** → record and end (the worktree keeps the work for inspection).

Do not proceed until the user approves the diff.

## 3. Validate — fresh `dispatch-validator`

Spawn `dispatch-validator` (light tier: pass `model: haiku`):

> Validate the current worktree.

Record the PASS/FAIL result in the log and present it.

- **If it FAILED on the light tier:** follow **Escalation** below.
- **If it FAILED on the full tier:** show the failures and ask whether to dispatch a fix iteration (a fresh `dispatch-executor` scoped to just the failures), edit the plan, or stop. Loop back through Execute → Validate as the user directs.
- **If it PASSED:** ask the user to confirm they're ready for the final review, then continue.

## 4. Hand back to the user

Run the `~/.claude/skills/review-understanding/SKILL.md` flow yourself — confirm the change matches the user's understanding, ask 3–5 comprehension questions, surface watch-outs — and end with the natural next step (e.g. `/commit` or open a PR). This human checkpoint is the point of the whole loop; never skip it.

## Escalation (light tier only)

A light run that hits trouble should move up a tier rather than fix-iterate on the cheap model. If `dispatch-executor-light` returns `ESCALATE`, or light-tier validate FAILS, present what happened and ask: **escalate to full** (recommended), retry on light with the user's guidance, or stop. On escalate, log it with the reason, then go to step 1 and append what the light attempt found to the planner's brief (the `ESCALATE` reason; or the failing output plus a note that the worktree holds a light-tier attempt to keep, fix, or replace). From there the full-tier gates apply.

## Rules

- **Stop at every gate.** Steps 0, 1, 2, and 3 each end by waiting for the user. Never carry one stage's output into the next without an explicit go.
- **Sequential only.** One stage at a time; read the prior stage's summary before the next dispatch.
- **Keep your context thin.** Subagents return summaries, not transcripts. Don't re-read their work unless a gate (or the user's feedback) requires it.
- **One source of truth.** The agents preload the existing stage SKILL.md files; you never reimplement their logic here.
- **The tier is a cost decision, not a quality one.** The review-understanding checkpoint and every gate apply to both tiers.
- **Thread feedback, don't override.** When the user asks for changes, fold their words into a fresh subagent brief rather than editing code yourself.
- **You don't manage fan-out.** To try an alternative approach, the user opens another worktree window — `claude-dispatch new <repo> <N> <suffix> -i` names them `issue-N-<suffix>` so worktrees and windows don't collide. Comparing attempts is the user's call.
