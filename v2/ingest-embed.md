# Vector Embedding for 10M+ Document Production RAG

Deep dive on embedding model selection, dimensionality economics, multilingual / domain coverage, fine-tuning ROI, and a concrete cost model at 10M-document scale. Assumes the chunking and hybrid-retrieval defaults from `EXPORT.md` are already settled; this section focuses on what to embed *with* and what it costs.

## 1. Embedding model selection at 10M+ scale

Numbers below are headline MTEB averages (English) unless noted; treat ±1 point as noise across leaderboard refreshes and tokenizer-level differences. Prices are list, per 1M input tokens.

| Model | Provider / license | Dim (native) | Max tokens | MTEB avg | $/1M tok | Notes / source |
|---|---|---|---|---|---|---|
| NV-Embed-v2 | NVIDIA, open-weight (CC-BY-NC 4.0) | 4096 | 32768 | **72.31** (legacy MTEB 56-task) | self-host | SOTA open-weight English; not commercial-friendly license [awesomeagents.ai/leaderboards/embedding-model-leaderboard-mteb-april-2026] |
| BGE-en-ICL | BAAI, open (MIT) | 4096 | 32768 | 71.24 | self-host | In-context learning variant; few-shot lifts task-specific quality [awesomeagents.ai] |
| Voyage-3-large | Voyage AI, closed | 2048 (256/512/1024 MRL) | 32000 | 66.80 (retrieval ~74 on Voyage's eval) | $0.06–0.18 | Strong on technical + code; Matryoshka + int8/binary native [blog.voyageai.com/2025/01/07/voyage-3-large] |
| OpenAI text-embedding-3-large | OpenAI, closed | 3072 (truncatable to 256) | 8191 | 64.6 | $0.13 ($0.065 Batch) | Matryoshka via `dimensions` param; broadest ecosystem [respan.ai/articles/openai-embeddings-guide] |
| Cohere embed-v4.0 (multilingual) | Cohere, closed | 1536 (truncatable to 256) | 128000 | ~65.2 | $0.10–0.12 | 100+ languages; multimodal (text+image); long context [teachmeidea.com/openai-voyage-cohere-embeddings] |
| BGE-large-en-v1.5 | BAAI, open (MIT) | 1024 | 512 | 63.6 | self-host | Workhorse open baseline; cheap to serve on a single L4/A10 [pecollective.com/tools/text-embedding-models-compared] |
| BGE-M3 | BAAI, open (MIT) | 1024 | 8192 | 63.0 (dense); ~0.58–0.62 NDCG hybrid | self-host | Dense+sparse+multi-vector in one pass; best open multilingual [bge-model.com/bge/bge_m3.html] |
| OpenAI text-embedding-3-small | OpenAI, closed | 1536 (truncatable to 256) | 8191 | 62.26 | $0.02 ($0.01 Batch) | 6.5× cheaper than 3-large for ~4 pt MTEB delta [tokenmix.ai/blog/openai-embedding-pricing] |
| BAAI LLM-Embedder | BAAI, open (MIT) | 768 | 512 | ~60 (general MTEB; tuned for LLM-augmented retrieval) | self-host | The doc's default; instruction-aware, small footprint [huggingface.co/BAAI/llm-embedder] |

**Staff-engineer read.** At 10M docs the real picks collapse to four:

1. **Self-hosted BGE-M3** when you need multilingual + hybrid + no per-call cost (and you have a GPU budget).
2. **Voyage-3-large or 3.5** when retrieval quality is the conversion-rate metric and you can afford $0.06/M.
3. **OpenAI 3-large @ 1024 dims (Matryoshka)** when you already pay OpenAI and want one fewer vendor.
4. **OpenAI 3-small or BGE-large-en-v1.5** as the cost-floor baseline.

Avoid premature jumps to 4096-dim open models (NV-Embed, Qwen3-8B) unless you've measured a ≥3-point Recall@10 lift on your own eval set — at 10M chunks, the 4× index inflation dominates the win.

## 2. Dimensionality tradeoff at scale

Storage math, fp32, assuming 10M docs × **5 chunks/doc = 50M vectors** (the EXPORT.md chunking default). Add ~25–35% on top for HNSW graph overhead, payload, and replication.

| Dim (D) | Bytes/vec (fp32) | Raw vector bytes for 50M | + HNSW (~30%) | Notes |
|---|---|---|---|---|
| 384 | 1,536 | **76.8 GB** | ~100 GB | bge-small-en, voyage-3-lite truncated |
| 768 | 3,072 | **153.6 GB** | ~200 GB | LLM-Embedder, Nomic, bge-base |
| 1024 | 4,096 | **204.8 GB** | ~266 GB | BGE-large, BGE-M3, Cohere v4 truncated, OpenAI-3-large MRL "sweet spot" |
| 1536 | 6,144 | **307.2 GB** | ~400 GB | OpenAI 3-small native, Cohere v4 native |
| 3072 | 12,288 | **614.4 GB** | ~800 GB | OpenAI 3-large native |
| 4096 | 16,384 | **819.2 GB** | ~1.07 TB | NV-Embed-v2, Qwen3-Embedding-8B |

A doubling of D doubles RAM and roughly doubles HNSW query latency for the same `efSearch`. At 10M docs the practical ceiling per HNSW node is ~300–400 GB resident; above that you're sharding (which inflates p99 from tail-latency fanout) or moving to disk-backed (DiskANN, Vamana) with a 2–10× latency hit.

**Matryoshka shortcut.** OpenAI v3 and Voyage 3/3-large/3.5 support truncation via the `dimensions` parameter, and Microsoft's Azure SQL benchmark shows `text-embedding-3-large` at 256 dims **still beats `ada-002` at 1536** on MTEB retrieval — a 12× storage cut for *better* quality [devblogs.microsoft.com/azure-sql/embedding-models-and-dimensions]. The empirically-observed sweet spot for 3-large is **1024 dims** (≈99% of 3072 quality at 1/3 the bytes).

**Quantization stack** (recall numbers from Tacnode + Qdrant benchmarks):

| Method | Compression | Recall vs fp32 | When to use |
|---|---|---|---|
| fp16 scalar | 2× | ~99% | Always-on; near-free |
| int8 scalar (SQ) | 4× | ~97% | Default for production HNSW [tacnode.io/post/vector-quantization-explained] |
| Product quantization (M=96, k=256) | 32–64× | 90–95% | 1B+ vectors, RAM is the constraint [krunalkanojiya.com/blog/product-quantization-explained] |
| Binary (1-bit) + rescoring | 32× | 85–95% with rerank, collapses below ~256 dim | Voyage/Cohere binary rescoring; pair with full-precision rerank [mongodb.com/company/blog/voyage-code-3] |

Realistic stack at 10M docs: **Matryoshka-truncate to 1024 → int8 SQ → HNSW**. That turns 1.07 TB (NV-Embed native) into ~50 GB resident — fits on one r6i.2xlarge — with a few-point recall hit that a cross-encoder rerank recovers.

## 3. Multilingual + domain-specific

**General multilingual.** BGE-M3 dominates open-weight MIRACL (the 16-language IR benchmark) and is the default for self-hosted multilingual; its **hybrid (dense + sparse) NDCG@10 lands at 0.58–0.62** on the BEIR-style retrieval suite vs. 0.545 dense-only [iotdigitaltwinplm.com/embedding-models-benchmark-openai-cohere-voyage-bge-2026]. Cohere `embed-multilingual-v3` / `embed-v4` covers 100+ languages and is the managed-API default; it is SOTA on MIRACL among hosted models per Cohere's own published results [ai.azure.com/catalog/models/Cohere-embed-v3-multilingual]. OpenAI 3-large is multilingual but asymmetric — strong on Spanish/French/German, lags on Vietnamese/Thai/long-tail.

**Voyage domain-specific.** Voyage's specialist models still beat the generalist Voyage-3-large on their respective verticals (in the 5–10% NDCG range), though `voyage-3-large` has closed most of the gap and is the simpler operational choice for mixed-domain corpora:

| Model | Trained on | Headline result |
|---|---|---|
| `voyage-law-2` | ~1T legal tokens | **+6% NDCG@10 average over OpenAI 3-large** across 8 legal datasets; **84.44 vs 68.40 NDCG@10** (+23% relative) on long-context legal retrieval [vercel.com/ai-gateway/models/voyage-law-2] |
| `voyage-finance-2` | Filings, news, tables | **0.831 avg NDCG@10** across 11 finance sets; +7% vs OpenAI 3-large, +12% vs Cohere v3 [vercel.com/ai-gateway/models/voyage-finance-2] |
| `voyage-code-3` | 238 code corpora | **+13.80% avg vs OpenAI 3-large** and +16.81% vs CodeSage-large across 32 code datasets; +14.64% at 1024 dim and +17.66% at 256 dim [mongodb.com/company/blog/voyage-code-3] |

A 6–14 point NDCG gap on a domain corpus is roughly equivalent to a full generation of model improvement. If your 10M docs are 80%+ in one of these verticals, the specialist beats raising the generalist's dim count.

## 4. Fine-tuning embeddings on your corpus

**When it's worth it (domain-shift heuristics).**
- Base-model Recall@10 on a 200–500 query golden set is below ~0.70.
- Domain vocabulary that web-text doesn't see (CPT codes, IFRS line items, Bluebook citations, internal product IDs).
- Hybrid search is already on, the reranker is already tuned, and you're still missing the relevant chunk.
- You have (or can synthesize) ≥10k query–positive pairs.

**Training recipe (current best practice).** Use InfoNCE / MultipleNegativesRankingLoss (treats other in-batch examples as negatives — N-1 free negatives per step) with explicit **hard negatives mined with positive-aware filtering**: cap the negative-relevance threshold at **95% of the positive score** to avoid false negatives. NVIDIA's TopK-PercPos paper [arxiv.org/pdf/2407.15831] reports this is the single biggest lever in the mining stack, lifting average NDCG@10 to 60.55 — best-in-class among open mining methods. Triplet loss is the older alternative; in practice MNRL + hard negatives dominates because hard negatives are the bottleneck, not the loss form.

**Synthetic positive pair generation.** Standard pipeline: for each chunk, prompt an LLM (Llama 3.1-70B or GPT-4o-mini is plenty) with *"Generate 5 questions this passage uniquely answers"*. NVIDIA's published recipe uses this exact synthetic pipeline with **no manual labeling** and reports:

- **+10–11% Recall@10 and NDCG@10** on NVIDIA's own internal docs in <1 day on a single GPU [huggingface.co/blog/nvidia/domain-specific-embedding-finetune].
- Atlassian, applying the same recipe to JIRA, lifted **Recall@60 from 0.751 → 0.951 (+26%)** on a single GPU.

The commonly-quoted "2–5% MTEB-equivalent uplift" band is what you see when you fine-tune *and* evaluate against general MTEB — i.e., overfitting to your domain costs you a couple of points on out-of-domain tasks. On your own eval set the in-domain uplift is typically 5–15 points Recall@10, with the high end on jargon-heavy corpora.

**Cost.** A 1–2B-param dual-encoder fine-tune (BGE-M3 / E5-Mistral / GTE-base) on 50–100k synthetic pairs runs in 4–12 hours on one A100 (~$10–30 on RunPod / Lambda spot). The synthetic data generation is the dominant cost: 50k × 5 questions × ~150 output tokens × $0.60/M (GPT-4o-mini) ≈ **$22**. Total under $100 end-to-end.

## 5. Cost model at 10M docs

Assume 10M docs × 1000 tokens/doc avg = **10B tokens to embed** (the 1 chunk/doc case; multiply by chunk count for finer chunking).

**Hosted APIs — one-shot ingest cost (Batch API where available):**

| Model | $/1M tok (Batch) | 10B tok bill | Latency for one full reindex |
|---|---|---|---|
| OpenAI 3-small | $0.01 | **$100** | <24h (Batch SLA) |
| OpenAI 3-large | $0.065 | **$650** | <24h |
| Cohere embed-v4 | ~$0.10 (no batch discount) | **$1,000** | Streaming, ~1–3 days |
| Voyage-3-large | ~$0.06 (recent cut) | **$600** | Streaming |
| Voyage-3.5 | $0.06 | **$600** | Streaming |

**Self-hosted BGE-large-en-v1.5 on AWS g5.xlarge (1× A10G, $1.006/hr on-demand us-east-1; $0.30–0.40 spot):**

- Throughput: ~1,000 short-chunk embeddings/sec on A10G with batching (community-reported; the "1000 docs/min" figure in this brief is conservative — assume that as a floor for safety).
- At 1000 docs/sec: 50M chunks / 1000 = **50,000 sec ≈ 13.9 hours**.
- At 1000 docs/min (conservative): 50M / (1000×60) = ~830 hours single-GPU.
- On-demand: **$14** at the optimistic rate, **$835** at the conservative rate. Spot: $4–$300. Parallelize across 8 A10Gs to wall-clock under 2 hours.

**Re-embed cost ("how much does it cost to change models?")** is the real bill — you'll do this 2–4×/year if you're tracking the frontier. Plan for $600–$1k per re-embed on a hosted API, or ~$50–$300 on self-hosted infra, **plus** the index rebuild (often the dominant wall-clock).

**Storage cost** at the recommended 1024-dim + int8 stack (50 GB resident for 50M chunks): one r6i.2xlarge ($0.504/hr on-demand) = **~$370/mo**. Even with 3× replication and headroom, RAM cost is small relative to compute on retrieval QPS.

**Bottom line.** Embedding is **not** the expensive line item at 10M docs — a one-time $100–$1k charge plus ~$370/mo of RAM. The expensive line items are the *reranker* GPU pool serving queries and the LLM generation cost. Choose the embedding model that maximizes downstream Recall@10 (which directly controls hallucination rate in the generator) and quantize aggressively; the cost gradient on the embed itself is rarely the binding constraint.
