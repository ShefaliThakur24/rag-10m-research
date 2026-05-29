---
name: rag-retrieval-rerank
description: Defaults and diagnostics for the query-time path of a production RAG system over a 10M+ document corpus. The agent reads this skill when designing or debugging the retrieve → rerank → (optional multi-hop) leg, when a teammate reports low recall@K, low nDCG@10, faithfulness regressions, or "the bot retrieves the wrong chunks." Covers hybrid retrieval (BM25 + dense with Reciprocal Rank Fusion), cross-encoder rerank over top-100 → top 5-10, contextual BM25 on the query side, and a multi-hop decision tree that prefers HippoRAG 2 in production with IRCoT as fallback and CoVe for post-hoc verification of longform answers. Each decision is a single default with quantified rationale and an escape hatch, not a menu. Trigger terms: "RAG retrieval", "hybrid retrieval", "BM25 + dense", "cross-encoder rerank", "Cohere Rerank", "multi-hop retrieval", "IRCoT", "HippoRAG", "retrieval failure", "recall@K", "nDCG@10".
---

## When to use this skill

Read this skill when working on the **query-time path** of a production RAG system at 10M+ doc scale: query → retrieve → rerank → (multi-hop loop if needed) → generator. Concretely, use it when:

- designing a new retriever/reranker stack and you need defaults, not options;
- debugging "the bot retrieves the wrong chunks" or "answers cite irrelevant passages";
- triaging a regression where recall@K, nDCG@10, or faithfulness moved;
- deciding whether a query needs multi-hop decomposition.

NOT for:

- ingestion-side index decisions (chunk size, embedding model, vector DB, contextual augmentation at index build time) → use `rag-ingestion-pipeline`.
- generator behavior, citation grammar, refusal calibration, faithfulness gates → use `rag-grounded-generation`.

This skill assumes the ingestion pipeline already produced a Contextual Embeddings index and a Contextual BM25 index per Anthropic's pattern. If it did not, escalate to `rag-ingestion-pipeline` first — query-side fixes cannot recover what the ingestion pipeline failed to encode.

## The query-time pipeline

```
query
  │
  ▼
hybrid retrieve  ─── Contextual BM25 ──┐
(top 100)        ─── dense (HNSW)  ────┤── RRF (k=60) ──► top 100 candidates
  │
  ▼
cross-encoder rerank
(bge-reranker-large / Cohere Rerank 3.5)
  │
  ▼
top 5-10 chunks ──► generator
```

If the single-pass answer fails an entailment check, or the query has ≥ 2 named entities joined by a relational operator, route to the multi-hop decision tree (§ Multi-hop decision tree) before generating. Otherwise stop at single pass.

## The 7 retrieval/rerank decisions

Defaults, not menus. Quantified rationale. Escape hatch only when the default's regime breaks.

