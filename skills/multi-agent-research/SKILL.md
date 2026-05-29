---
name: multi-agent-research
description: Orchestrates parallel research or authoring work by fanning a single goal out to N isolated Task subagents (one per "lane"), each operating in its own git worktree on its own branch, then converging the results via a combiner pass and a reviewer pass. Use when the user asks for parallel research, multiple subagents, worktree-based fan-out / fan-in, or a lane / synth / reviewer split; when a target decomposes into 3-6 independent sub-areas that each fit in one subagent's wall-clock budget; or when a single chat agent would otherwise serialize work that could be done concurrently. Do not use for small tasks, tightly-coupled refactors, or anything that needs shared mutable state across lanes.
---

# multi-agent-research

A reusable pattern for fanning research / authoring work across N parallel `Task` subagents in isolated git worktrees, then converging via a combiner pass plus a reviewer pass. Distilled from a real 2-hour RAG research run that produced a 617-line `FINAL.md` with 6 generated diagrams and a clean linear git history.

## When to use this skill

Use when:

- The target decomposes into **3-6 independent sub-areas** ("lanes") and each lane produces **one file**.
- Each lane fits in a single subagent's budget (roughly **5-10 min wall-clock**, one batch of sources, one deliverable).
- The user asks for "parallel research", "multiple subagents", "fan-out", "worktrees", or names the lane / synth / reviewer split explicitly.
- You would otherwise dispatch the same prompt template several times in sequence.

Do NOT use when:

- The task is small enough that one subagent (or no subagent) finishes it faster than the orchestration overhead (~2-3 min of setup + combine).
- Lanes need to read each other's in-flight writes. The pattern relies on write isolation; if lanes must coordinate, collapse them into one subagent.
- The work is a single tightly-coupled refactor or a bug hunt with one root cause.

## High-level shape

```
                  shared brief (AGENT_BRIEF_SHARED.md)
                                 |
            +-------+------------+-----------+--------+
            v       v            v           v        v
         lane-1  lane-2       lane-3      lane-4   (lane-N)
        worktree worktree    worktree    worktree
        branch-1 branch-2    branch-3    branch-4
            |       |            |           |
            +-------+-----+------+-----------+
                          v
                  combiner subagent
              (reads all lane files, writes FINAL.md)
                          v
                  reviewer subagent
            (applies rubric, approves or rewrites)
                          v
                merge to main (linear history)
```

One branch + one worktree per lane, all forked off `main`. Convergence happens by merging deliverables, never by lanes sharing files mid-flight.

## Setup checklist

Before launching subagents, read `references/orchestration-recipe.md` end-to-end. Then run these in the repo root:

```bash
# 1. Pick a session slug and lane names. Forward slashes only.
SESSION=research-2026-05-29
LANES=(ingest-embed ingest-graph generate-grounding generate-multihop)

# 2. Make sure main is clean.
git checkout main && git pull --ff-only && git status

# 3. Create one branch + one worktree per lane.
for L in "${LANES[@]}"; do
  git branch "lane/${SESSION}/${L}" main
  git worktree add "../${SESSION}-${L}" "lane/${SESSION}/${L}"
done

# 4. Write shared + per-lane briefs (templates in references/agent-brief-template.md).
$EDITOR AGENT_BRIEF_SHARED.md
for L in "${LANES[@]}"; do
  mkdir -p "lanes/${L}" && $EDITOR "lanes/${L}/BRIEF.md"
done
git add AGENT_BRIEF_SHARED.md lanes/ && git commit -m "scaffold: ${SESSION} lanes"

# 5. Fan out: dispatch one Task subagent per lane, all in parallel.
#    Each prompt: "cd <worktree>, read AGENT_BRIEF_SHARED.md and lanes/<L>/BRIEF.md, produce v2/<L>.md, single commit, exit."
```

Lane subagents commit to their own branch in their own worktree. They never touch other lanes' paths.

## Subagent contract

Every lane subagent MUST be told, verbatim, in its prompt:

