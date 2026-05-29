# Standing orders for all RAG-10M research agents

You are one of five agents working in parallel on a production-grade design document for "10M+ documents, near-zero hallucination" RAG. Re-read this file at the start of every batch. Re-read your lane/synth/reviewer-specific `BRIEF.md` too.

## 1. Audience and tone (HARD, reviewer enforces)

- **Audience**: senior ML / staff engineers shipping production RAG. They know vector DBs, embeddings, chunking, transformers, basic RAG. **Do not explain any of these.**
- **Lead with approach, not concept.** Write `We use HNSW because <numeric tradeoff>`, not `HNSW is a graph-based ANN structure that...`. Concept explanations belong in `appendix/concepts.md` only.
- **Inline-concept policy**: when a less-common term appears (RAPTOR, ColBERT late interaction, RRF, MMR, MIPS, PQ/SQ quantization, GraphRAG, contextual retrieval, FActScore, FAISS-IVF, etc.), do this and only this:
  - In prose: `<term> ([ref](appendix/concepts.md#<anchor>))` with a one-line tip if needed (e.g. `RAPTOR (hierarchical summary tree, [ref](appendix/concepts.md#raptor))`).
  - In appendix: 2-4 sentences max + one canonical link. No more. Reviewer rejects anything longer.
- **Tradeoffs explicit with regimes.** Replace `X is generally better` with `X wins when corpus < N and recall@10 > 0.9 (cite)`. If you don't have the number, mark the claim `confidence: low` instead of hedging in prose.
- **Forbidden openers** (reviewer rejects on sight):
  - `In recent years...`, `In the rapidly evolving...`, `<X> has emerged as...`
  - `<X> is a technique that...`, `<X> refers to...`, `<X> is the process of...`
  - Restating the section title in the first sentence.

## 2. Write isolation (HARD invariant)

Each agent writes to ONLY these paths on ONLY its branch. Violating this triggers immediate reject by the reviewer.

| Agent | Branch | Allowed write paths |
|---|---|---|
| papers lane | `lane/papers` | `lanes/papers/**` only |
| engineering lane | `lane/engineering` | `lanes/engineering/**` only |
| community lane | `lane/community` | `lanes/community/**` only |
| synth | `synth/main` | `synth/**`, `doc/**`, `appendix/**`, `CHANGELOG.md` |
| reviewer | `main` | `main` (ff-merges), `review/**`, `skills/rag10m-*/**` |

Read access is unrestricted; only writes are scoped.

## 3. Batch + merge protocol

A **batch** = ~10 sources processed OR ~20 claims added OR ~1 doc section rewritten (synth only).

Per-batch loop (lanes):

1. `cd <your worktree>`
2. `git fetch && git rebase main` (picks up new skills and shared-brief changes; should never conflict because of write isolation)
3. Read `AGENT_BRIEF_SHARED.md`, your lane's `BRIEF.md`, and `skills/` for any new accepted skills.
4. Check `review/feedback/<your-lane>-*.md` for unaddressed feedback; if present, address it FIRST.
5. Pick 5-10 unread items from your scope (from `sources/seed.yaml` or leads discovered in prior batches).
6. For each item: append 1 source row to `lanes/<lane>/sources.jsonl`, append 1-5 claim rows to `lanes/<lane>/claims.jsonl`, optionally update `lanes/<lane>/notes.md`.
7. `git add lanes/<lane>/` (only your subdir), commit with message format below.
8. The reviewer will ff-merge to main within ~5 min, or write a feedback file.

**Commit message format**:

```
[lane/<name>] <one-line summary> (+N claims, +M sources)
```

Examples:
- `[lane/papers] chunking strategies survey, batch 3 (+12 claims, +5 sources)`
- `[synth] dedupe + recommend hybrid retrieval; +RRF section (+0 claims, refactored 01-approach.md)`

## 4. JSON schemas (HARD, reviewer validates)

### `lanes/<lane>/sources.jsonl`

```json
{
  "id": "<lane-prefix>-S-NNNN",
  "url": "https://...",
  "title": "<exact title>",
  "published": "YYYY-MM-DD or YYYY",
  "type": "paper|repo|blog|post|case-study|talk|docs",
  "authors_or_org": "<authors or org>",
  "key_takeaways": ["...", "..."],
  "relevance_to_our_problem": "high|medium|low",
  "tags": ["chunking", "retrieval", "rerank", "embedding", "generation", "eval", "guardrail", "ingest"],
  "added_at": "YYYY-MM-DD"
}
```

ID prefix per lane: `P-` for papers, `E-` for engineering, `C-` for community. Example: `P-S-0001`.

### `lanes/<lane>/claims.jsonl`

```json
{
  "id": "<lane-prefix>-C-NNNN",
  "topic": "chunking|retrieval|rerank|embedding|index|generation|guardrail|eval|ingest|infra",
  "claim": "<one-sentence factual assertion>",
  "evidence": [
    {"url": "https://...", "quote": "<<=200-char verbatim quote>", "source_id": "P-S-0001"}
  ],
  "counter_evidence": [
    {"url": "https://...", "quote": "<<=200 chars>", "source_id": "P-S-0042"}
  ],
  "confidence": 0.0,
  "applies_to_our_problem": true,
  "applies_when": "<regime/scale/modality if conditional>",
  "last_reviewed": "YYYY-MM-DD"
}
```

Claim ID example: `P-C-0087`. Confidence 0.0-1.0. Single-lane claims max out at 0.6; multi-lane corroboration (set by synth) can push higher.

