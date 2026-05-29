# Graph + Entity-Relation Retrieval for 10M+ Document RAG

**Lane:** `lane/v2-ingest-graph` · **Scope:** when (and how) to put a knowledge graph in front of a vector index at 10M+ doc scale, with concrete cost/latency/accuracy numbers.

Pure dense retrieval saturates on a well-defined class of queries: *multi-hop*, *global sensemaking*, and *aggregation-over-entities*. Graph/KG retrieval exists to close that gap — but at indexing costs that are 1–2 orders of magnitude above embedding-only. This section quantifies the trade.

---

## 1. The regime where graph beats vector

Vector RAG wins on single-hop factual lookup ("when did X ship?"). Graph RAG wins where the answer requires *joining* information that no single chunk contains. Concretely:

| Query regime | Vector RAG | Graph / KG RAG | Source |
|---|---|---|---|
| Single-hop factual | strong baseline | parity or worse | `[https://arxiv.org/html/2502.11371v1]` |
| Multi-hop QA (2WikiMultiHopQA) | baseline | **+11% R@2, +20% R@5** (HippoRAG) | `[https://arxiv.org/pdf/2405.14831]` |
| Multi-hop QA (MuSiQue) | baseline | **+3 pts** single-step; **+17 pts F1** w/ IRCoT-HippoRAG | `[https://proceedings.neurips.cc/paper_files/paper/2024/file/6ddc001d07ca4f319af96a3024f6dbd1-Paper-Conference.pdf]` |
| HotpotQA | competitive | marginal (~+1% F1) — dataset is mostly 2-hop with strong lexical bridges | `[https://arxiv.org/html/2405.14831v1]` |
| Global sensemaking ("what are the main themes?") | weak | **72–83% comprehensiveness win-rate, 62–82% diversity win-rate** vs vector RAG | `[https://arxiv.org/pdf/2404.16130]` |
| Enterprise multi-doc reasoning | ~32% accuracy | **~86% accuracy** with Microsoft GraphRAG hierarchical communities | `[https://medium.com/graph-praxis/graphrag-vs-hipporag-vs-pathrag-vs-og-rag-choosing-the-right-architecture-for-your-knowledge-graph-a4745e8b125f]` |

**Regime rule of thumb (use this to gate the investment):**

- **Ship graph** if ≥30–50% of production queries are multi-hop, aggregation, or "summarize across N tickets/papers/contracts."
- **Skip graph** if the query distribution is dominated by lookup ("what is the latest revision of X?"). Microsoft's own evaluation shows vector RAG produces the *most direct* answers and a similar SelfCheckGPT faithfulness score on simple queries `[https://www.microsoft.com/en-us/research/blog/graphrag-unlocking-llm-discovery-on-narrative-private-data/]`.

