---
name: rag-ingestion-pipeline
description: Picks defaults for the offline indexing path of a production RAG system at 100k-100M document scale. Names ONE recommended choice per decision (chunk size, contextual augmentation, embedding model, dimensionality, vector index, hard-negatives, graph add-on) with the quantified rationale from primary sources (Anthropic Contextual Retrieval, NVIDIA TopK-PercPos, MSR GraphRAG, MTEB). Use when the user is designing or debugging RAG ingestion, asking about "chunking strategy", "contextual retrieval", "embedding model selection", "vector index" sizing, "HNSW vs IVF-PQ", "BM25 + dense" hybrid, "Anthropic contextual embeddings", "hard negatives" mining, "10M documents" scale, or "production RAG indexing". NOT for query-time retrieval, reranking, or generation logic — see rag-retrieval-rerank, rag-grounded-generation, and rag-security-guardrails. Defaults are calibrated for 10M docs / English-primary / faithfulness-gated workloads; escape hatches name the regime where each default flips.
---

## When to use this skill

Read this skill when the user is designing or debugging the **offline indexing path** of a RAG system:

- Picking a chunk size or boundary policy.
- Deciding whether to add Anthropic-style contextual chunk augmentation.
- Selecting an embedding model and dimensionality at 10k-100M doc scale.
- Choosing an ANN index (HNSW vs IVF-PQ) and its build parameters.
- Deciding whether fine-tuning embeddings (and mining hard negatives) is worth the spend.
- Deciding whether to layer a knowledge graph on top of vector retrieval.

**NOT for** query-time retrieval logic, hybrid fusion weighting, reranker selection, or generator/guardrail design. Those belong to `rag-retrieval-rerank`, `rag-grounded-generation`, and `rag-security-guardrails` (sibling skills).

The skill names ONE default per decision with a numeric rationale. Escape hatches are named explicitly; absent the escape-hatch regime, the default wins.

## The 7 ingestion decisions

