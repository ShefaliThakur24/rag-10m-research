# Subagent prompt checklist (10 items)

Copy each item's prompt-line verbatim (with `<…>` slots filled) into the prompt you pass to `Task`. Order matters: budget first, exit-contract last.

## 1. Hard wall-clock budget

Why: the agent self-polices. Without an explicit budget, a single tool call can burn unbounded minutes.

Prompt line:

```
Time budget: <N> minutes wall-clock. If you exceed this, stop, commit what you have, and exit.
```

Typical N: 3 min for a single WebSearch summary, 5 min for a single ≤5-item batch, 8 min for a single file synthesis, 10 min hard cap on anything labeled "short".

## 2. Exactly one deliverable path

Why: makes "is it done?" a `git log` / `ls` query, not a transcript inspection.

Prompt line:

```
Write to: <repo>/<exact/relative/path>
Do not create or modify any other file. No "while I'm here" edits.
```

## 3. Exactly one commit

Why: prevents phantom commits that break `git merge --ff-only` on the parent's branch.

Prompt line:

```
Make exactly one commit, then exit. No second commit. No amend.
```

## 4. Pinned commit message

Why: lets the parent grep `git log` deterministically for the subagent's commit.

Prompt line:

```
Commit message exactly: "<prefix> <slug>"
e.g. "feat(skills): add subagent-reliability skill"
```

## 5. Pinned branch

Why: prevents the subagent from creating its own branch, drifting, or pushing.

Prompt line:

```
Work on branch: <feat/...>
Do not create new branches. Do not push. Do not open a PR.
```

## 6. WebSearch by default, WebFetch denied unless explicit

Why: `WebFetch` is the #1 cause of silent hangs. arXiv PDF fetches have been observed to stall 15+ minutes with no output.

Prompt line:

```
Use WebSearch for any web research.
Do NOT use WebFetch unless the URL is in this list: <explicit URLs or "none">.
```

## 7. Per-fetch timeout if WebFetch is unavoidable

Why: even allowed WebFetches can hang. The agent must be told to drop slow ones.

Prompt line:

```
For each WebFetch call: 90-second budget. If it stalls or returns slowly, skip the URL and move on. Do not retry the same URL more than once.
```

## 8. No recursive dispatch

Why: depth-2 subagents double the stall surface and break the audit trail. Observed: a subagent dispatched its own subagent for citation verification, doubling latency and producing commits the parent could not trace.

Prompt line:

```
Do NOT dispatch other subagents. Do NOT use the Task tool. You are a leaf agent.
```

## 9. Exit-immediately-after-commit

Why: prevents the "kept exploring after the commit" pattern that produces phantom commits and busts `git merge --ff-only`.

Prompt line:

```
After your commit, your only remaining output is the final reply. Do not continue work. Do not "explore further". Do not "consider also".
```

## 10. Final reply format

Why: prevents post-deliverable monologue that wastes budget and pollutes the transcript.

Prompt line:

```
Final reply: one short paragraph only.
- files written (paths)
- total line count
- commit SHA (`git rev-parse HEAD`)
- one-sentence summary

Nothing else. No bullet lists of considerations. No next-steps section.
```

## Quick paste-block (all 10, fill the slots)

```
Time budget: <N> minutes wall-clock. If you exceed this, stop, commit what you have, and exit.

Write to: <repo>/<exact/relative/path>
Do not create or modify any other file.

Work on branch: <feat/...>. Do not create new branches. Do not push. Do not open a PR.
Make exactly one commit, then exit. No second commit. No amend.
Commit message exactly: "<prefix> <slug>"

Use WebSearch for any web research.
Do NOT use WebFetch unless the URL is in this list: <explicit URLs or "none">.
If WebFetch is allowed: 90-second budget per call. Skip slow URLs. No retries beyond 1.

Do NOT dispatch other subagents. Do NOT use the Task tool. You are a leaf agent.

After your commit, your only remaining output is the final reply. Do not continue work.

Final reply: one short paragraph — files written, line count, commit SHA, one-sentence summary. Nothing else.
```

## Batch-sizing addendum

If the task is a list of items:

- Cap each subagent at **≤5 items**.
- Give each subagent its own deliverable path (`batch-01.md`, `batch-02.md`, …).
- Dispatch in parallel; each is independently recoverable.
- A 5-item subagent that succeeds beats a 20-item subagent that hangs at item 14.
