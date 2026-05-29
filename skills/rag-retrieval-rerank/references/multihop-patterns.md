# Multi-hop patterns

Read this when deciding which multi-hop pattern to ship, debugging a runaway agent loop, or writing a decomposition prompt. The default in `SKILL.md` (decision 6: HippoRAG 2 in production, IRCoT fallback, CoVe for post-hoc verification) is derived here.

## 1. One-row-per-pattern comparison

| Pattern | When it wins | Cost vs single-pass | Benchmark numbers (source) |
|---|---|---|---|
| **IRCoT** (Trivedi et al. 2023) | 2-4 hop QA where graph extraction is infeasible (corpus turnover > 5%/day). Each CoT sentence becomes a retrieval query; alternate generate-one-step / retrieve / append. | 3-5× LLM calls per query (one per hop + final compose); proportional latency increase. | Recall@k +11-21 pts; QA F1 +7.1 HotpotQA, +13.2 2Wiki, +5-9 MuSiQue; **factual errors in CoT cut by ~50% on HotpotQA, ~40% on 2Wiki** (v2/generate-multihop.md §2). |
| **CoVe** (Dhuliawala et al. 2023) | Longform / claim-list / Wikidata-flavor outputs where each sentence is a verifiable claim. Post-hoc verification, not retrieval. | 4 sequential LLM calls (~3-4× cost). Use *factored* (separate verification calls), not joint. | MultiSpanQA F1 0.39 → 0.48 (**+23%**); Wikidata list precision 0.17 → 0.36 (**2.1×**); FactScore longform 55.9 → 71.4 (+15.5 pts). |
| **Self-RAG** (Asai et al. 2023) | Mixed-workload traffic where ~30-50% of segments don't need retrieval (Retrieve=No tokens skip retrieval entirely). Requires fine-tuned generator. | ~1.5-2× a vanilla RAG decode at 13B; net win because of skipped retrievals on Retrieve=No segments. | Self-RAG-13B: PubHealth 74.5, ARC-Challenge 73.1, PopQA 55.8, TriviaQA 69.3, ASQA citation precision 70.3 (v2/generate-multihop.md §4). |
| **HippoRAG** (Gutiérrez et al. 2024) | 2-4 hop QA over a stable corpus where you can afford one OpenIE pass per chunk at index time. Single-pass Personalized PageRank — no per-hop LLM call. | Index-time cost: ~1 LLM call/chunk for OpenIE. Query-time: **10-30× lower cost, 6-13× lower latency** than IRCoT. | +20 pts R@5 on 2Wiki, +11 R@2; +17 F1 on MuSiQue when combined with IRCoT (v2/ingest-graph.md §1). |
| **HippoRAG 2** (Gutiérrez et al. 2025) | **Production default for multi-hop.** Same shape as HippoRAG with refined PPR seeding and entity linking. | Same index-time profile as HippoRAG; same 10-30× cost / 6-13× latency advantage over IRCoT. | **R@5: MuSiQue 74.7, 2Wiki 90.4, HotpotQA 96.3** (v2/generate-multihop.md §2). |
| **ReAct** (Yao et al. 2022) | Tool-use agentic flows where the loop must call retrieval, calculators, and other tools, not just retrieve-and-reason. | Per-trace token cost grows with trace length; needs hard guardrails (§3 below). | HotpotQA EM: ReAct 27.4; ReAct+CoT (best-of) **35.1**; FEVER 60.9. Pure ReAct under-uses parametric knowledge — ship the ReAct↔CoT fallback policy, not pure ReAct. |

**Default decision (from `SKILL.md`):**

- HippoRAG 2 if you can precompute the entity graph (most production corpora can; corpus turnover < 5%/day).
- IRCoT if you cannot precompute the graph.
- CoVe layered on top of either, only for longform / claim-list outputs.
- ReAct only when the agent needs tools beyond retrieve-and-reason.
- Self-RAG only if you control the generator weights (fine-tune required).

## 2. Canonical decomposition prompt template

Use this exactly. It is intentionally terse — agents drift when the decomposition prompt has too many examples.

```text
You are decomposing a multi-hop question into a sequence of single-hop sub-questions.

Question: {q}

Rules:
- Output a numbered list of sub-questions, each answerable from a SINGLE retrieved passage.
- Each sub-question must depend only on (a) the original question and (b) the answers to PRIOR sub-questions in this list.
- If a sub-question references a prior answer, write it as {{answer_to_subq_N}} where N is the index of the prior sub-question.
- Stop when the original question can be answered from the union of all sub-question answers.
- Maximum 5 sub-questions. If you need more than 5, the question is out of scope — output exactly: DECOMPOSITION_FAILED.

Output format:
1. <sub-question 1>
2. <sub-question 2 [may reference {{answer_to_subq_1}}]>
...
```