| # | Decision | Default | Quantified rationale | Escape hatch |
|---|---|---|---|---|
| 1 | Retrieval mode | **Hybrid (BM25 + dense), Reciprocal Rank Fusion (k=60)** | BM25 alone nDCG@10 ~50; hybrid lifts to ~72 on TREC-DL19 with the LLM-Embedder backbone (FINAL.md §7). Pure dense loses on rare-term and entity-heavy queries. | Pure BM25 only when latency budget < 100ms hard and recall floor tolerates nDCG@10 ~50 (rare). |
| 2 | Top-K from retrieval | **top-100** | Cheap recall pad before rerank. The cross-encoder is the precision step; the retriever's job is to keep recall@100 above the rerank model's ceiling. Smaller K (e.g. 20) starves the reranker; larger K (200+) buys little recall and doubles rerank cost. | Drop to top-50 only if rerank latency dominates p95 and ablation shows recall@100 ≈ recall@50 on your eval. |
| 3 | Reranker | **Cross-encoder over top-100 → top-N=5-10** (bge-reranker-large or Cohere Rerank 3.5) | Adding a reranker on top of Contextual Embeddings + Contextual BM25 cuts top-20 retrieval failure from 2.9% → 1.9% — an additional **-34%** over hybrid alone (Anthropic, FINAL.md §8). On reasoning-heavy queries Cohere Rerank 3.5 hits 81.59% retrieval accuracy vs hybrid 48.80% (~30 absolute points). | Skip rerank only when p95 budget < 300ms hard, or query distribution is overwhelmingly trivial single-hop lookups where hybrid already saturates recall. |
| 4 | Contextual retrieval at query time | **Yes — query-side BM25 must be the Contextual BM25 variant** | Anthropic Contextual Embeddings + Contextual BM25 cuts top-20 retrieval failure 5.7% → 2.9% (-49%) vs plain dense (FINAL.md §7). The query path must hit the contextual indexes; querying a plain BM25 index defeats the ingest-time investment. | None. If the ingest pipeline did not build contextual indexes, fix that first — do not paper over it at query time. |
| 5 | Multi-hop trigger | **Decompose only when** (a) entailment check on the single-pass answer fails, OR (b) query contains ≥ 2 named entities joined by a relational operator (compare/cause/before-after/who-also). Otherwise single pass. | Most production queries are single-hop lookup. Decomposing them adds 4-5× latency for zero accuracy lift. The compositionality bound only bites at ≥ 3 implicit hops; 2-hop bridge queries usually resolve in single pass once hybrid + rerank land the right chunks. | Force decomposition when the query class is known multi-hop (e.g., a "compare X and Y" UI flow). |
| 6 | Multi-hop pattern | **HippoRAG 2 for production**, IRCoT only if you cannot precompute the entity graph, CoVe for post-hoc verification of any longform answer | HippoRAG 2 R@5: MuSiQue 74.7 / 2Wiki 90.4 / HotpotQA 96.3 at **10-30× lower cost and 6-13× lower latency** than IRCoT (v2/generate-multihop.md §2). IRCoT cuts CoT factual errors by ~50% on HotpotQA when graph extraction is infeasible. CoVe lifts MultiSpanQA F1 by 23% on longform claim-list outputs. | Skip multi-hop entirely on single-hop traffic (decision 5). Use IRCoT when corpus turnover > 5%/day makes graph rebuild uneconomic. |
| 7 | Diagnostic playbook | **Decompose by which signal moved** (see § Diagnostic playbook). Recall@K + faithfulness are independent axes; treat them as such. | Misattributing a generator regression to the retriever (or vice versa) burns weeks. The 4-cell signal matrix routes the investigation in O(1). | None. |

## Multi-hop decision tree

Most queries should NOT be decomposed. Decomposition is 4-5× latency and a fresh source of cycle/runaway failures.

```
                    ┌─────────────┐
query ─────────────►│ single-pass │── answer ──► entailment check
                    │ retrieve +  │                 │
                    │ rerank +    │           pass ─┴─► return
                    │ generate    │           fail ──┐
                    └─────────────┘                  │
                                                     ▼
                          ┌────────────────────────────────────┐
                          │ Trigger met? (≥ 2 named entities + │
                          │ relational op, OR entailment fail) │
                          └─────────────┬──────────────────────┘
                                        │ yes
                                        ▼
                          ┌────────────────────────────────────┐
                          │ Entity graph available?            │
                          ├──────── yes ─────► HippoRAG 2       │
                          │                  (PPR over OpenIE   │
                          │                   triples; 1 pass)  │
                          ├──────── no ──────► IRCoT             │
                          │                  (interleave        │
                          │                   retrieve ↔ CoT,   │
                          │                   max 4-5 rounds)   │
                          └────────────────────────────────────┘
                                        │
                                        ▼
                          ┌────────────────────────────────────┐
                          │ Longform / claim-list output?      │
                          ├──────── yes ─────► append CoVe      │
                          │                   (factored verify) │
                          └──────── no ──────► return            │
                          └────────────────────────────────────┘
```

**Hard guardrails on every multi-hop loop** (v2/generate-multihop.md §5):

