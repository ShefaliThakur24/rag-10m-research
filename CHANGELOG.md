# Changelog

One line per synth tick. Format: `[<ISO timestamp>] <synth-sha> +N claims, +M sources, +K contradictions, doc sections touched: <list>`

---

(empty - synth appends here)

[2026-05-29T11:00:42Z] synth-tick-1 no-op (main unchanged, awaiting reviewer); not converged
[2026-05-29T11:57:12Z] synth-tick-2: +13 claims to synth/, +5 sources; doc sections populated: chunk, embed, index, retrieve, generate, hallucination-control; appendix promoted: ragas; appendix new stubs: bm25, cove, hybrid-retrieval, hyde, modular-rag, piperag, self-rag; open-questions: 2 stage gaps (ingest, rerank) + 8 regime-???

## synth tick 3 — 2026-05-29 ~+78min

**Inputs:** engineering batch 1 (8 claims, 4 sources), community batch 1 (5 claims, 5 sources). All manually sourced by chat-agent under degraded-subagent fallback path.

**Doc updates:**
- doc/01-approach.md §2 Chunk: added contextual chunking augmentation (E-C-0001), confidence bumped low→medium.
- doc/01-approach.md §5 Retrieve: added Anthropic hybrid+context numbers (E-C-0002) as second independent corroboration of hybrid direction; confidence bumped low→medium.
- doc/01-approach.md §6 Rerank: filled (was _TBD_). Cross-encoder over top-100, citing E-C-0003/0004/0005/0008.
- doc/01-approach.md §7 Generate: cite-or-refuse contract added (C-C-0005), now a system-level rather than generator-level recommendation.
- doc/02-tradeoffs.md §rerank: filled (was _TBD_), 4 rows with explicit regimes.
- doc/02-tradeoffs.md §generate: +2 rows (cite-or-refuse, permissive+filter).
- appendix/concepts.md: contextual-retrieval entry promoted from stub with numbers; +cite-or-refuse, +retriever-eval-antipattern, +eval-driven-development, +production-derived-eval.

**Coverage:** doc/01 sections filled = 6/7 (only §1 Ingest remains _TBD_). Tradeoff tables: ingest still _TBD_, all others have ≥3 rows with regimes.

**Open question (carries):** ingest stage has no claims — document parsing strategy (Unstructured / vendor APIs / custom OCR for 10M+ heterogeneous docs) was deferred by every lane.
