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

(stub) Anthropic's technique: prepend an LLM-generated 50-100 token chunk-specific context summary to each chunk before embedding. Improves retrieval on snippet-level fragments at the cost of one extra LLM call per chunk at indexing time.

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

(stub) Open-source eval framework for RAG: faithfulness, answer relevance, context precision, context recall metrics, computed by judge LLMs. Standard baseline; known to be judge-LLM-sensitive.

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
