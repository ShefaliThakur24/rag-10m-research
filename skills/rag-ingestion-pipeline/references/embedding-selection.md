# Embedding model selection

Detailed selection guide for the embedding model in the `SKILL.md` 7-decision table. Default is `bge-base-en-v1.5` for English ≤ ~100k docs and `bge-large-en-v1.5` at 100k-10M; this file gives the leaderboard, the decision tree for non-default regimes, and the fine-tuning trigger.

## Leaderboard snapshot

Numbers below are headline **MTEB averages (English)** unless noted. Treat ±1 pt as noise (tokenizer + leaderboard refresh variance). Prices are list, per 1M input tokens. Latency p50 is for short chunks (~200 tok) on the provider's reference hardware; self-host latencies assume A10G-class GPU with batching.

| Model | License / host | Native dim | MRL truncatable to | Max input tok | MTEB avg | $/1M tok (Batch) | Latency p50 | Language support | Notes / source |
|---|---|---|---|---|---|---|---|---|---|
| **`bge-base-en-v1.5`** | BAAI, MIT, self-host | **768** | n/a | 512 | ~62-63 | self-host | ~10 ms (A10G, batched) | English | Default for English ≤ ~100k docs. Workhorse. |
| **`bge-large-en-v1.5`** | BAAI, MIT, self-host | **1024** | n/a | 512 | **63.6** | self-host | ~15 ms (A10G, batched) | English | Default at 100k-10M English docs. |
| **`bge-m3`** | BAAI, MIT, self-host | 1024 | n/a | 8192 | 63.0 dense (hybrid NDCG 0.58-0.62 on MIRACL) | self-host | ~25 ms (A10G, batched) | **100+ languages**; dense + sparse + multi-vector in one pass | Default for multilingual. Best open-weight on MIRACL. |
| **`text-embedding-3-small`** (OpenAI) | OpenAI, closed | 1536 | 256-1536 (MRL) | 8191 | 62.26 | **$0.02 ($0.01 Batch)** | ~50-150 ms (API) | English + asymmetric multilingual | 6.5× cheaper than 3-large for ~4 pt MTEB delta. Cost-floor managed option. |
| **`text-embedding-3-large`** (OpenAI) | OpenAI, closed | 3072 | 256-3072 (MRL) | 8191 | 64.6 | **$0.13 ($0.065 Batch)** | ~100-200 ms (API) | English + asymmetric multilingual | At 256 dims still beats `ada-002` at 1536 on MTEB retrieval (Azure SQL benchmark). Empirical sweet spot: **1024 dims**. |
| **`embed-v3`** / **`embed-v4`** (Cohere) | Cohere, closed | 1024 / 1536 | 256-1536 (MRL on v4) | 512 / **128000** (v4) | ~64-65.2 | ~$0.10-0.12 (no batch discount) | ~50-100 ms (API) | **100+ languages**; v4 is multimodal (text+image) | Managed multilingual default. Long context on v4. |
| **`voyage-3-large`** | Voyage, closed | 2048 | 256 / 512 / 1024 (MRL) | 32000 | 66.80 (retrieval ~74 on Voyage's eval) | ~$0.06-0.18 | ~50-100 ms (API) | English + multilingual | Strong on technical + code. Matryoshka + int8/binary native. |
| **`voyage-3.5`** | Voyage, closed | 1024 (MRL) | 256 / 512 | 32000 | parity-to-better vs 3-large at lower price | $0.06 | ~50-100 ms (API) | English + multilingual | Cost-optimized successor. |
| **`jina-embeddings-v3`** | Jina, Apache-2.0 / API | 1024 | 32-1024 (MRL) | 8192 | ~62.4 (English MTEB) | self-host or ~$0.02 (API) | ~30 ms (A10G) | Multilingual (89 langs) + task-LoRA adapters | Solid open-weight alternative with task-specific adapters (retrieval / classification / separation). |
| **`voyage-law-2`** / **`voyage-finance-2`** / **`voyage-code-3`** | Voyage, closed | 1024 (MRL) | 256-1024 | 16000-32000 | domain-leading | $0.06-0.12 | ~50-100 ms (API) | English | Domain specialists. See decision tree below for the trigger. |

**Numbers to remember:**

- `bge-large-en-v1.5`: **63.6 MTEB**, 1024 dim, MIT, no per-call cost.
- `bge-m3`: **0.58-0.62 NDCG@10 hybrid** on MIRACL multilingual.
- `voyage-code-3` vs OpenAI 3-large on 32 code datasets: **+13.80% NDCG@10 avg**, +14.64% at 1024 dim, +17.66% at 256 dim.
- `voyage-law-2` vs OpenAI 3-large on long-context legal: **84.44 vs 68.40 NDCG@10 (+23% relative)** across 8 legal datasets.
- `voyage-finance-2` vs OpenAI 3-large across 11 finance sets: **+7% NDCG@10**.
- OpenAI `text-embedding-3-large` at 256 dim **still beats `ada-002` at 1536** on MTEB retrieval (Microsoft Azure SQL benchmark).
- `text-embedding-3-large` empirical Matryoshka sweet spot: **1024 dim ≈ 99% of 3072 quality at 1/3 the bytes**.

## Decision tree

```
1. Is the corpus primarily English?
   YES → step 2.
   NO  → multilingual default: bge-m3 (self-host) OR Cohere embed-v4 (managed).
         Skip the rest of this tree.

2. Is the domain a Voyage specialist territory (legal / finance / code)
   with ≥80% of docs in that domain?
   YES → use the matching Voyage specialist
         (voyage-law-2 / voyage-finance-2 / voyage-code-3).
         Specialists beat generalists by 6-14 NDCG pts in their vertical —
         roughly a full generation of model improvement.
   NO  → step 3.

3. Is cost effectively unlimited AND quality margin matters (e.g.,
   retrieval recall directly controls revenue)?
   YES → OpenAI text-embedding-3-large, Matryoshka-truncated to 1024 dim.
         Pay $0.065/M tok Batch; ~$650 to one-shot embed 10B tokens.
   NO  → step 4.

4. Is the corpus ≤ ~100k docs?
   YES → bge-base-en-v1.5 (768 dim). Cheap, fast, MIT.
   NO  → step 5.

5. Is the corpus in the 100k-10M doc range?
   YES → bge-large-en-v1.5 (1024 dim). Quantize to int8 SQ for HNSW.
         Storage: ~50 GB for 50M chunks on one r6i.2xlarge (~$370/mo).
   NO  → step 6.

6. Is the corpus > 10M docs AND base-model Recall@10 on a 200-500
   query golden set is below ~0.70?
   YES → fine-tune the base model (see the fine-tuning trigger section).
   NO  → bge-large-en-v1.5 stays the default. Quantize harder
         (int8 → binary + rescore with cross-encoder rerank) before
         climbing to a 4096-dim model.
```

## Cost model (one-shot reindex)

Assume 10M docs × 1000 tok/doc = **10B tokens to embed** (or multiply by chunk count for finer chunking).

| Model | $/1M tok (Batch) | 10B tok bill | Latency to full reindex |
|---|---|---|---|
| OpenAI 3-small | $0.01 | **$100** | <24h (Batch SLA) |
| OpenAI 3-large | $0.065 | **$650** | <24h |
| Cohere embed-v4 | ~$0.10 | **$1,000** | Streaming, 1-3 days |
| Voyage-3-large | ~$0.06 | **$600** | Streaming |
| Voyage-3.5 | $0.06 | **$600** | Streaming |

**Self-host bge-large-en-v1.5 on AWS g5.xlarge** (1× A10G, ~$1.006/hr on-demand, $0.30-0.40 spot):

- Throughput: ~1,000 short-chunk embeddings/sec with batching.
- At 1000/sec: 50M chunks / 1000 = **50,000 sec ≈ 14 h ≈ $14** on-demand.
- Conservative 1000/min: ~830 hours = **$835** on-demand (~$300 on spot).
- Parallelize across 8× A10G: wall-clock < 2 hours.

**Re-embed cadence.** Plan 2-4× per year (model upgrades, fine-tunes). Each re-embed is **$600-$1k hosted or $50-$300 self-hosted**, **plus** the HNSW/IVF index rebuild (often the dominant wall-clock).

**Bottom line.** Embedding is not the binding cost at 10M docs — a one-time $100-$1k charge plus ~$370/mo RAM. The dominant line items are the reranker GPU pool and the generator LLM cost. Pick the embedder that maximizes downstream Recall@10 (which controls hallucination rate) and quantize aggressively.

## Fine-tuning trigger

Fine-tune the embedder on your corpus only when **all three** hold:

1. **Headroom exists.** Base-model Recall@10 on a 200-500 query golden set is below ~0.70 (or you're seeing the relevant chunk missing from top-K on production failure analyses).
2. **Domain shift is real.** Vocabulary your base model has not seen at training time: CPT codes, IFRS line items, Bluebook citations, internal product IDs, language-specific jargon.
3. **You have data.** ≥10k query-positive pairs in domain — labelled or synthesizable. Pairs are cheap to synthesize: prompt an LLM with *"Generate 5 questions this passage uniquely answers"* per chunk.

**Expected lift.** Published in-domain Recall@10 lifts at this scale:

- NVIDIA (internal docs, synthetic pairs, 1 GPU-day): **+10-11% Recall@10 and NDCG@10**.
- Atlassian (JIRA, same recipe): **Recall@60 0.751 → 0.951 (+26%)**.
- General range: **5-15 pp Recall@10** in-domain, with the high end on jargon-heavy corpora. Out-of-domain MTEB loses 2-5 pts (overfitting cost).

**Recipe (defaults):**

- **Base model**: BGE-M3 / E5-Mistral / GTE-base (1-2B params). Larger backbones offer diminishing returns relative to the GPU cost.
- **Loss**: MultipleNegativesRankingLoss (in-batch negatives = N-1 free) + explicit mined hard negatives. Triplet loss is the older alternative — in practice MNRL + hard negatives dominates because hard negatives are the actual bottleneck.
- **Hard-negatives mining**: **positive-aware filtering** capping negative-relevance at **95% of the positive score** to avoid false negatives. NVIDIA's TopK-PercPos paper: this is the single biggest lever in the mining stack, lifting average NDCG@10 to **60.55** — best-in-class among open mining methods.
- **Synthetic pair generation**: prompt Llama-3.1-70B or GPT-4o-mini per chunk with *"Generate 5 questions this passage uniquely answers"*. 50k × 5 × ~150 output tok × $0.60/M ≈ **$22** total generation cost on GPT-4o-mini.
- **Training cost**: 4-12 hours on one A100 (~$10-30 on RunPod / Lambda spot) for a 1-2B dual-encoder on 50-100k pairs.
- **End-to-end cost**: under $100 (synthetic data + training + a few iterations of eval).

**When NOT to fine-tune.** If base Recall@10 is already ≥0.85, headroom is small enough that hybrid + rerank usually buys more than fine-tuning. Also skip when the domain is well-covered by general web text (general consumer-facing English) — generalists are within a couple of MTEB points and the upkeep cost (re-fine-tune on every base-model upgrade) isn't worth it.

## Notes on dim selection (cross-link to SKILL.md decision #4)

- **256 dim**: only after Matryoshka truncation from a model trained to support it (Voyage-3-large, OpenAI v3, bge-m3, jina-v3). Pairs with cross-encoder rerank to recover the 1-2 pt Recall@10 hit.
- **768 / 1024 dim**: production sweet spot. `bge-base-en-v1.5` (768) and `bge-large-en-v1.5` (1024) are the open defaults; OpenAI 3-large Matryoshka-1024 is the managed default when you already pay OpenAI.
- **1536 / 3072 dim**: only after a measured ≥3 pt Recall@10 lift on your own eval set. RAM and HNSW latency both roughly double per doubling of D.
- **4096 dim** (NV-Embed-v2, Qwen3-Embedding-8B, BGE-en-ICL): treat as research baselines. The 4× index inflation at 10M chunks almost always dominates the headline MTEB gain. Earn the upgrade with measurement, not vibes.

## Quantization stack (cross-link to SKILL.md decision #4)

| Method | Compression | Recall vs fp32 | When to use |
|---|---|---|---|
| fp16 scalar | 2× | ~99% | Always-on, near-free. |
| **int8 scalar (SQ)** | **4×** | **~97%** | **Default for production HNSW** (Tacnode / Qdrant benchmarks). |
| Product quantization (M=96, k=256) | 32-64× | 90-95% | 1B+ vectors, RAM-bound. |
| Binary (1-bit) + rescoring | 32× | 85-95% with rerank, collapses below ~256 dim | Voyage / Cohere binary rescoring. Always pair with a full-precision rerank to recover the recall hit. |

Realistic 10M-doc stack: **Matryoshka-truncate to 1024 → int8 SQ → HNSW**. Turns 1.07 TB (NV-Embed native) or 205 GB (1024-dim fp32) into ~50 GB resident — fits on one `r6i.2xlarge`, leaves headroom for replication.
