# Orchestration recipe: worktree fan-out + combiner + reviewer

Detailed recipe for the pattern described in `../SKILL.md`. Numbers and shapes are from a real RAG research run that produced a 617-line `FINAL.md` across 4 lanes in roughly 2 hours of wall-clock.

## 0. Mental model

- **One lane = one branch + one worktree + one deliverable file + one subagent.**
- **Isolation by filesystem, convergence by git merge.** Lanes never read each other's in-flight writes.
- **The orchestrator (you, in chat) is the only thing that calls `Task`.** Lane subagents do not dispatch.
- **Three roles**: lane (produces), combiner (integrates), reviewer (gates). Three role types, distinct briefs.

## 1. Decide whether to fan out

Fan out only if all four hold:

1. The goal decomposes into 3-6 independent sub-areas. Fewer than 3: serialize. More than 6: budget will blow.
2. Each sub-area produces a single file (or a single tightly-bounded directory).
3. Each sub-area fits in ~5-10 min of subagent wall-clock and ~5 sources of input. If a lane needs 20 sources, split it.
4. The sub-areas don't need each other's output mid-flight. Read-only references to main are fine.

If any of these fails, prefer a single subagent or do it inline.

From the real run: 4 lanes (`v2-ingest-embed`, `v2-ingest-graph`, `v2-generate-grounding`, `v2-generate-multihop`), each producing one ~1500-1800 word markdown file in `v2/<lane>.md`. Total fan-out wall-clock: ~10 minutes.

## 2. Scaffold the session

Pick a session slug and lane names. Use kebab-case, forward slashes everywhere.

```bash
SESSION=research-2026-05-29
LANES=(ingest-embed ingest-graph generate-grounding generate-multihop)

# main must be clean
git checkout main && git pull --ff-only && git status

# one branch + one worktree per lane, all off current main
for L in "${LANES[@]}"; do
  git branch "lane/${SESSION}/${L}" main
  git worktree add "../${SESSION}-${L}" "lane/${SESSION}/${L}"
done

git worktree list   # sanity check
```

Worktrees live as sibling directories to the main repo. The branch name pattern `lane/<session>/<lane>` keeps history readable and lets you grep merges.

## 3. Write the briefs

Two files, both on `main`, committed before any subagent runs:

- `AGENT_BRIEF_SHARED.md` — universal rules every lane / combiner / reviewer re-reads.
- `lanes/<lane>/BRIEF.md` — one per lane, names the deliverable path and acceptance bar.

Use `../references/agent-brief-template.md` as the starting point. Commit on main:

```bash
git add AGENT_BRIEF_SHARED.md lanes/ && git commit -m "scaffold: ${SESSION} lanes"
```

Each worktree picks up the briefs via `git rebase main` at the start of its subagent's run.

## 4. Dispatch the fan-out

Dispatch one `Task` subagent per lane, **all in one assistant turn** (parallel tool calls). Each prompt MUST include, verbatim:

```
You are the "<lane>" lane subagent.

1. cd /absolute/path/to/<session>-<lane>
2. git fetch origin && git rebase main
3. Read AGENT_BRIEF_SHARED.md and lanes/<lane>/BRIEF.md end-to-end.
4. Produce exactly one file: <relative path to deliverable, e.g. v2/<lane>.md>.
5. Single commit on this branch, message: "lane(<lane>): <one-line summary>".
6. Exit. Do not push. Do not open a PR. Do not dispatch any subagents.

Hard rules:
- Wall-clock budget: 10 minutes.
- Prefer WebSearch over WebFetch. WebFetch on PDFs is the #1 hang risk.
- Source cap: 5 per subagent.
- Write only to your deliverable path + lanes/<lane>/notes.md.
- No nested Task calls.

Final reply: one short paragraph naming the file written, commit SHA, and any unreachable sources.
```

Set the subagent to background mode if your orchestrator supports it; otherwise let them run synchronously.

## 5. Reap, rebase, merge

Once all lane subagents have replied (or timed out at the 10-min budget):

```bash
git checkout main && git pull --ff-only
for L in "${LANES[@]}"; do
  B="lane/${SESSION}/${L}"
  git checkout "$B"
  git rebase main          # picks up scaffold; should be a no-op given write isolation
  git checkout main
  git merge --ff-only "$B" || {
    echo "ff-only failed for $B; squash-merging"
    git merge --squash "$B"
    git commit -m "lane(${L}): squash-merge after divergence"
  }
done
```

Why rebase-before-merge: if `main` advanced (e.g. you committed a small fixup), the lane branch will refuse `--ff-only`. Rebasing keeps history linear. In the real run, a no-op synth commit caused exactly this; the fix was the explicit rebase step above.

## 6. Combiner pass

Dispatch ONE combiner subagent. Its brief:

