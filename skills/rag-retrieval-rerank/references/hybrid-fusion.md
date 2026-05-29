# Hybrid fusion: BM25 + dense

Read this when picking a fusion method, tuning its parameters, or selecting a cross-encoder reranker. The defaults in `SKILL.md` (decisions 1, 3) are derived here.

## 1. Reciprocal Rank Fusion (RRF) — the default

For each candidate `d` and each retriever `r ∈ {BM25, dense}` returning rank `rank_r(d)`:

```
score_RRF(d) = Σ_r  1 / (k + rank_r(d))
```

- `k` = 60 in the canonical Cormack et al. 2009 paper. **Use k=60.** It is robust across corpus sizes; ablations in the literature put the sweet spot at 50-70.
- Only ranks matter, not raw scores. RRF dodges the score-calibration problem (BM25 is unbounded, cosine is in [-1, 1]) which is what makes it the production default.
- Symmetric: BM25 and dense contribute equally. If you have evidence one retriever dominates on your eval, drop to weighted sum (next).

## 2. Fusion methods compared

| Method | Formula | Wins when | Loses when | Default? |
|---|---|---|---|---|
| **RRF** (Reciprocal Rank Fusion) | `Σ_r 1/(k + rank_r)` | retrievers' raw scores are uncalibrated; you want a parameter-light default | one retriever is provably stronger and you have an eval to prove it | **Yes** |
| **Weighted sum** | `α·score_dense + (1-α)·norm(score_BM25)` | you have a labeled dev set to tune α; retrievers' scores are min-max normalizable | small/no eval set; α drifts as the corpus grows | No |
| **Cascade** (BM25 → dense rerank, or dense → BM25 filter) | hard pipeline | extreme latency budgets where one stage must be cheap; a known regime boundary (e.g., entity queries → BM25 first) | mixed query workloads; loses recall on whichever stage runs second | Only with regime evidence |
| **CombSUM / CombMNZ** (Fox & Shaw) | sum or sum × #retrievers that hit | classic IR baselines | needs score normalization; outperformed by RRF on most modern dense retrievers | No |

Quantified comparison reported across IR literature:

- RRF vs weighted sum: usually within 1-2 nDCG@10 points either way; RRF wins when α is mis-tuned (common as the corpus drifts).
- Cascade BM25 → dense rerank: ceiling-bounded by BM25 recall@K. Loses on paraphrase-heavy queries where BM25 misses the candidate entirely.
- CombSUM: requires per-retriever score normalization; sensitive to outliers. RRF strictly dominates in modern hybrid stacks.

**Practitioner default:** RRF with k=60. Move to weighted sum only with an eval set ≥ 200 labeled queries.

## 3. Cross-encoder reranker leaderboard

All numbers below are nDCG@10 (or BEIR-class retrieval accuracy) on top of a hybrid (BM25 + dense, RRF) baseline returning top-100. "$ / 1k queries" assumes 100 (query, candidate) pairs per query and the vendor's published unit cost or a measured $/GPU-hour at typical throughput. Latency p50 is for a single (query, 100-candidate) batch on a single A10/L4-class GPU or the vendor's hosted endpoint.

| Reranker | Latency p50 (top-100) | $ / 1k queries | nDCG@10 lift over hybrid baseline | Notes |
|---|---|---|---|---|
| **bge-reranker-large** (BAAI) | ~120-180 ms (A10, fp16, 100 pairs) | ~$0.10 (self-hosted, $1/GPU-hr) | +6 to +9 pts on BEIR avg, +12 on reasoning-heavy slices | Open weights; sensible default for self-hosted stacks. Model card: BAAI/bge-reranker-large. |
| **Cohere Rerank 3.5** | ~80-120 ms (hosted) | ~$2.00 (Cohere posted price) | **+30 abs pts on reasoning-heavy queries** (81.59% vs hybrid 48.80%); +10 pts nDCG@10 multilingual avg across 18 languages (62.18 vs 52.10) | Hosted; best out-of-the-box on reasoning + multilingual. (FINAL.md §8.) |
| **mxbai-rerank-large-v1** (Mixedbread) | ~150-200 ms (A10) | ~$0.12 (self-hosted) | ~parity with bge-reranker-large on BEIR; slightly stronger on long-form passages | Open weights; good when chunk length > 1k tokens. |
| **cross-encoder/ms-marco-MiniLM-L-12-v2** | ~25-40 ms (A10) | ~$0.02 (self-hosted) | +3 to +5 pts on BEIR avg | Cheapest cross-encoder. Use when latency budget is tight (~50ms rerank) and rerank-as-precision-step is "good enough." Underperforms bge-reranker-large by ~3-5 pts on reasoning. |
| **Voyage rerank-2** | ~80-130 ms (hosted) | ~$0.30-1.20 (Voyage posted; cheaper than Cohere) | competitive with Cohere Rerank 3.5 on multilingual; lighter on reasoning queries | Hosted; pick over Cohere when cost matters more than the reasoning ceiling. |

**Picking among them:**

- **Default for self-hosted:** bge-reranker-large. Open weights, ~$0.10 per 1k queries, +6-9 nDCG@10 over hybrid.
- **Default for hosted, reasoning-heavy queries:** Cohere Rerank 3.5. The +30 abs-points delta on reasoning data is the largest moat any single component buys in the query path.
- **Latency-bound (<50ms rerank budget):** cross-encoder/ms-marco-MiniLM. Accept the smaller lift.
- **Long-passage corpora (chunk length > 1k tokens):** mxbai-rerank-large-v1.

In all cases, **rerank top-100 → top-N where N ∈ [5, 10]**. Above N=15, "lost in the middle" effects in the generator dominate any further recall gain.

## 4. Things that look like fusion knobs but aren't

- **BM25 b and k1.** Tune these on the BM25 retriever itself, not the fusion layer. Lucene defaults (b=0.75, k1=1.2) are fine for English open-domain; bump k1 to 1.5-2.0 for long technical documents.
- **Dense embedding dimensionality.** A retrieval-side concern. Match the model the index was built with; do not project at query time.
- **RRF k vs reranker top-N.** Independent. RRF k=60 controls fusion; rerank top-N controls how much context the generator gets.

## 5. References

- Cormack, Clarke, Büttcher 2009 — "Reciprocal Rank Fusion outperforms Condorcet and individual rank learning methods" (the RRF paper).
- Anthropic — "Introducing Contextual Retrieval" (5.7% → 2.9% → 1.9% retrieval-failure ladder).
- FINAL.md §7-8 in this corpus — Hybrid + Rerank quantified rationale.
- Cohere Rerank 3.5 announcement — reasoning + multilingual numbers.