The `DECOMPOSITION_FAILED` escape valve is load-bearing. Without it, the model will pad to its training distribution length and emit speculative sub-questions.

## 3. Cycle-detector + iter-cap heuristics

These are guardrails for any iterative loop (IRCoT, ReAct, HippoRAG 2 + IRCoT). Wire them in day one — observed failure modes are not hypothetical.

| Guardrail | Default | Why this number |
|---|---|---|
| Max graph iterations / recursion | **20-25** (LangGraph default 25) | The compositionality bound bites at ~3 implicit hops; >25 iterations means the agent is looping, not reasoning. |
| Cycle detector | **Reject any sub-query with cosine-sim > 0.95 to any prior sub-query in the trace** | Prevents the prompt-loop pathology where the agent re-issues a near-identical query forever. |
| Tool-call budget | **≤ 8 retrievals + 4 verifications per query** | Per ReAct paper trajectory analysis. Higher budgets observe diminishing returns and unbounded cost growth. |
| Per-trace token budget | **Hard kill at 32k input + 4k output** | Cost ceiling. Soft signal — if you hit this, the loop has gone wrong. |
| Escape-hatch refusal | **If verification IsSup score < threshold after 2 revisions, return "insufficient evidence" + cite top-k passages** | Mirrors Self-RAG's IsSup gate. Converts "confidently wrong" into "I don't know" — the only acceptable failure mode for a hallucination-sensitive system. |
| Wall-clock SLA | **p99 ≤ 12s; on preempt, fall back to single-pass with high-recall retrieval** | Latency contract. The fallback is intentional: a worse-but-fast answer beats a stalled agent. |
| Context bloat mitigation | **Summarize observation traces older than N=2 hops; keep raw text only for the last 2 hops** | The model's effective context window degrades long before its hard limit ("lost in the middle"). |

**Implementation order when you ship multi-hop:**

1. iter cap + wall-clock SLA (cheapest; prevents cost catastrophe).
2. cycle detector (the next-most-common failure once iter cap is in place).
3. escape-hatch refusal (highest leverage on user trust).
4. tool-call + token budgets (enforce SLOs).
5. context-bloat summarization (only when hop length grows enough to matter).

## 4. Pattern-specific gotchas

- **IRCoT**: the prompt that emits each CoT sentence must be conditioned on the *retrieved-so-far* context, not just the question. Without this, the CoT drifts and re-issues redundant retrieval queries.
- **CoVe**: must be **factored**, not joint. Answering all verification questions in a single batched prompt lets the draft contaminate verification — the very failure mode CoVe is supposed to prevent. Use a fresh context per verification question.
- **HippoRAG 2**: Personalized PageRank seeds come from ANN hits over entity-name embeddings, not from the raw query embedding. If your entity linker is weak, PPR quality collapses; pre-pass with `colbertv2` / `contriever`-style canonicalization.
- **ReAct**: ship `ReAct↔CoT (best-of)`, not pure ReAct. Pure ReAct under-uses parametric knowledge and loses to plain CoT on questions the model already knows.
- **Self-RAG**: requires fine-tuning the generator. Do not attempt prompt-only Self-RAG — the reflection tokens are an artifact of the training procedure, not a prompt format.

## 5. References

- IRCoT — Trivedi et al. 2023, "Interleaving Retrieval with Chain-of-Thought Reasoning for Knowledge-Intensive Multi-Step Questions" (ACL 2023.acl-long.557).
- CoVe — Dhuliawala et al. 2023, "Chain-of-Verification Reduces Hallucination in Large Language Models" (arXiv:2309.11495).
- Self-RAG — Asai et al. 2023, "Self-RAG: Learning to Retrieve, Generate, and Critique through Self-Reflection" (arXiv:2310.11511).
- HippoRAG — Gutiérrez et al. 2024 (NeurIPS).
- HippoRAG 2 — Gutiérrez et al. 2025 (arXiv:2502.14802).
- ReAct — Yao et al. 2022, "ReAct: Synergizing Reasoning and Acting in Language Models" (arXiv:2210.03629).
- Source material in this corpus: `v2/generate-multihop.md` (quantified comparison + guardrails); `v2/ingest-graph.md` §1 (graph regime boundaries).