- iter cap: max 20-25 graph iterations (LangGraph default 25).
- cycle detector: reject any sub-query with cosine-sim > 0.95 to a prior sub-query in the trace.
- tool-call budget: ≤ 8 retrievals + 4 verifications per query.
- escape-hatch refusal: if support score still below threshold after 2 revisions, return "insufficient evidence" + top-k candidates.
- wall-clock SLA: p99 ≤ 12s; on preempt, fall back to single-pass with high-recall retrieval.

## Diagnostic playbook

When something regresses, look at recall@K and faithfulness independently. They factor into a 2×2:

| | **Faithfulness flat / up** | **Faithfulness drops** |
|---|---|---|
| **Recall@K flat / up** | Healthy. Investigate elsewhere (latency, cost, refusal rate). | **Generator regression.** Model version, system message, decoding temperature, citation prompt drift. The retriever is feeding the same chunks; the generator is doing a worse job grounding on them. Check model rollout, prompt diff, temperature, and structured-output schema. |
| **Recall@K drops** | **Retriever regression.** Faithfulness lags because the model still groundedly cites whatever it got — the chunks are just less correct. Check: index build freshness, query-side BM25 (is it still the contextual variant?), embedding model version, RRF weights, top-K cut. | **Both.** Usually a stale index + a model bump landed in the same release. Bisect the deploy; revert one axis at a time. |

Cross-checks before concluding:

- Confirm the query-side BM25 hits the **Contextual BM25** index, not a vanilla BM25 index someone redeployed by accident. This is the single most common silent regression.
- Confirm rerank top-N is unchanged (5-10). Generators degrade fast above N=15 because of the "lost in the middle" effect.
- Confirm τ_sim and τ_rerank thresholds are calibrated **per query class**, not a single global threshold. Per-class calibration captures 2-3× the precision lift of a global cut.

## Anti-patterns

- **Pure dense retrieval as the default.** Loses ~22 nDCG@10 points to hybrid on TREC-DL19 (50 → 72). Dense alone fails on rare-term and entity-heavy queries; hybrid + RRF is strictly better at the cost of a sparse index. Do not ship pure dense.
- **Rerank over the full corpus.** Cross-encoders are O(N) per query; running them over millions of chunks is a cost catastrophe and gains nothing — the retriever's job is to land the recall@100 set first.
- **Multi-hop decomposition on every query.** 4-5× latency and a fresh source of cycle/runaway failures, with zero gain on the single-hop traffic that dominates production. Gate decomposition on the trigger in decision 5.
- **Tuning τ_sim or τ_rerank as a single global threshold.** Different query classes (factual lookup, comparison, longform synthesis) have different similarity distributions; a global cut over- or under-refuses on every class. Calibrate per query class.
- **Rerunning ingestion-side fixes at query time.** "Add more context to the chunks" only works at index build time. If the indexes were not built with contextual augmentation, query-side patches will not recover the -49% retrieval-failure delta.
- **Skipping the entailment check before deciding to decompose.** Cheaper than running multi-hop on every query. Decompose only when the single-pass answer fails entailment or the query syntactically requires it.

## Additional resources

- `references/hybrid-fusion.md` — read when picking a fusion method (RRF vs weighted sum vs cascade), tuning RRF k, or selecting a cross-encoder reranker. Includes a leaderboard table with latency p50, $/1k queries, and nDCG@10 lift over hybrid for bge-reranker-large, Cohere Rerank 3.5, mxbai-rerank-large, and cross-encoder/ms-marco-MiniLM.
- `references/multihop-patterns.md` — read when deciding which multi-hop pattern to ship, debugging a runaway agent loop, or writing a decomposition prompt. Contains a one-row-per-pattern comparison (IRCoT, CoVe, Self-RAG, HippoRAG, HippoRAG 2, ReAct), a canonical decomposition prompt template, and the cycle-detector / iter-cap heuristics.
- Source material in this corpus: `doc/01-approach.md` §5-6 (Retrieve / Rerank), `doc/02-tradeoffs.md` (retrieve / rerank tables), `v2/generate-multihop.md` (multi-hop techniques + guardrails), `v2/ingest-graph.md` (graph regime boundaries), `FINAL.md` §7-8 + §10.