| # | Decision | Default | Quantified rationale | Escape hatch |
|---|---|---|---|---|
| 1 | **Chunking** | Token-aware splits at **~512 tokens** with **10-15% overlap (≈50-80 tokens)** snapped to semantic boundaries (paragraph / heading / sentence). | On `lyft_2021` doc-QA with ada-002, 512-token chunks score faithfulness **97.59 / relevancy 97.41**; 1024-token drops to 94.26 / 95.56; 2048-token collapses to **80.37 / 91.11** (S-C-0003 in `doc/02-tradeoffs.md`). 512 is the only size that holds faithfulness ≥ 95 on a faithfulness-gated workload. | Recursive character splits for code / log / non-prose content (no sentence boundaries to snap to). Long-context generators (≥32k) tolerate larger chunks if the stitching cost dominates. |
| 2 | **Contextual chunk augmentation** | **Enable.** Prepend a 50-100 token LLM-generated doc-level context per chunk before both dense embedding and BM25 indexing. Cache the document once per Anthropic prompt-caching; one LLM call per chunk amortizes to ~$0.001/chunk on `claude-3-haiku` class models. | Anthropic's internal eval: contextual embeddings cut top-20 retrieval failure **5.7% → 3.7% (-35%)**; Contextual Embeddings + Contextual BM25 together cut it **5.7% → 2.9% (-49%)**; layering rerank reaches 1.9% (-67%). One of the largest single-knob deltas in the whole pipeline. | Skip when chunks are already long and self-contained (e.g., full SEC filings as one chunk) — document-level context is redundant. Also skip if you cannot afford one LLM call per chunk at index time. |
| 3 | **Embedding model** | **`bge-base-en-v1.5`** (768 dim, MIT) for English ≤ ~100k docs; **`bge-large-en-v1.5`** (1024 dim, MIT) at 100k-10M; domain-specific fine-tune above 10M or when base Recall@10 < 0.70. | BGE-large-en-v1.5 hits **MTEB 63.6** at 1024 dim, self-host, MIT-licensed. BAAI LLM-Embedder lands MRR@10 37.58 / R@10 66.45 on msmarco vs bge-large 37.66 / 66.09 — a wash at ~1/3 parameter count (S-C-0004). Embedding is **not** the dominant cost at 10M docs (~$100-$1k one-shot; ~$370/mo RAM) — pick the model that maximizes downstream Recall@10. | OpenAI `text-embedding-3-large` (Matryoshka-truncated to 1024) if cost is irrelevant and the +1-3 pt MTEB margin matters. Multilingual: **`bge-m3`** (1024 dim, dense+sparse+multi-vector, hybrid NDCG@10 0.58-0.62 on MIRACL). Domain (legal / finance / code): Voyage specialist (e.g., `voyage-code-3` is +13.80% NDCG vs OpenAI 3-large on 32 code datasets). |
| 4 | **Dimensionality** | **768** (bge-base) or **1024** (bge-large / bge-m3 / 3-large MRL). Quantize to **int8 scalar** (4× compression, ~97% recall vs fp32) as the production HNSW default. | A doubling of D doubles RAM and roughly doubles HNSW query latency at the same `efSearch`. 50M vectors × 1024 dim × fp32 ≈ 205 GB raw, ~266 GB with HNSW graph; int8 SQ collapses that to ~50 GB resident on one `r6i.2xlarge`. Microsoft's Azure SQL benchmark: `text-embedding-3-large` at **256 dims still beats `ada-002` at 1536** on MTEB retrieval — a 12× storage cut for *better* quality. | Matryoshka-truncate to **256 dims** for retrieval-only indexes when RAM is the binding constraint (Voyage-3-large, OpenAI v3, bge-m3 all support truncation natively). Use 1536+ only after a measured ≥3-pt Recall@10 win on your own eval set. |
| 5 | **Vector index** | **HNSW**, `M=16`, `efConstruction=200`, `efSearch=50` for **≤10M vectors**. Switch to **IVF-PQ** above ~50M vectors when RAM becomes the binding constraint. | M=16 / efC=200 is the widely-replicated HNSW recall/latency sweet spot (Faiss / Qdrant / Weaviate / Milvus default within ±a notch). `efSearch=50` typically lands Recall@10 ≥ 0.95 at p95 < 30ms on commodity hardware. Above ~50M vectors the index no longer fits in a single node's RAM; IVF-PQ collapses 1024-dim vectors 32-64× at the cost of 90-95% recall (recoverable with rerank). | Disk-backed ANN (DiskANN / Vamana) when even IVF-PQ blows RAM (1B+ vectors), accepting a 2-10× latency hit. Milvus is the production default when billion-scale + hybrid + cloud-native are all required; for ≤10M corpora, plain HNSW in Qdrant / Weaviate / Faiss is simpler. |
| 6 | **Hard-negatives mining** | **Enable** BM25-mined hard negatives with **positive-aware filtering** (cap negative-relevance at 95% of the positive score) for fine-tuning above ~100k labelled or synthesized pairs. Train with MultipleNegativesRankingLoss (in-batch negatives = N-1 free) + the mined hard negatives. | NVIDIA's TopK-PercPos paper: positive-aware filtering is the single biggest lever in the mining stack, lifting **average NDCG@10 to 60.55** — best-in-class among open mining methods. Synthetic-pair pipeline (prompt LLM with "Generate 5 questions this passage uniquely answers") yields **+10-11% Recall@10 / NDCG@10** in <1 day on a single A100. Atlassian replicated it on JIRA: **Recall@60 0.751 → 0.951 (+26%)**. Total cost typically <$100 end-to-end. | Skip when base Recall@10 is already ≥0.85 on your eval set (no headroom), when you have <10k pairs (overfit risk), or when the domain is well-covered by web text (general English). |
| 7 | **Graph / entity-resolution add-on** | **Skip.** Vector + hybrid + rerank covers the lookup and 2-hop QA regime. Ship the graph layer only if ≥30-50% of production queries are multi-hop / aggregation / global-sensemaking, OR if compliance / lineage requires provenance graphs. | Indexing cost overhead is **10-5000×** embedding-only ($1-5k for embed → $50k-$25M for full MSR GraphRAG extraction at 10M docs). Wins are regime-specific: HippoRAG +20 pts R@5 on 2WikiMultiHopQA, MSR GraphRAG 72-83% comprehensiveness win on global sensemaking. The independent eval in arXiv:2502.11371 finds GraphRAG-Global *under-performs* vanilla RAG on symmetric LLM-as-judge — treat single-vendor win-rate plots with caution. | Ship when query mix is ≥30% multi-hop / aggregation / "summarize across N docs" AND ROI pencils out (LinkedIn's KG-RAG cut median issue resolution time **-28.6%** A/B over 6 months). Use LightRAG- or HippoRAG-style ingestion (REBEL/GLiNER for triples + small LLM for coreference + GPT-4-class only for top-5% community summaries) — not MSR GraphRAG's 4-LLM-call-per-chunk pipeline. |

## Standard pipeline (pseudocode)

```python
for doc in corpus:                                     # 1. parse
    text = parse(doc)                                  #    (PDF/HTML/MD — see references/chunking-recipes.md)
    chunks = token_aware_split(text, size=512,         # 2. chunk
                               overlap=64,
                               boundary="paragraph|heading|sentence")
    doc_ctx_prompt = cache_document(text)              # 3. contextualize (Anthropic)
    for chunk in chunks:
        chunk.context = llm.contextualize(             #    50-100 token doc-level context
            doc=doc_ctx_prompt, chunk=chunk.text)      #    prompt-cached: ~$0.001/chunk
        chunk.augmented = chunk.context + "\n" + chunk.text
        chunk.vec = embed(chunk.augmented,             # 4. embed
                          model="bge-large-en-v1.5",   #    1024 dim, int8 SQ
                          quantize="int8")
        index.add(chunk.vec, chunk.augmented)          # 5. index: HNSW M=16, efC=200
        bm25.add(chunk.augmented)                      #    also index for Contextual BM25
```

Five steps, one default per step. Everything else (graph extraction, fine-tune, re-embed cadence) is bolted onto this skeleton; see the references for when.

## Anti-patterns

- **Fixed-size character chunks that ignore boundaries.** Splits sentences mid-clause, kills BM25 lexical signal, and degrades dense recall. Token-aware splits snapped to paragraph/heading boundaries cost nothing extra and recover the lost recall.
- **Embedding raw chunks without contextual augmentation.** Leaves the 35-49% Anthropic retrieval-failure cut on the table. The cost is one LLM call per chunk; the cost of *not* doing it is paid on every query forever.
- **Relying on cosine similarity alone without rerank.** Hybrid (BM25 + dense) lifts nDCG@10 from ~50 (BM25 alone) to ~72 on TREC DL19; rerank on top adds another ~34% retrieval-failure cut. Skipping rerank to "save 100ms" is the single most common false economy.
- **Choosing dimensionality by vibes.** Going to 3072 dim because "bigger is better" inflates RAM 3× and latency 2× for typically <2 pt MTEB. Always measure Recall@10 on your eval set before increasing D.
- **Premature 4096-dim open models (NV-Embed, Qwen3-Embedding-8B).** 4× index inflation almost never beats hybrid + rerank on the same baseline. Earn the upgrade with a measured ≥3-pt Recall@10 lift.
- **Building HNSW after creating payload/metadata filters.** Qdrant: payload indexes MUST be created **before** ingestion to benefit from filter-aware HNSW edges; post-hoc payload indexes require full graph rebuild.
- **Shipping MSR GraphRAG on a high-churn corpus.** Community structure must be fully rebuilt per refresh (~2× original cost). For >5%-daily-churn corpora use LightRAG or HippoRAG, or skip the graph layer.
- **Re-embedding without budgeting the index rebuild.** "How much to change models?" is the real bill — 2-4× per year. Plan $600-$1k per re-embed on hosted APIs or $50-$300 self-hosted, **plus** the rebuild wall-clock (often dominant).
- **Single-shard HNSW above ~300-400 GB resident.** Above the per-node RAM ceiling, you're sharding (which inflates p99 from tail-latency fanout) or moving to disk-backed (2-10× latency hit). Plan the IVF-PQ / disk-backed transition before you hit the wall.

## Additional resources

- **Read `references/chunking-recipes.md` when** you're (a) processing a non-prose source (code, logs, HTML, PDF, markdown), (b) choosing between sentence-window / recursive / token-aware splitters, or (c) wiring up the Anthropic contextual-chunk prompt verbatim with prompt caching.
- **Read `references/embedding-selection.md` when** you need (a) the model leaderboard table (dim / MTEB / $/1M tok / latency / language coverage), (b) the English / multilingual / domain / cost-no-object decision tree, or (c) the fine-tuning trigger heuristic (when ≥5 pp Recall@10 lift is reachable with ≥10k labelled pairs).
