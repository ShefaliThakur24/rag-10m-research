# Brief templates for multi-agent-research

Drop-in templates for `AGENT_BRIEF_SHARED.md` and per-lane / combiner / reviewer briefs. Copy, fill the `<bracketed>` placeholders, commit on `main` before dispatching.

Forward slashes everywhere. Kebab-case for lane names and slugs.

---

## Template 1: `AGENT_BRIEF_SHARED.md` (committed on main)

Every lane, the combiner, and the reviewer re-read this at the start of their run.

```markdown
# Standing orders for all <session-slug> agents

You are one of <N> subagents working in parallel on <one-sentence goal>. Re-read this file at the start of your run. Also re-read your role-specific BRIEF.md.

## 1. Audience and tone

- Audience: <who reads the final artifact>.
- Lead with approach, not concept. Assume the reader knows <list of things to NOT explain>.
- Forbidden openers: "In recent years...", "<X> has emerged as...", "<X> is a technique that...", restating the section title in the first sentence.
- Tradeoffs explicit with regimes. Replace "X is generally better" with "X wins when <numeric regime>". If you don't have the number, mark the claim `confidence: low` instead of hedging.

## 2. Write isolation (HARD invariant)

Each role writes ONLY to its allowed paths on ONLY its branch. Reviewer rejects out-of-scope writes.

| Role | Branch | Allowed write paths |
|---|---|---|
| lane <lane-1> | `lane/<session>/<lane-1>` | `v2/<lane-1>.md`, `lanes/<lane-1>/notes.md` |
| lane <lane-2> | `lane/<session>/<lane-2>` | `v2/<lane-2>.md`, `lanes/<lane-2>/notes.md` |
| ... | ... | ... |
| combiner | `main` | `FINAL.md`, `doc/**`, `doc/images/**` |
| reviewer | `main` | `review/**`, ff-merges of lane branches |

Read access is unrestricted; only writes are scoped.

## 3. Hard contracts (every subagent)

- One deliverable file path, named exactly in your role brief. No other writes.
- Single commit, then exit. No amends. No follow-up dispatches.
- Wall-clock budget: <N> minutes. Commit what you have when the budget hits.
- Prefer `WebSearch` over `WebFetch`. `WebFetch` on PDFs / large pages is the #1 hang risk.
- Source cap: 5 per subagent. If you need more, stop and flag it in your final reply.
- No nested `Task` calls.

## 4. Image policy (combiner only)

- Default to mermaid for: architecture, data flow, sequence, state machines, tradeoff matrices.
- `GenerateImage` only when a richer visual genuinely beats prose.
- Save to `doc/images/<descriptive-name>.png`. Every generated image gets a one-paragraph caption explaining the takeaway and an HTML `<!-- regen-prompt: ... -->` comment immediately beside the markdown reference.

## 5. Final reply format

One short paragraph naming: file written, commit SHA, any unreachable sources or skipped items. Nothing else.

## 6. Forbidden actions

- Editing files outside your allowed paths.
- Force-push, history rewrite, `git commit --amend`.
- Pushing to remotes or opening PRs (unless your role brief explicitly says so).
- Dispatching subagents from inside a subagent.
- Citing a URL you didn't actually read.
```

---

## Template 2: `lanes/<lane>/BRIEF.md` (one per lane, committed on main)

```markdown
# Brief: lane <lane-name>

## Goal

<one paragraph stating what this lane investigates / authors and why it's independent of the other lanes>

## Deliverable

Single file at `v2/<lane-name>.md`. Length target: <e.g. 1500-1800 words>. Structure:

1. <section 1 name> — <one-line description>
2. <section 2 name> — <one-line description>
3. ...

## Acceptance bar

Reviewer approves if all of:

- File exists at the exact path above.
- Sections in the order listed above, with the named headings.
- Each claim either cites a source or is marked `confidence: low`.
- No forbidden openers (see `AGENT_BRIEF_SHARED.md` section 1).
- No edits to paths outside `v2/<lane-name>.md` and `lanes/<lane-name>/notes.md`.

## Source scope

In-scope: <list of source types / domains / search terms this lane should cover>.
Out-of-scope (other lanes handle these): <list>.

Source cap: 5. If you need more, stop and flag it.

## Working notes

You may append working notes to `lanes/<lane-name>/notes.md` (single commit at the end, together with the deliverable). This is optional and not reviewed.
```

---

## Template 3: dispatch prompt for a lane subagent

Paste this into the `Task` tool's `prompt` field, with placeholders filled. The prompt re-states the hard contracts because the subagent does not see this template file.

```
You are the "<lane-name>" lane subagent for the <session-slug> session.

Workflow:
1. cd /absolute/path/to/<session-slug>-<lane-name>
2. git fetch origin && git rebase main
3. Read AGENT_BRIEF_SHARED.md and lanes/<lane-name>/BRIEF.md end-to-end.
4. Produce exactly one file at v2/<lane-name>.md per the brief.
5. Optionally append to lanes/<lane-name>/notes.md.
6. Single commit on this branch:
     git add v2/<lane-name>.md lanes/<lane-name>/notes.md
     git commit -m "lane(<lane-name>): <one-line summary>"
7. Exit. Do not push. Do not open a PR. Do not dispatch any subagents.

Hard rules:
- Wall-clock budget: 10 minutes. Commit what you have when it hits.
- Prefer WebSearch over WebFetch. WebFetch on PDFs is the #1 hang risk.
- Source cap: 5.
- Write only to v2/<lane-name>.md and lanes/<lane-name>/notes.md.
- No nested Task calls.

Final reply: one short paragraph naming the file written, commit SHA, and any unreachable sources. Nothing else.
```

---

## Template 4: combiner brief + dispatch prompt

`combiner/BRIEF.md` (committed on main):

```markdown
# Brief: combiner

## Goal

Integrate all merged lane deliverables (`v2/*.md`) plus any pre-existing `doc/` content into one consolidated artifact at `<artifact-path, e.g. FINAL.md>`.

## Deliverable

Single file at `<artifact-path>`. Structure:

1. <section 1>
2. <section 2>
3. ...

## Acceptance bar

- File exists at the exact path above.
- Every lane deliverable is reflected (no lane silently dropped).
- Cross-lane contradictions are surfaced explicitly, not papered over.
- Images (if any) follow the image policy in `AGENT_BRIEF_SHARED.md` section 4.

## Tools

File IO, `GenerateImage`. No `Task`.
```

Dispatch prompt:

```
You are the combiner subagent for the <session-slug> session.

Workflow:
1. cd /absolute/path/to/<main-repo>
2. git checkout main && git pull --ff-only
3. Read AGENT_BRIEF_SHARED.md and combiner/BRIEF.md.
4. Read every file in v2/*.md and any pre-existing doc/ content.
5. Produce <artifact-path>. Use mermaid by default; GenerateImage only when it beats prose.
6. Single commit on main:
     git add <artifact-path> doc/images/
     git commit -m "combiner: integrate <N> lanes into <artifact-path>"
7. Exit. Do not dispatch any subagents.

Hard rules:
- Wall-clock budget: 15 minutes.
- No nested Task calls.
- Every generated image gets a caption + regen-prompt HTML comment.

Final reply: one short paragraph naming the artifact path, commit SHA, image count.
```

---

## Template 5: reviewer brief + dispatch prompt

`reviewer/BRIEF.md` (committed on main):

```markdown
# Brief: reviewer

## Goal

Evaluate the combined artifact at `<artifact-path>` against the rubric below. Approve, request changes, or rewrite.

## Rubric (any clause failure = reject)

1. **Structure**: sections present in the order specified by combiner/BRIEF.md.
2. **Coverage**: every lane deliverable is reflected.
3. **Audience fit**: no prose explaining things the audience already knows (see `AGENT_BRIEF_SHARED.md` section 1).
4. **Tradeoff specificity**: no hedge phrases without a numeric regime.
5. **Image necessity**: every generated image earns its place (mermaid would not have sufficed) and has a caption + regen-prompt.
6. **Forbidden openers**: none present.

NOTE: Quote-verification is intentionally NOT in this rubric. It requires `WebFetch` which is the #1 hang risk. Sample-verify by hand post-hoc if it matters.

## Outcomes

- Approve: write `review/log/<sha>.md` with a one-line summary. Done.
- Request changes: write `review/feedback/<artifact>-<sha>.md` listing rubric clauses violated, quoting offending lines, suggesting fixes. Single commit on main. Orchestrator dispatches a fix subagent.
- Rewrite: edit the artifact directly in a single commit on main. Use sparingly.

## Tools

File IO, `git`. No `WebFetch`. No `Task`.
```

Dispatch prompt:

```
You are the reviewer subagent for the <session-slug> session.

Workflow:
1. cd /absolute/path/to/<main-repo>
2. git checkout main && git pull --ff-only
3. Read AGENT_BRIEF_SHARED.md and reviewer/BRIEF.md.
4. Read <artifact-path>.
5. Evaluate against the rubric in reviewer/BRIEF.md.
6. Pick exactly one outcome: approve | request-changes | rewrite.
7. Single commit on main with the appropriate review/ file(s).
8. Exit. Do not dispatch any subagents.

Hard rules:
- Wall-clock budget: 3 minutes.
- No WebFetch. No nested Task calls.
- Pick one outcome; do not mix.

Final reply: one short paragraph naming the outcome, the review file path(s), and the commit SHA.
```