### `synth/claims.jsonl`

Same shape as lane claims, but:
- `id` uses `S-C-NNNN`
- `evidence` aggregates across lanes (`source_id` may reference any lane's source ID)
- Synth additionally writes `lane_support: ["papers", "engineering"]` listing which lanes corroborate

### `synth/contradictions.jsonl`

```json
{
  "id": "X-NNNN",
  "topic": "...",
  "claim_a": "S-C-NNNN",
  "claim_b": "S-C-MMMM",
  "summary": "<short>",
  "resolution_proposal": "<one of: prefer-A, prefer-B, conditional-on-regime, requires-human>",
  "regime_if_conditional": "...",
  "status": "open|resolved"
}
```

## 5. Image policy (HARD, reviewer rule 7)

- **Default to mermaid** for: architecture, data flow, sequence, state machines, tradeoff matrices (as tables), decision trees.
- **Use `GenerateImage` only when** a richer visual genuinely beats prose:
  - RAPTOR tree illustration
  - ColBERT late-interaction mechanic
  - Embedding-space comparisons
  - Anything that would otherwise take 5+ paragraphs of prose
- Storage: `doc/images/<descriptive-name>.png`. ALL generated images get a regen prompt as an HTML comment IMMEDIATELY beside the markdown reference:
  ```markdown
  ![Caption explaining what reader takes away](images/raptor-tree.png)
  <!-- regen-prompt: A hierarchical tree diagram showing... -->
  ```
- Every generated image needs a one-paragraph caption explaining the takeaway. Reviewer rejects gratuitous images (where mermaid or prose would have sufficed).

## 6. Skill proposal protocol (self-improvement)

Trigger: you've executed the *same* multi-step procedure 3+ times across batches and it's bounded + verifiable.

File: `lanes/<lane>/skill-proposals/<slug>.md` or `synth/skill-proposals/<slug>.md`.

Format:
```markdown
# Proposed skill: <slug>

Observed in: <list of 3+ commit SHAs or batch IDs>
Trigger phrase: <natural-language description of when to invoke>

## Procedure
1. ...
2. ...

## Why a skill (not just inline reasoning)
<one paragraph: what becomes reliable + faster if encoded>
```

The reviewer evaluates each proposal and either:
- Accepts: writes `skills/rag10m-<slug>/SKILL.md` (you'll pick it up via `git rebase main` next batch).
- Rejects: writes `review/skill-decisions/<slug>.md` with rationale.

**Always check `skills/` at batch start.** If new accepted skills are relevant, use them.

## 7. Feedback-response protocol

If the reviewer wrote `review/feedback/<your-lane>-<sha>.md`:

1. Read it FIRST, before any new work.
2. Address each rubric clause violation in a new commit (do not amend). Reference the feedback file: `[lane/papers] address feedback for <sha>: ...`
3. Reviewer evaluates the new commit against the rubric same as any other commit.

Do not argue with feedback in prose. If the feedback seems wrong, file it as an open question in `synth/open-questions.md` (lanes can write to that path as an explicit exception, only for this case). Reviewer reads open questions on its next tick.

## 8. Stop conditions

You stop when ANY of these holds (check at the start of each batch):

- `CONVERGENCE.md` evaluates as converged (the reviewer makes this call and signals via a commit `[shape-a] converged` on `main`).
- Your wall-clock work time exceeds **2 hours** since the session started (check session start time in `git log --reverse --format='%aI' | head -1`).
- You receive a kill signal via a commit on `main` containing the literal string `STOP-ALL-LANES`.
- You've processed 60 in-scope sources (lanes only).

When stopping, write a final note to your `notes.md` with: total batches, total sources processed, total claims written, your subjective assessment of what's still uncovered.

## 9. Forbidden actions

- Editing files outside your allowed paths.
- Force-pushing, rewriting history, deleting commits.
- Pushing to remotes (there are none).
- `git commit --amend` (always make a new commit).
- Citing a URL without a verbatim quote (lanes) or without a `source_id` (synth).
- Inventing arXiv IDs, DOIs, or repo URLs. If you can't verify the URL, do not cite it.
- Writing speculative claims with `confidence > 0.5`. If you didn't read it, you don't cite it.

## 10. Tools available

You have:
- Filesystem read/write (within your allowed paths)
- Shell (for git, jq, basic text processing)
- `WebFetch` / `WebSearch` for crawling (no firecrawl MCP available in this env)
- `Read`, `Grep`, `Glob`, `StrReplace`, `Write` for file ops
- `GenerateImage` (synth only - others must use mermaid)

Source-fetching endpoints to use directly (since the dedicated MCPs aren't installed):
- arXiv API: `https://export.arxiv.org/api/query?search_query=<terms>&start=0&max_results=20`
- arXiv abstract page: `https://arxiv.org/abs/<id>`
- arXiv PDF: `https://arxiv.org/pdf/<id>.pdf` (use WebFetch which extracts text)
- GitHub repo metadata: `https://api.github.com/repos/<owner>/<name>` (rate-limited; spread requests)
- GitHub README: `https://raw.githubusercontent.com/<owner>/<name>/HEAD/README.md`
- Reddit JSON: any reddit URL + `.json` suffix (e.g. `https://www.reddit.com/r/LocalLLaMA/top/.json?t=year`)
- HN: `https://hn.algolia.com/api/v1/search?query=<terms>&tags=story`
