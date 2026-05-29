# Tradeoff matrices

One section per pipeline stage. Each section: a table of candidate approaches, the regime where each wins, and citations to supporting claims in `synth/claims.jsonl`. Synth writes; reviewer enforces tradeoff specificity (no hedge words without numeric regimes).

## ingest

| Approach | Wins when | Loses when | Supporting claims | Confidence |
|---|---|---|---|---|
| _TBD_ | _TBD_ | _TBD_ | _TBD_ | _TBD_ |

(No claims in this batch — see [synth/open-questions.md](../synth/open-questions.md).)

## chunk

| Approach | Wins when | Loses when | Supporting claims | Confidence |
|---|---|---|---|---|
| 512-token sentence-level, 20-token overlap | doc-QA, ada-002 + GPT-3.5/Zephyr-7B class generators, faithfulness floor ≥ 95 (measured 97.59/97.41 on lyft_2021) | long-context generators (≥ 32k) where the stitching cost of small chunks dominates — `<regime: ???>` | S-C-0003 | low |
| 1024-token sentence-level | latency-bound batching where the 3.3-point faithfulness drop (94.26 vs 97.59) is acceptable | faithfulness floor > 95 | S-C-0003 | low |
| 2048-token | tasks gated only on recall, not faithfulness | any task gating on faithfulness ≥ 90 (drops to 80.37 on lyft_2021) | S-C-0003 | low |

## embed

| Approach | Wins when | Loses when | Supporting claims | Confidence |
|---|---|---|---|---|
| BAAI LLM-Embedder | English open-domain retrieval, 10M+ corpus where embedding RAM matters (3x smaller than bge-large-en, MRR@10 37.58 vs 37.66 on msmarco) | non-English, code/finance/law-specialized retrieval — `<regime: ???>` | S-C-0004 | low |
| BAAI bge-large-en | RAM headroom available and a ~0.08 MRR@10 / ~0.36 R@10 gain matters | embedding-store cost dominates infra budget | S-C-0004 | low |

## index

| Approach | Wins when | Loses when | Supporting claims | Confidence |
|---|---|---|---|---|
| Milvus + IVF / IVF-PQ, nprobe under generation-latency budget | corpus ≥ 10M with billion-scale headroom, hybrid (dense + sparse) required, cloud-native deployment | sub-10M corpora where simpler in-memory HNSW fits — `<regime: ???>` | S-C-0006, S-C-0008 | low |
| HNSW in-memory (Faiss / Qdrant / Weaviate) | corpus ≤ ~50M and RAM fits, p95 latency target < 50ms, no billion-scale roadmap | corpus > 50M or RAM-constrained — `<regime: ???>` | S-C-0008 | low |
| Fixed-nprobe IVF without latency hiding | offline batch retrieval where p95 isn't bounded | online RAG with a generation-latency budget (PipeRAG-style sizing wins) | S-C-0008 | low |

## retrieve

| Approach | Wins when | Loses when | Supporting claims | Confidence |
|---|---|---|---|---|
| Hybrid (BM25 + dense) | latency budget ≥ 3.2s/query and recall floor on TREC-DL-style nDCG@10 ≥ 72 | latency budget < 1s/query — `<regime: ???>` | S-C-0005 | low |
| Hybrid + HyDE | latency budget ≥ ~12s/query OR HyDE can be precomputed/cached/async; nDCG@10 target ≥ 73 | online interactive paths with latency < 5s/query | S-C-0005 | low |
| BM25 only | latency < 0.1s/query and recall floor tolerates nDCG@10 ~ 50 | any faithfulness-gated workload | S-C-0005 | low |
| Modular adaptive (Search / Routing / Predict modules) | heterogeneous query types where single-pass misses dominate retrieval failures | uniform query workload where a fixed hybrid pipeline suffices — `<regime: ???>` | S-C-0001, S-C-0002 | low |
| Periodic retrieval with PipeRAG-style pipelining | Retro-style cross-attention architectures, long generations where retrieval can hide under inference (2.6x speedup at iso-perplexity on C4 200B-token DB) | one-shot RAG over chat LLMs | S-C-0007 | low |

## rerank

| Approach | Wins when | Loses when | Supporting claims | Confidence |
|---|---|---|---|---|
| Cross-encoder rerank over top-100 → top-N (Cohere Rerank 3.5 / BGE-reranker-v2 / Voyage rerank-2) | hybrid retrieval bottlenecked on noise; reasoning-heavy queries (Cohere Rerank 3.5: 81.59% vs Hybrid 48.80%); multilingual (62.18% vs 52.10% nDCG@10); latency budget tolerates +100-500ms | latency budget < ~300ms hard; trivial single-hop lookups where hybrid already saturates recall | E-C-0003, E-C-0004, E-C-0005, E-C-0008 | medium |
| Rerank-as-LLM-judge (pairwise / listwise LLM scoring) | top-K is small (< 20) and per-pair LLM cost is amortizable; need explicit rationale per ranking decision | top-100 candidates × per-pair LLM calls blows latency and cost budget | (gap — see [synth/open-questions.md](../synth/open-questions.md)) | low |
| No reranker, hybrid only | latency budget < 200ms hard or rerank service is unavailable | recall floor on top-20 chunks > 95% required (hybrid alone leaves ~2.9% failure rate vs reranked 1.9% per E-C-0003) | E-C-0003 | medium |
| Bigger top-K to generator instead of rerank | generator has long context and tolerates noise; rerank service unavailable | "lost in the middle" effect dominates; noise crowds out relevant context and increases hallucination risk | C-C-0002 | medium |

## generate

| Approach | Wins when | Loses when | Supporting claims | Confidence |
|---|---|---|---|---|
| Single-prompt generation (generator chains facts internally) | single-hop / direct-lookup QA where retrieved chunks already contain the answer | multi-hop chains > ~3 hops or any numeric/logical composition (GPT-4 at 59% on 3x3 multiply; OOD depth fails near-zero) | S-C-0009, S-C-0010 | medium |
| Explicit decomposition + per-step verification (Self-RAG / CoVe / external solver) | multi-hop chains ≥ 3 hops, numeric/logical compositions, faithfulness target ≥ 0.95 | single-hop where decomposition overhead isn't repaid; latency budget too tight for verify loops | S-C-0010 | medium |
| Hard cite-or-refuse contract at validation layer | near-zero hallucination is a product requirement; chat UI can present refusal gracefully | best-effort search UIs where refusal is worse than a partial answer; high-stakes "must answer" workflows | C-C-0005 | medium |
| Permissive generation, eval-only filter (LLM-judge faithfulness post-hoc) | offline batch generation where bad answers can be hand-filtered | online interactive RAG where bad answers reach users before the judge flags them | C-C-0005 | low |