The independent NAACL-style evaluation in [`arXiv:2502.11371`](https://arxiv.org/html/2502.11371v1) is important: GraphRAG-Global *under-performs* vanilla RAG on comprehensiveness when measured by symmetric LLM-as-judge (position bias matters). Treat single-vendor win-rate plots with caution.

---

## 2. Architectures

| System | Indexing cost (per 1M docs, GPT-4o-class) | Query latency | Multi-hop accuracy | Incremental update? |
|---|---|---|---|---|
| **MSR GraphRAG** | **$50k–$250k**; ~3–10 LLM calls per chunk for entity+relation+claim extraction + community summaries | 2–10s (global mode traverses 100s of community reports, ~610k tokens/query on the test corpus) | Best on global sensemaking | **No** — community structure must be rebuilt; ~5k tokens × 2 × #communities `[https://arxiv.org/html/2410.05779v1]` |
| **LightRAG** | **~10–20× cheaper than GraphRAG** (one paper run: $0.10–$0.15 vs $4 for a single book corpus) | <100 tokens, **1 API call** per retrieval; sub-second | Beats GraphRAG on diversity (60–77% win-rate across 4 domains), parity-to-loss on comprehensiveness | **Yes** — graph merge without community rebuild `[https://github.com/HKUDS/LightRAG]` |
| **HippoRAG / HippoRAG 2** | ~1 LLM call/chunk for OpenIE-style extraction; PPR is offline-once | **10–30× cheaper** and **6–13× faster** than IRCoT iterative retrieval; single-pass PPR | +20 pts R@5 on 2Wiki, +11 R@2; +17 F1 on MuSiQue when combined with IRCoT | Yes (graph append + re-rank PPR seeds) `[https://arxiv.org/pdf/2405.14831]` |
| **Neo4j + LangChain `GraphCypherQAChain`** | Indexing cost = your extractor cost; Neo4j itself is cheap. Query-time cost dominated by LLM Cypher generation | **p99 ~100ms** with native HNSW + Cypher in a *single* GDS pipeline (vs 320ms sequential); 200–500ms with full hybrid + LLM `[https://markaicode.com/architecture/the-graph-production-system-design-architecture/]` | Accuracy is bottlenecked by Cypher generation: **77% correct from raw NL→Cypher → 96% with template/parameter slotting** `[https://particula.tech/blog/graphrag-implementation-enterprise-data-platform]` | Yes (UNWIND batched MERGE) |

Microsoft GraphRAG's pipeline is the most expensive because it does *four* LLM passes per chunk: (1) entity extraction, (2) relation extraction, (3) claim extraction, (4) community summarization per Leiden-detected community. At 10M docs × ~10 chunks/doc × ~$0.01/chunk for GPT-4o-mini you get **~$1M for a full re-index** — and Microsoft recommends GPT-4-class models for extraction quality. With GPT-4-class extractors costs scale to **~$10–25M** for the full corpus. This is why production teams either (a) downgrade extraction to `gpt-4o-mini` or open-weight Llama-3.1-70B, or (b) restrict graph extraction to a "hot" subset (10–20% of docs).

A Fortune-500 deployment processed 50k docs in **9 days → 18 hours (12×)** through semantic chunking, batched relationship loading, parallel conflict resolution, and bounded traversal — useful order-of-magnitude for budgeting `[https://dotzlaw.com/insights/benchmarking-and-optimizing-graphrag-systems-performance-insights-from-production-part-4-of-4/]`.

---

## 3. KG construction strategies

**LLM-based ER extraction (the default in 2025–26).** GPT-4o, Claude 3.5/4, and Llama-3.1-70B all produce usable triples. Reliability ranges from 70–85% precision on open-domain corpora. Cost dominates: at 4 LLM calls/chunk with GPT-4-class models, extraction is **30–100× the cost of embedding-only ingestion** (embedding 10M docs ≈ $1–5k with `text-embedding-3-small`; full GraphRAG-style extraction ≈ $0.5–25M as above).

**Coreference resolution is mandatory at corpus scale.** Without it, "the company", "Acme Corp", and "ACME" are three nodes, fragmenting PageRank/community signal. Options:
- LLM-resolved coref inside the extraction prompt (cheapest, lowest precision).
- Dedicated coref model (e.g., `s2e-coref`, `maverick-coref`) run as a pre-pass.
- Embedding-based entity linking + canonicalization (HippoRAG does this with `colbertv2`/`contriever` cosine similarity over node strings) `[https://github.com/OSU-NLP-Group/HippoRAG]`.

**Schema-free vs schema-guided.** Schema-free (GraphRAG, LightRAG, HippoRAG) generalizes across domains but produces relation-vocabulary explosion (often 10k+ unique predicates on a 1M-doc corpus). Schema-guided (OG-RAG, Neo4j with a fixed ontology) **reduces hallucinations by ~40%** via constrained extraction at the cost of recall on novel relations `[https://medium.com/graph-praxis/graphrag-vs-hipporag-vs-pathrag-vs-og-rag-choosing-the-right-architecture-for-your-knowledge-graph-a4745e8b125f]`. **Production guidance at 10M docs: schema-guided for closed-domain (legal, medical, internal-IT); schema-free for narrative/research corpora.**

**Iterative refinement vs one-shot.** GraphRAG's "gleaning" loop (re-prompt for missed entities, max 1–2 rounds) recovers ~15–25% additional entities but doubles cost. LightRAG skips gleaning; HippoRAG runs one OpenIE pass per chunk. *Recommendation:* one-shot at 10M scale, with offline gleaning on the top 5% most-queried docs.

**Open-source extractors (cost-pareto).**
- **REBEL** (`Babelscape/rebel-large`, BART-based seq2seq, ~200 relation types, 74 micro-F1 / 51 macro-F1) — purpose-built RE, no LLM. Throughput ~1k chunks/min on a single A10 `[https://aclanthology.org/2021.findings-emnlp.204.pdf]`.
- **GLiNER / GLiREL** — encoder-based, zero-shot, much faster than 70B LLM calls.
- **Llama-3.1-8B fine-tuned for IE** (e.g., SciPhi *Triplex*) — claimed ~98% cost reduction vs GPT-4 for KG construction `[https://model.aibase.com/models/details/1915693282472124418]`.

The pragmatic 10M-scale stack is *not* "GPT-4 for everything." It is **REBEL/GLiNER for triples + small LLM (8B–70B) for coreference and summary nodes + GPT-4-class only for the top community summaries.**

---

## 4. Hybrid graph + vector (the production pattern)

Pure-graph retrieval loses on the long tail of paraphrase recall; pure-vector loses on multi-hop. Every shipped 2024–26 system that we found is hybrid. The canonical pipeline:

1. **ANN over chunk/entity embeddings** → top-K seeds (K=10–50).
2. **Entity expansion via graph** — 1–2 hop traversal (or Personalized PageRank with the ANN hits as seeds, à la HippoRAG).
3. **Re-rank** the merged candidate set with a cross-encoder or LLM-as-judge.
4. **Generate** with the re-ranked context.

**LinkedIn customer service (Xu et al. 2024, deployed)** is the cleanest published case: Jira tickets become a dual-level graph (intra-issue tree + inter-issue semantic edges), retrieval = EBR seed → subgraph expansion → LLM answer. Measured: **MRR +77.6%, BLEU +0.32, median issue resolution time −28.6%** in a randomized A/B over 6 months `[https://arxiv.org/html/2404.17723v2]` `[https://www.zenml.io/llmops-database/knowledge-graph-enhanced-rag-for-customer-service-question-answering]`.

**Neo4j unified pattern** (now standard with `GraphCypherQAChain` + native HNSW index): one Cypher query does `CALL db.index.vector.queryNodes(...) YIELD node` followed by `MATCH (node)-[*1..2]->(related)`. This keeps p99 < 100ms for the *retrieval* leg vs ~320ms for sequential vector→graph calls `[https://markaicode.com/architecture/the-graph-production-system-design-architecture/]`.

**LangChain `MultiVectorRetriever` / parent-document / graph-augmented patterns** generalize this to "small embedding for recall, big context via graph parent for grounding."

**When the hybrid beats either alone:** queries that mention 2+ entities, queries with comparative/causal language ("why did X cause Y?"), and any query whose ground-truth answer spans ≥2 documents. On HotpotQA-distractor and 2WikiMultiHopQA, hybrid recall@5 is typically 10–20 points above either pure-vector or pure-graph.

---

## 5. Cost overhead and ROI at 10M docs

| Dimension | Embedding-only RAG | Graph/Hybrid RAG | Multiplier |
|---|---|---|---|
| **Index cost (10M docs)** | $1–5k (small embedding model) | **$50k–$25M** depending on extractor (GPT-4 vs Llama-8B fine-tuned) | **10–5000×** |
| **Index time** | ~hours on a single GPU box | Days to weeks; one 50k-doc benchmark went 9d→18h after heavy optimization `[https://dotzlaw.com/insights/benchmarking-and-optimizing-graphrag-systems-performance-insights-from-production-part-4-of-4/]` | 10–100× |
| **Storage** | embeddings dominate (e.g., 10M × 1k chunks × 1.5 KB ≈ 15 TB) | KG triples are small: typically **2–5% of raw corpus size** (entities + relations + properties); community summaries add another 0.5–2% | +3–7% |
| **Query latency** | ANN 10–30ms p99 | Hybrid: **+50–500ms** for 1–2 hop traversal + Cypher; Microsoft GraphRAG global mode: **2–10 s** because it reads 100s of community summaries | 5–500× |
| **Per-query token cost** | ~1k context tokens | LightRAG ~1k; GraphRAG global ~610k tokens (610 community reports × 1k tokens) `[https://arxiv.org/html/2410.05779v1]` | up to 600× for global mode |
| **Incremental update** | embedding for new docs only | LightRAG/HippoRAG: append-friendly; MSR GraphRAG: full community rebuild (~2× original cost per refresh) | 1× vs ∞ |

**When ROI fails to pencil out:**

- **Single-hop factual lookup** ("what's the SLA for product X?") — graph adds latency and cost, zero accuracy lift.
- **Short queries (<5 tokens) with no named entities** — entity expansion has nothing to expand.
- **High-churn corpora** where >5% of docs change daily — MSR GraphRAG's community rebuild dominates cost; use LightRAG/HippoRAG or skip the graph layer.
- **Strict latency budgets (<150ms p99)** — graph traversal + Cypher generation usually breaks the SLA unless you pre-compute query-pattern templates (Neo4j case: 77% → 96% Cypher accuracy *only* after switching to 30 parameterized templates) `[https://particula.tech/blog/graphrag-implementation-enterprise-data-platform]`.

**ROI tends to pencil out when:**

- Query mix is ≥30% multi-hop / aggregation / sensemaking.
- Each query saves ≥1 human-minute (LinkedIn: 28.6% resolution-time reduction at thousands of tickets/day pays for any extraction budget within weeks).
- The corpus is largely append-only or has a well-defined hot subset (run extraction only on the hot 10–20%).

**10M-doc recommended baseline stack:** LightRAG- or HippoRAG-style ingestion (REBEL/GLiNER for triples + Llama-3.1-70B for coreference + LLM only for community summaries on top 5% of docs) → Neo4j or LanceDB-graph store with native HNSW → hybrid retrieval (ANN seeds → 1–2 hop PPR/Cypher → cross-encoder rerank). Budget: **$50–250k extraction, +3–5% storage, +100–300ms p99 latency** over the embedding-only baseline.

---

### Key sources

- Microsoft GraphRAG paper: `[https://arxiv.org/pdf/2404.16130]`
- LightRAG paper / repo: `[https://arxiv.org/html/2410.05779v1]` `[https://github.com/HKUDS/LightRAG]`
- HippoRAG (NeurIPS '24): `[https://arxiv.org/pdf/2405.14831]` `[https://github.com/OSU-NLP-Group/HippoRAG]`
- RAG vs GraphRAG systematic eval: `[https://arxiv.org/html/2502.11371v1]`
- LinkedIn KG-RAG production: `[https://arxiv.org/html/2404.17723v2]`
- Neo4j production latency: `[https://markaicode.com/architecture/the-graph-production-system-design-architecture/]`
- Particula 12M-node case study: `[https://particula.tech/blog/graphrag-implementation-enterprise-data-platform]`
- REBEL: `[https://aclanthology.org/2021.findings-emnlp.204.pdf]`
- Architecture comparison: `[https://medium.com/graph-praxis/graphrag-vs-hipporag-vs-pathrag-vs-og-rag-choosing-the-right-architecture-for-your-knowledge-graph-a4745e8b125f]`
