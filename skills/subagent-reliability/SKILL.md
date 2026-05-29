---
name: subagent-reliability
description: Hardening patterns for dispatching Cursor `Task` subagents so they do not silently hang for 10-50 minutes. The parent agent reads this BEFORE writing the prompt for any subagent it intends to run in the background or in parallel. Covers wall-clock budgets, single-deliverable contracts, WebSearch-over-WebFetch defaults, no-recursive-dispatch, no-follow-up-loops, batch sizing, git-log monitoring, kill/relaunch recovery, and fallback to the parent chat agent. Trigger terms: "subagent hangs", "subagent timeout", "Task tool", "long-running agent", "dispatch subagent reliably", "subagent silent failure", "parallel subagents Cursor", "subagent stalled", "background agent stuck", "WebFetch hang".
---

## When to use this skill

Read this skill BEFORE you draft the prompt for ANY Cursor `Task` subagent that:

- runs in the background (`run_in_background: true`), OR
- runs in parallel with other subagents, OR
- is expected to take more than ~3 minutes, OR
- performs web fetches, file crawls, or any I/O of unknown duration.

The failure mode this skill prevents is the silent 10-50 minute stall: the subagent prints nothing, the transcript shows no progress, and the parent agent waits indefinitely while no real work happens. This is a recurring, observed failure in long-running multi-agent Cursor sessions — not a hypothetical.

Do NOT use this skill for: synchronous one-shot subagents that you explicitly await and that complete in under a minute.

## The 10-line prompt checklist (one-line summary)

Every dispatched subagent prompt must contain these ten lines (full templates and exact wording in `references/prompt-checklist.md`):

1. Hard wall-clock budget: `Time budget: N minutes. If you exceed this, stop and write what you have.`
2. Exactly one deliverable path: `Write to <exact/path>. No other files.`
3. Exactly one commit: `Make one commit, then exit. No second commit.`
4. Commit message format pinned: `Commit message: "<exact prefix> <slug>"`.
5. Branch pinned: `Work on branch <branch>. Do not create new branches.`
6. WebSearch by default: `Use WebSearch. Do NOT use WebFetch unless explicitly listed below.`
7. Per-fetch timeout if WebFetch is unavoidable: `For each WebFetch: 90s budget. If slow, drop it and move on.`
8. No recursive dispatch: `Do NOT dispatch your own subagents. You are a leaf.`
9. Exit-on-commit contract: `After your commit, your final reply is your only remaining output. Do not continue work.`
10. Final reply format: `Final reply: one short paragraph — files written, line count, commit SHA, one-sentence summary. Nothing else.`

## Standard subagent prompt template

Copy this block, fill the `<…>` slots, and paste it as the `prompt` for `Task`. Do not omit lines.

```markdown
# Task: <one-line task name>

## Time budget
Time budget: <N> minutes wall-clock. If you exceed this, stop, commit what you have, and exit.

## Deliverable (exactly one)
Write to: <repo>/<exact/relative/path>
Do not create or modify any other file. No "while I'm here" edits.

## Branch and commit
Work on branch: <feat/...>
Do not create new branches. Do not push. Do not open a PR.
Make exactly one commit:

    git add <exact/relative/path>
    git commit -m "<exact commit message>"

Then exit. No second commit. No amend. No follow-up exploration.

## Tools
- Default to WebSearch for any web research.
- Do NOT use WebFetch unless the URL is in this list: <explicit URLs or "none">.
  - If WebFetch is allowed: each call has a 90-second budget. If it stalls, skip the URL and move on.
- Do NOT dispatch other subagents (no Task tool). You are a leaf agent.
- Do NOT start long-running background processes.

## Scope guardrails
- One deliverable. Do not "also consider", "additionally explore", or "follow up on".
- If you finish early, exit. Do not look for extra work.
- If you discover the task is bigger than the budget, write what you have, commit, exit, and report the gap in your final reply.

## Final reply format
One short paragraph only:
- files written (paths)
- total line count
- commit SHA (`git rev-parse HEAD`)
- one-sentence summary of what's in the file

Nothing else. No bullet lists of "things I considered". No next-steps section.
```

