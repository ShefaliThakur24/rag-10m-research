# Open questions for the human

Synth and reviewer append here. Human answers in place by editing this file. The next synth tick incorporates answers and removes them.

---

## Stage coverage gaps (synth-tick-2, 2026-05-29)

- What's the recommended approach for the **ingest** stage? No claims yet (lanes/engineering and lanes/community are still on first batches).
- What's the recommended approach for the **rerank** stage? No claims yet — `lanes/papers` batch 1 did not cover cross-encoder vs ColBERT vs LLM-rerank tradeoffs at 10M scale.

## Tradeoff regimes flagged `<regime: ???>` (synth-tick-2)

The following rows in `doc/02-tradeoffs.md` have a "wins when" or "loses when" cell without a numeric regime — needs lane evidence:

- chunk / 512-token: loses regime for long-context (≥ 32k) generators.
- embed / LLM-Embedder: loses regime for non-English / domain-specialized retrieval.
- embed / bge-large-en: needs a sharper "wins" regime beyond RAM headroom.
- index / Milvus + IVF-PQ: needs a sharper "loses" corpus-size threshold.
- index / HNSW: needs corroborating evidence on the ~50M corpus break-even.
- retrieve / hybrid: needs a sharper "loses" regime for sub-1s latency budgets.
- retrieve / modular adaptive: needs a workload-heterogeneity threshold.
- generate / decompose+verify: needs a "loses" regime threshold (single-hop overhead cost).
