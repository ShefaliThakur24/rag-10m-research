# Problem statement

Design a retrieval-augmented generation (RAG) pipeline that serves user-facing answers over a corpus of **10M+ documents** with **near-zero hallucination**. This document is the design reference for senior ML and staff engineers building such a system.

## Operational definition of success

A pipeline ships when **all** of these hold on a held-out eval set of >=200 queries representative of production traffic:

1. **Faithfulness >= 0.95** (RAGAS faithfulness or equivalent: every assertion in the answer is entailed by retrieved context).
2. **Citation accuracy >= 0.98** (every cited span appears verbatim or near-verbatim in the cited document).
3. **Context precision @k=10 >= 0.8** (retrieved chunks are relevant to the query).
4. **Answer relevance >= 0.9** (the answer addresses the question).
5. **p95 latency <= 3s** end-to-end for the median query, **<= 8s** for multi-hop.
6. **Refusal rate when context is insufficient** >= 0.9 on adversarial probes designed to elicit hallucination.

"Near-zero hallucination" operationalized: when the corpus does not contain the answer, the system says so. When it does, the answer is grounded in cited evidence. Failure mode rate (confidently wrong answer with a real-looking citation) target: **< 1 per 10,000 queries**.

## Constraints and dimensions the design must address

The design document treats these as variables and provides tradeoff guidance per stage. The implementation team picks values per their deployment.

| Dimension | Range covered |
|---|---|
| Corpus size | 1M - 100M docs (10M is the design center) |
| Doc length distribution | from 200-token snippets to 100k-token PDFs |
| Update frequency | from quarterly snapshot to streaming hourly |
| Query type | factual lookup, multi-hop reasoning, summarization, conversational |
| Modalities | text-dominant (images/tables addressed as extensions) |
| Languages | English primary; multilingual addressed as a variant |
| Deployment | cloud, VPC, on-prem all covered |
| Latency budget | 1s - 10s |
| Per-query cost budget | $0.001 - $0.05 (inference + retrieval) |

## Non-goals

- Building a single off-the-shelf solution. Production RAG at 10M+ is configuration-specific; this doc gives the decision framework, not one implementation.
- Replacing search engines. The system answers questions with grounded citations, not lists of links.
- Solving truly unsourced knowledge. If the corpus doesn't contain the answer, the right behavior is refusal with a clear reason, not generation.
- Multi-modal generation (image/audio output). Multimodal input is in scope; multimodal output is not.

## What this document delivers

- [doc/01-approach.md](01-approach.md): the recommended pipeline end-to-end, with a justified default for each of the 7 stages and a pointer to alternatives.
- [doc/02-tradeoffs.md](02-tradeoffs.md): per-stage tradeoff matrices with regimes (corpus size, latency budget, cost budget).
- [doc/03-rejected.md](03-rejected.md): approaches considered and rejected, one-line each.
- [appendix/concepts.md](../appendix/concepts.md): 2-4 sentence refs for any technical term mentioned in the docs above.
- [appendix/evals.md](../appendix/evals.md): the eval methodology (datasets, metrics, harness) for validating a build against the success criteria.

## What this document does NOT contain

- Source code or scripts. This is a design reference. Implementations live in a separate repo and reference this doc.
- Vendor recommendations beyond what's required to anchor tradeoffs in real numbers. Where a vendor is named, it's because their published benchmarks/numbers are cited.
- A specific corpus. Replace the variables in the "Constraints and dimensions" table with your corpus's actual values to instantiate.