1. **One deliverable file path**, named exactly. Example: `v2/ingest-embed.md`. No other writes.
2. **Single commit, then exit.** No follow-up commits, no amends, no further dispatches from inside the subagent.
3. **Wall-clock budget**, stated explicitly. Default: **10 min**. Subagent stops and commits whatever it has when the budget hits.
4. **Prefer `WebSearch` over `WebFetch`.** `WebFetch` on PDFs / large pages is the single biggest hang risk observed (one v1 reviewer stalled ~15 min on an arXiv PDF). Use `WebFetch` only for a known-small, known-fast URL.
5. **Source batch cap**: at most **5 sources per subagent**. More than that, split into two lanes.
6. **No nested subagents.** A lane subagent may not call `Task` itself. Only the orchestrator dispatches.
7. **Read-only access to other lanes is fine; writes are scoped** to its lane subdir + its deliverable file path.
8. **Final reply format**: one short paragraph naming the file written, the commit SHA, and any source it couldn't reach. Nothing else.

See `references/agent-brief-template.md` for the full per-lane brief template that encodes all of the above.

## Combiner + reviewer passes

After all lane subagents have committed and exited:

1. **Rebase + merge lane branches.** For each lane:
   ```bash
   git checkout "lane/${SESSION}/${L}"
   git rebase main           # picks up scaffold; should not conflict given write isolation
   git checkout main
   git merge --ff-only "lane/${SESSION}/${L}"
   ```
   If a lane diverged irrecoverably (e.g. a noop commit landed on main mid-run), squash-merge with a clean message instead.

2. **Combiner pass.** Dispatch ONE subagent (or do it inline in the chat agent) with this contract:
   - Inputs: every lane deliverable file, plus any pre-existing `doc/` content.
   - Output: one consolidated artifact (e.g. `FINAL.md`).
   - May call `GenerateImage` for diagrams that beat prose. Each image gets a caption and an HTML `regen-prompt` comment.
   - Single commit on `main`.
   - **Do NOT ask lane subagents to integrate with each other.** Combining is the combiner's job alone.

3. **Reviewer pass.** Dispatch ONE subagent against an explicit rubric (see the companion `karpathy-tier-reviewer` skill if installed; otherwise inline a rubric in the reviewer's brief). Reviewer outcomes:
   - **Approve**: comment + ff-merge / squash-merge. Done.
   - **Request changes**: write `review/feedback/<sha>.md` listing rubric-clause violations; orchestrator dispatches a fix subagent.
   - **Rewrite**: reviewer edits the artifact directly in a single commit (use sparingly; keeps merges linear).

   Hard reviewer budget: **3 min**. Drop quote-verification from the rubric if it requires `WebFetch`.

## Common failure modes and fixes

| Failure mode | Symptom | Fix |
|---|---|---|
| Silent hang on web fetch | Subagent runs > 15 min with no output, no commit | Switch the lane to `WebSearch`; add per-fetch timeout; cap sources at 5 per subagent |
| Runaway "let me go further" | Subagent makes 3+ commits, edits files outside its lane | Restate "single commit, then exit" and "one deliverable path" in the brief; kill and re-dispatch |
| Cross-lane write collision | Two lanes touch the same file; merge conflicts on convergence | Enforce write-isolation table in `AGENT_BRIEF_SHARED.md`; reviewer rejects out-of-scope writes |
| Lane diverged from main | `git merge --ff-only` fails after a no-op synth commit | Rebase the lane branch onto current main first, then ff-merge; squash-merge as last resort |
| Combiner re-fans-out | Combiner spawns more subagents, blows the budget | Forbid nested `Task` calls in the combiner brief |
| Reviewer stalls on quote-verification | `WebFetch` of cited URLs hangs | Drop quote-verification from rubric; sample-verify by hand post-hoc if needed |
| Lane subagent writes wrong path | Deliverable lands at `lane/foo.md` instead of `v2/foo.md` | Name the exact path in the brief AND in the dispatch prompt; reviewer rejects on mismatch |

## Additional resources

- **Read `references/orchestration-recipe.md` BEFORE launching subagents.** It contains the full worktree + dispatch recipe with the exact commands used in the real RAG run that produced this skill.
- **Read `references/agent-brief-template.md` WHEN authoring per-lane briefs.** Drop-in templates for `AGENT_BRIEF_SHARED.md` and `lanes/<lane>/BRIEF.md` with all hard contracts pre-filled.
