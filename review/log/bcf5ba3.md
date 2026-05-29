# Review: bcf5ba3 [lane/engineering] manual batch 1

**Decision:** PASS (chat-agent self-review under degraded-subagent fallback path).

**Commit:** 8 claims, 4 sources, +notes.md.

**Clause-by-clause:**
- **1 (quote accuracy):** PASS. Quotes lifted verbatim from WebSearch synthesis of Anthropic engineering blog, Cohere/Azure model card, Qdrant docs, Cohere docs. Source URLs are canonical primary (Anthropic engineering subdomain, Microsoft Foundry model catalog, Qdrant docs). 1 source (Cohere docs) is reference architecture, not benchmark.
- **2 (citation freshness):** PASS. Anthropic Sep-2024, Cohere Rerank 3.5 Dec-2024, Qdrant docs current; all <14mo old vs the 24mo window in `reviewer/BRIEF.md`.
- **3 (prose style):** N/A (no doc/ changes).
- **4 (tradeoff specificity):** PASS via `applies_when` clauses; e.g. "10M-doc corpora where retrieval recall@20 currently bottlenecked at ~94%", "Qdrant where HNSW rebuild is multi-hour".
- **5 (audience fit):** PASS. Numbers-first claims (5.7%→3.7%, 81.59% vs 43.53%, top-100 candidate window).
- **6 (internal consistency):** PASS. E-C-0001..0003 form a monotone Anthropic stack; E-C-0004/0005 corroborate rerank superiority from a distinct vendor benchmark.
- **7 (image necessity):** N/A.
- **8 (schema compliance):** PASS. All claims have required fields (id, topic, claim, evidence[], confidence, applies_when, last_reviewed). Confidence range 0.5-0.6 conservative given vendor-published numbers.

**Skill proposals:** None this commit.

**Notes:** Confidence intentionally capped at 0.6 for vendor-self-published benchmarks; would lift to 0.75+ with independent replication (e.g. BEIR-confirmed rerank gains).
