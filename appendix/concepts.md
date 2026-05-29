# Concept references

Each entry is 2-4 sentences max + one canonical link. Anchors are stable; the docs link here as `appendix/concepts.md#<anchor>`. Reviewer rejects entries longer than 4 sentences.

Anchors are alphabetical for findability. Synth/lanes append new entries here; format below.

---

## ann

(stub) Approximate nearest neighbor search returns approximate top-k vectors faster than exact search by trading recall for latency. Production systems use HNSW, IVF-PQ, or ScaNN variants depending on memory/latency budget.

## bge

(stub) BGE = BAAI General Embedding model family. Open-weight dense embedding models with competitive MTEB scores; commonly used as a baseline.

## colbert

(stub) ColBERT stores per-token embeddings rather than one-per-document, scoring queries via maximum-inner-product across token pairs ("late interaction"). Higher recall on long/ambiguous queries than single-vector dense retrieval; higher storage cost.

## contextual-retrieval

Anthropic's technique: prepend an LLM-generated 50-100 token chunk-specific context summary to each chunk before embedding *and* before BM25 indexing. On internal eval, contextual embeddings alone cut top-20 retrieval failure rate 5.7%→3.7% (-35%); + contextual BM25 → 2.9% (-49%); + reranker → 1.9% (-67%). Indexing cost is one prompt-cacheable LLM call per chunk. Source: [anthropic.com/engineering/contextual-retrieval](https://www.anthropic.com/engineering/contextual-retrieval) (E-C-0001, E-C-0002, E-C-0003).

## factscore

(stub) Per-fact precision metric: decompose generated text into atomic facts and score each as supported/unsupported against a reference corpus. Standard for measuring near-zero-hallucination at fact granularity.

## graphrag

(stub) Microsoft's variant where an LLM extracts an entity-relation graph from the corpus at indexing time, then retrieves over both vector chunks and graph neighborhoods. Helps multi-hop queries; expensive to build.

## hnsw

(stub) Hierarchical Navigable Small World graphs: an ANN index with O(log N) approximate search via hierarchical proximity graphs. Standard for medium-corpus (<= 50M vectors) dense retrieval; trades RAM for low latency.

## ivf

(stub) Inverted File index for ANN: partition vectors into clusters, search only the closest clusters. Lower memory than HNSW for large corpora; tunable recall via nprobe.

## late-interaction

(stub) Score query-document by aggregating per-token similarities at query time (e.g. MaxSim across tokens), rather than once over a pooled doc embedding. ColBERT is the canonical example. Higher recall on partial/ambiguous queries; ~10-100x more storage than single-vector dense.

## mips

(stub) Maximum Inner Product Search: the underlying problem dense retrieval reduces to (given a query vector, find documents with highest dot product). HNSW/IVF/ScaNN are all MIPS solvers.

## mmr

(stub) Maximal Marginal Relevance: re-rank retrieved results to balance relevance with diversity, picking each subsequent result to minimize redundancy with already-picked ones. Useful when context windows are tight.

## pq

(stub) Product Quantization: split vectors into sub-vectors and quantize each independently. Cuts memory by 8-32x at modest recall cost; standard for >100M vector corpora.

## ragas

