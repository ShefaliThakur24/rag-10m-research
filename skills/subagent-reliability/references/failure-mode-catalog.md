# Subagent failure-mode catalog

Index this file by the symptom you observed. Each entry: (symptom, cause, fix, prompt-line to add).

All examples are observed in a real RAG-10M research session running 3 lane subagents + 1 synth + 1 reviewer in parallel over multiple hours.

## 1. Silent stall on `WebFetch` (arXiv PDF case)

| Field | Value |
|---|---|
| Symptom | v1 reviewer subagent ran 15+ minutes with no transcript output before being killed. `git log` showed no commit. |
| Cause | `WebFetch` on arXiv PDF URLs hangs indefinitely with no timeout, no progress signal. Confirmed reproducible across multiple arXiv IDs. |
| Fix | Default to `WebSearch`. Drop `WebFetch` entirely for this lane. Re-dispatch with 3-minute budget. |
| Prompt line to add | `Use WebSearch for any web research. Do NOT use WebFetch. Time budget: 3 minutes.` |

## 2. 30-50 minute timeout on multi-source synthesis

| Field | Value |
|---|---|
| Symptom | v1 engineering-lane and community-lane subagents both ran past 30 minutes with no commit, no transcript progress. |
| Cause | Single subagent given a 20+ source synthesis task. One slow source stalled the whole batch; the agent had no signal to drop it. |
| Fix | Split into batches of ≤5 sources. One subagent per batch. Each subagent gets an 8-minute budget. Parallel dispatch. |
| Prompt line to add | `Sources to summarize (exactly 5): <list>. Time budget: 8 minutes. If any single source is slow, skip it and move on.` |

## 3. Phantom commits / divergent git state

| Field | Value |
|---|---|
| Symptom | Subagent committed the deliverable, then continued exploring and made 2-3 additional commits. Parent's `git merge --ff-only <subagent-branch>` failed because the parent's branch had also moved. |
| Cause | Prompt did not include an exit-on-commit contract. Agent treated "done" as "ready for more work" instead of "exit". |
| Fix | Add explicit exit-on-commit clause. Final reply is the only remaining output. |
| Prompt line to add | `After your single commit, your only remaining output is the final reply. Do not continue work. Do not "explore further". No second commit. No amend.` |

## 4. Recursive subagent dispatch

| Field | Value |
|---|---|
| Symptom | A reviewer subagent dispatched its own subagent for citation verification. Parent saw doubled latency, two unfamiliar transcripts, and commits the parent could not trace to a known dispatch. |
| Cause | No depth cap. The subagent had access to the `Task` tool and used it. |
| Fix | Cap dispatch depth at 1. Subagents must be leaf agents. |
| Prompt line to add | `Do NOT dispatch other subagents. Do NOT use the Task tool. You are a leaf agent. Do all work yourself.` |

## 5. Transcript blackout on completed work

| Field | Value |
|---|---|
| Symptom | Subagent transcript showed minimal output for 6+ minutes. Operator assumed the agent was dead. In fact, the work had completed and the commit was already in `git log`. |
| Cause | Cursor subagent transcripts are low-fidelity for long-running work. The transcript is not a liveness signal. |
| Fix | Check `git log --oneline -5 <branch>` first, before assuming a subagent is stalled. The commit is the source of truth. |
| Prompt line to add | (parent-side procedure, not a prompt line) Always run `git log --oneline -5 <branch>` before deciding a subagent is stalled. |

## 6. "While I'm here" scope creep

| Field | Value |
|---|---|
| Symptom | Subagent was asked to write file A. Subagent wrote file A correctly, then also modified files B and C "for consistency", committing all three. Parent's diff review became much larger than expected. |
| Cause | Prompt did not constrain the deliverable to a single path. |
| Fix | Pin the deliverable to one exact path. Forbid all other writes. |
| Prompt line to add | `Write to: <exact/path>. Do not create or modify any other file. No "while I'm here" edits.` |

## 7. Budget burned on post-deliverable monologue

| Field | Value |
|---|---|
| Symptom | Subagent completed its commit at 4 minutes, then spent 6 additional minutes writing a long final-reply essay about considerations, alternatives, and possible next steps. Total wall-clock exceeded budget. |
| Cause | No final-reply format constraint. The agent treated the post-commit window as free time. |
| Fix | Constrain the final reply to a one-paragraph status line. Forbid bullet lists of considerations. |
| Prompt line to add | `Final reply: one short paragraph — files written, line count, commit SHA, one-sentence summary. Nothing else.` |

## 8. Two retries, no progress

| Field | Value |
|---|---|
| Symptom | Same subagent shape (same prompt, same task) was launched, killed, and relaunched twice. Third launch was being prepared. |
| Cause | No retry cap. Operator was in a "one more try" loop. |
| Fix | Cap subagent retries at 2 for the same task shape. After two failures, the parent chat agent does the work inline. |
| Prompt line to add | (parent-side procedure) After 2 failed subagent attempts on the same task, fall back to the parent chat agent doing the work directly. |

## 9. Deliverable path collision on retry

| Field | Value |
|---|---|
| Symptom | Retry subagent's partial output overwrote the killed subagent's partial output. Forensics on the original failure became impossible. |
| Cause | Retry used the same deliverable path as the original. |
| Fix | Rename the deliverable path on each retry (`batch-03.md` → `batch-03-retry.md`). |
| Prompt line to add | `Write to: <path>-retry<N>.md` for any relaunch after a stall. |

## 10. New branch created by subagent

| Field | Value |
|---|---|
| Symptom | Subagent created its own branch instead of working on the branch the parent expected. Parent's `git log <expected-branch>` showed no progress; the commits were on a different branch. |
| Cause | Prompt did not pin the branch explicitly. |
| Fix | Pin the branch by exact name. Forbid new-branch creation. |
| Prompt line to add | `Work on branch: <feat/...>. Do not create new branches. Do not switch branches.` |

## Quick triage table

| What you observe | Most likely cause | Jump to entry |
|---|---|---|
| 10+ min silence, no commit | `WebFetch` stall | 1 |
| 20+ min silence, multi-source task | Batch too large | 2 |
| `git merge --ff-only` fails | Phantom commits after deliverable | 3 |
| Mysterious extra transcripts | Recursive dispatch | 4 |
| Long silence, but commit is there | Transcript blackout (not a failure) | 5 |
| Unexpected files in diff | Scope creep | 6 |
| Budget exceeded, deliverable done early | Post-deliverable monologue | 7 |
| About to launch retry #3 | Retry cap violated | 8 |
| Forensics impossible on retry | Path collision | 9 |
| Expected branch has no commits | Subagent on a different branch | 10 |
