# Lane: papers

## Scope

You read **academic and benchmark papers only**: arXiv, ACL/EMNLP/NeurIPS/ICLR/SIGIR proceedings, Google Scholar leads from those, and eval benchmark papers (FActScore, RAGAS, etc.).

## Hard write rule

You write ONLY to `lanes/papers/**`. You commit ONLY on branch `lane/papers`.

## Acceptance bar (reviewer enforces)

- Source MUST be peer-reviewed OR arXiv with a valid arXiv ID.
- Claims MUST cite specific figures, tables, or sections (e.g. "Table 3 in arxiv.org/abs/2309.15217 reports recall@10 of 0.84 ... for ColBERT"), not just the paper title.
- Every claim quote MUST be verbatim from the paper (PDF or abstract page). The reviewer will randomly fetch+grep 3 claims per batch; one fabrication rejects the batch.
- Out-of-scope items (great-looking GitHub repo, fascinating Reddit thread) -> record as a one-liner in `lanes/papers/notes.md` under `## Out-of-scope leads` and continue. The synth/other lanes will see it.

## Tools

- `WebFetch` for arxiv abstract pages (`https://arxiv.org/abs/<id>`) and PDFs (`https://arxiv.org/pdf/<id>.pdf`)
- arXiv HTTP API: `https://export.arxiv.org/api/query?search_query=<terms>&start=0&max_results=20`
- `WebSearch` for discovery
- No source-fetching MCPs installed in this env. Built-ins only.

## Batch loop

Per batch (~10 papers):

1. `cd /Users/shefalithakur/cursor-exp/rag-10m-research-papers` (your worktree, branch `lane/papers`)
2. `git fetch && git rebase main` (pick up new skills + shared brief edits; should be conflict-free)
3. Read `AGENT_BRIEF_SHARED.md`, this file, and `skills/` for any new accepted skills.
4. Check `review/feedback/papers-*.md` for unaddressed feedback. Address it in a new commit before doing anything else.
5. Pick 5-10 unread papers (from `sources/seed.yaml#papers` or leads in your `notes.md`).
6. For each paper:
   - Fetch abstract page (and PDF body when you need specific numbers).
   - Append 1 source row to `lanes/papers/sources.jsonl` (id format: `P-S-NNNN`).
   - Append 1-5 claim rows to `lanes/papers/claims.jsonl` (id format: `P-C-NNNN`). Each claim: verbatim quote, exact URL with anchor when possible, cite figure/table/section.
   - Tag claims by `topic` (one of: ingest, chunk, embed, index, retrieve, rerank, generation, guardrail, eval).
7. `git add lanes/papers/` (your subdir only)
8. `git commit -m "[lane/papers] <summary> (+N claims, +M sources)"` (no `git push`; reviewer reads local branches)

## Stop conditions (any)

- 2-hour session wall-clock elapsed (check `git log --reverse --format='%aI' main | head -1`)
- 60 papers processed
- `[shape-a] converged` commit appears on main
- `STOP-ALL-LANES` string appears in any commit message on main

## Initial focus order

Within your 2-hour budget, prioritize:

1. Survey/best-practices papers (arXiv 2312.10997, 2407.01219, 2403.05676) - they cross-reference everything else.
2. Faithfulness/eval (FActScore 2305.18654, RAGAS 2309.15217, Chain-of-Verification 2311.09210, Lost in the Middle 2307.03172) - direct hits on "near-zero hallucination".
3. Retrieval mechanics (ColBERT 2112.09118, Query Rewriting 2305.14283, hybrid retrieval threads).
4. Architectures (RAPTOR 2401.18059, GraphRAG 2404.16130, Self-RAG 2310.11511, HippoRAG 2403.10131).
5. Long-context vs RAG (2406.14739) - relevant for the "10M docs vs huge context window" tradeoff.

## Self-improvement (skill proposals)

If you find yourself running the same multi-step procedure 3+ times (e.g. "extract recall@k numbers from arXiv tables", "deduplicate claims by topic similarity"), draft a proposal under `lanes/papers/skill-proposals/<slug>.md` per the format in `AGENT_BRIEF_SHARED.md` section 6.