- **Inputs**: every merged lane deliverable (e.g. `v2/*.md`), plus any pre-existing `doc/` content.
- **Output**: one consolidated artifact at a fixed path (e.g. `FINAL.md`).
- **Allowed tools**: file IO, `GenerateImage`. No `Task`.
- **Image policy**: mermaid by default; `GenerateImage` only when a richer visual beats prose. Every generated image needs a one-paragraph caption explaining the takeaway plus an HTML `<!-- regen-prompt: ... -->` comment beside the markdown reference. Save under `doc/images/<descriptive-name>.png`.
- **Single commit on main**, message: `combiner: integrate <N> lanes into <artifact>`.
- **Wall-clock budget**: 15 min (combiner does more synthesis than any single lane).

Do NOT ask lane subagents to integrate with each other mid-flight. Integration is the combiner's job alone. From the real run: a separate chat-agent combiner pass produced `FINAL.md` (617 lines) integrating 4 deep dives plus existing `doc/` content and 6 embedded images.

## 7. Reviewer pass

Dispatch ONE reviewer subagent against an explicit rubric. Outcomes:

- **Approve**: post a short approval, ff-merge if reviewer worked on a branch, otherwise just leave a `review/log/<sha>.md` note. Done.
- **Request changes**: write `review/feedback/<artifact>-<sha>.md` listing the specific rubric clauses violated, quoting offending lines, suggesting fixes. Orchestrator dispatches a fix subagent that targets only those clauses.
- **Rewrite**: reviewer edits the artifact directly in a single commit. Use sparingly; preserves linear history but bypasses the lane.

Hard reviewer budget: **3 min**.

**Drop quote-verification from the rubric** if it requires `WebFetch`. From the real run: a v1 reviewer subagent stalled ~15 min on a `WebFetch` for arXiv PDFs. The fix was to switch to `WebSearch`, set a 3-min hard budget, and drop quote-verification from the rubric. Sample-verify quotes by hand post-hoc if it matters.

In the real run, the v2 reviewer posted 6 inline + 3 conversation PR comments, submitted a review, and squash-merged in under 3 min.

## 8. Failure mode catalog (from the real run)

### Silent hangs on web fetches

**Observed**: v1 lane subagents (engineering, community, papers batch 2) hung silently past 30-50 min with no output and no commit. v1 reviewer stalled ~15 min on `WebFetch` for arXiv PDFs.

**Mitigation**:
- Default to `WebSearch`. Use `WebFetch` only for a known-small, known-fast URL (HTML page < 100 KB, no PDF).
- Cap sources per subagent at 5. If a lane needs more, split it into two lanes.
- Single-batch subagents only: one fan-out, one merge. No per-subagent inner loops.
- Set explicit per-fetch and total wall-clock budgets in the brief.

### Runaway "let me go further" agents

**Observed**: Some v1 subagents made 3+ commits, kept fetching more sources, started editing files outside their lane.

**Mitigation**:
- "Single commit, then exit" must appear verbatim in the dispatch prompt AND the lane brief.
- "One deliverable file path" must be named exactly.
- Reviewer rejects any commit that touches paths outside the lane's allowed set.

### Cross-lane file edits

**Observed**: A subagent decided it could "help" by tweaking another lane's notes.

**Mitigation**:
- Write-isolation table in `AGENT_BRIEF_SHARED.md` listing exactly which paths each role can write.
- Reviewer rejects out-of-scope writes; orchestrator reverts.

### Lane branch diverged from main

**Observed**: A no-op synth commit landed on `main` mid-run. Lane branches then failed `git merge --ff-only`.

**Mitigation**: Always `git rebase main` on the lane branch before merging. If that fails for reasons other than the no-op (e.g. real conflicts), squash-merge with a clean commit message; do not try to surgically rebase mid-session.

### Combiner re-fans-out

**Observed**: A combiner that was given access to `Task` tried to spawn its own helper subagents and blew the wall-clock budget.

**Mitigation**: Forbid nested `Task` calls in the combiner brief. The combiner is a single subagent with file IO and image generation only.

## 9. Linear-history merge protocol (summary)

```
main: A --- B (scaffold) --- C (lane-1) --- D (lane-2) --- E (lane-3) --- F (lane-4) --- G (combiner) --- H (reviewer)
                  \             \             \             \             \
                   lane/.../1    lane/.../2    lane/.../3    lane/.../4    (merged via ff-only or squash)
```

- Lanes always branch off the same `B` (scaffold commit).
- Rebase each lane onto current main before merging.
- Prefer `--ff-only`. Fall back to `--squash` with a clean message if it fails.
- Combiner and reviewer commit directly on `main`.

## 10. Stop conditions

The orchestrator stops when ANY of these holds:

- Reviewer approves the combined artifact and there are no outstanding fix requests.
- Wall-clock for the whole session exceeds the budget you set up front (default: 2 hours for a research-scale run).
- More than one lane has hung past its budget and re-dispatch did not help. In this case, ship what's combined, document the gap, and stop.