### Why each line matters

- **Time budget**: gives the agent a self-policing trigger. Without it, the agent will burn unbounded minutes on a single tool call.
- **One deliverable, one path**: makes "is it done?" a `git log` query, not a transcript inspection.
- **Pinned commit message**: lets the parent grep for the commit deterministically.
- **WebSearch default**: `WebFetch` is the #1 cause of silent hangs (see catalog).
- **No recursive dispatch**: depth-2 subagents double the surface area for stalls and confuse the audit trail.
- **Exit-on-commit**: prevents phantom commits that break the parent's `git merge --ff-only`.
- **Final reply format**: prevents the post-deliverable monologue that wastes the remaining budget.

## Batch sizing

When the task is a list (e.g., "summarize these 20 sources", "review these 12 files"), do NOT send all of it to one subagent.

- Cap each subagent at **≤5 items**.
- Dispatch multiple subagents in parallel, each with its own deliverable path (`batch-01.md`, `batch-02.md`, …).
- A 5-item subagent that succeeds beats a 20-item subagent that hangs at item 14 and loses everything.

If a subagent's scope grows past 5 items mid-task, that's a sign to kill it and re-split.

## Monitoring

Subagent transcripts in Cursor are low-fidelity for long-running work — you will often see no output for many minutes even when the agent is healthy, and you will sometimes see no output when the agent is dead. The transcript is not a reliable liveness signal.

The reliable signals, in order:

1. `git log --oneline <branch>` on the subagent's worktree — has the expected commit landed?
2. `git status` and `ls <deliverable-path>` — is the file being written?
3. The subagent's final reply — only trustworthy once the commit exists.

**Operational rule**: when checking on a subagent, run `git log --oneline -5 <branch>` first. If the commit is there, the agent succeeded regardless of what the transcript says. If the commit is not there and the budget has elapsed, treat it as stalled.

"Alive but quiet" looks like: no transcript output for 5+ minutes, but `git log` shows no commit yet AND you are still within budget. Wait. Do not interrupt.

"Stalled" looks like: past the budget, no commit, transcript silent. Proceed to recovery.

## Recovery protocol

When a subagent stalls past **2× its declared budget**:

1. **Kill it.** Do not wait further. Every extra minute is wasted.
2. **Inspect git.** Run `git log --oneline -10 <branch>` and `git status` on the subagent's worktree. Often there is partial progress (a half-finished file, an uncommitted draft). Capture what's there.
3. **Diagnose the cause.** Most common: an unbounded `WebFetch`, an oversized batch, or a recursive dispatch attempt. Consult `references/failure-mode-catalog.md`.
4. **Relaunch fresh** with stricter constraints:
   - Smaller batch (halve the item count).
   - WebFetch removed or per-URL timeout tightened.
   - Budget tightened, not loosened.
   - Deliverable path renamed (e.g., `batch-03-retry.md`) so partial output doesn't collide.
5. **Cap retries at 2.** If the same task fails twice, do not launch a third subagent. Escalate.

### Fallback to chat-agent

After 2 failed subagent attempts on the same task, the parent chat agent does the work inline. The cost of one parent-agent execution is almost always less than the cost of three failed subagent dispatches plus the operator confusion they cause.

Signs it is time to fall back now:

- The task is small enough that the parent can do it in under 10 minutes.
- The task requires tools the subagent keeps mishandling (e.g., a flaky WebFetch endpoint).
- The operator is repeatedly killing and relaunching the same subagent shape.

Do not treat fallback as failure. Treat it as the correct allocation of agent time.

## Additional resources

- Read `references/prompt-checklist.md` BEFORE writing any subagent prompt. It is the literal 10-item checklist with exact prompt-line wording to copy in.
- Read `references/failure-mode-catalog.md` WHEN you observe a stall, a phantom commit, a transcript blackout, or any subagent symptom you don't recognize. It is a (symptom, cause, fix) table indexed by what you see.