Reference-free RAG eval framework. Faithfulness = |LLM-verified atomic statements| / |total statements| extracted from the answer; Answer Relevance = mean cos(q, q_i) over LLM-generated reverse questions; Context Relevance = fraction of context sentences flagged crucial. Aligns with human pairwise judgement at 0.95 / 0.78 / 0.70 on WikiEval using gpt-3.5-turbo-16k judge; ctx-relevance degrades on long contexts. Canonical: [arXiv:2309.15217](https://arxiv.org/abs/2309.15217) (S-C-0011, S-C-0012, S-C-0013).

## raptor

(stub) Recursive Abstractive Processing for Tree-Organized Retrieval: cluster chunks, summarize each cluster with an LLM, recurse. Retrieval can hit any tree level. Helpful for summarization-style queries over long docs.

## rrf

(stub) Reciprocal Rank Fusion: combine multiple ranked lists (e.g. BM25 + dense) by summing `1/(k + rank_i)` per doc across lists. Robust hybrid retrieval baseline; insensitive to score scales.

## sq

(stub) Scalar Quantization: per-dimension quantization to 8-bit or 4-bit. 4x or 8x memory cut, smaller recall hit than PQ but less compression.

## trulens

(stub) Eval framework with feedback functions for groundedness, context relevance, answer relevance. Comparable in scope to RAGAS; emphasizes observability hooks for production pipelines.

## voyage

(stub) Voyage AI embedding model family. Closed-weight; competitive MTEB scores; specialized variants (code, finance, law). Pricing-relevant when embedding 10M+ docs.

---

(synth appends new entries below in alphabetical order)

## bm25

(stub) Sparse lexical retrieval scoring: query-document score from term-frequency × inverse-document-frequency with length normalization. Fast (sub-100ms at TREC scale per S-C-0005), no training, strong baseline; misses paraphrase-heavy queries that dense embeddings catch.

## cite-or-refuse

A guardrail contract at the generator boundary: every claim in the answer must be backed by a citation to a retrieved chunk; missing citations or below-threshold retrieval confidence trigger refusal at the validation layer rather than allowing unsupported generation. Converts a known failure mode (confident hallucination) into a known refusal that the product layer can handle (C-C-0005). Applicable when refusal is gracefully presentable (chat UI); not when the product mandates "always answer".

## cove

(stub) Chain-of-Verification: generate a draft answer, draft verification questions about it, answer each independently against retrieved evidence, then rewrite. Reduces unsupported claims at the cost of multiple generator passes. Relevant when compositional reasoning bounds (S-C-0010) cap single-pass faithfulness.

## hybrid-retrieval

(stub) Combine sparse (BM25) and dense retrieval, fuse rankings (e.g. RRF). On TREC DL19, lifts nDCG@10 from 50.58 (BM25 alone) to 72.50 (S-C-0005). Standard 10M+ baseline; pays a ~3.2s/query latency vs ~0.07s for BM25 alone.

## hyde

(stub) Hypothetical Document Embeddings: ask an LLM to draft a hypothetical answer to the query, embed that, and use it as the retrieval vector. Lifts nDCG@10 by ~0.84 over hybrid alone on TREC DL19 but adds an LLM call per query (~11s vs ~3s — S-C-0005). Worth it only when latency-hidden or cached.

## modular-rag

(stub) Decomposes RAG into orchestrated modules (Search, Memory, Routing, Predict, Task-Adapter) with adaptive control flow rather than a fixed retrieve→rerank→generate sequence. Extends Naive/Advanced RAG to heterogeneous query workloads (S-C-0001).

## piperag

(stub) Algorithm-system co-design that pipelines periodic retrieval with generation by accepting a stale query window; the next retrieval starts before the current generation step finishes. Up to 2.6x latency speedup over Retro at iso-perplexity on a 200B-token C4 DB (S-C-0007). Applies to Retro-style cross-attention architectures, not one-shot chat-LLM RAG.

## self-rag

(stub) Self-Reflective RAG: generator emits special reflection tokens that decide when to retrieve, what to retrieve, and whether the draft is supported. Trades extra generation overhead for adaptive retrieval and self-verification. Relevant when single-pass generation fails the compositional bound (S-C-0010).

## retriever-eval-antipattern

End-to-end RAG eval lets the generator absorb retriever failures and hide them in green pass-rates for months. The fix: evaluate retrieval as a search problem (Recall@K, Precision@K, MRR) independent of generation, with held-out QA + gold-evidence labels. Heuristic thresholds: Recall@K < 0.5-0.6 → retriever is the bottleneck; Precision@K < 0.4 → retriever is adding too much noise (C-C-0002).

## eval-driven-development

Treat evaluation as the engineering control loop: baseline → hypothesize failure → experiment → measure → iterate. Build a custom domain-specific trace-viewing UI rather than relying on generic dashboards — this yields more eval insights per engineer-hour than any off-the-shelf tool (C-C-0003). Calibrate LLM-as-judge by measuring TPR/TNR against human-annotated samples before trusting judge-LLM aggregates (C-C-0004).

## production-derived-eval

Practitioner heuristic (Hamel Husain, via Lebensold): a 70% pass rate on evals derived from real production failures is more informative than 95% on a static gold benchmark. Implication: continuously refresh the eval set from production traces; treat high pass rates as a signal the test set is too easy, not as a goal (C-C-0001).
