---
name: rag-grounded-generation
description: Designs and debugs the generator boundary of a production RAG so every claim is either cited to a retrieved chunk or explicitly refused. Defaults to structured output with enum-bound chunk IDs (decoder physically cannot fabricate references), a cheap NLI post-hoc entailment verifier, three-signal OR refusal (τ_sim, τ_rerank, entailment), per-query-class threshold calibration (2-3× precision lift over a global threshold), constructive refusals, and a layered RAGAS + HHEM + Lynx faithfulness eval stack. Use when the user mentions grounded generation, cite-or-refuse, RAG citation, constrained decoding, structured output citation, refusal calibration, faithfulness, Chain-of-Verification, Self-RAG, NeMo Guardrails hallucination, Lynx faithfulness, RAGAS faithfulness, or is debugging confident hallucinations / unsupported citations in a RAG system. NOT for prompt injection from retrieved chunks, PII leakage, or jailbreak defenses — see rag-security-guardrails.
---

# rag-grounded-generation

Turns retrieved chunks into a cited answer OR an explicit refusal. Cite-or-refuse is the system-level invariant — not a prompt instruction, a contract enforced by the decoder, verified by a second pass, and gated by calibrated thresholds.

## When to use this skill

Triggers: designing the generator stage of a RAG over 10M+ docs; debugging "the model cited chunk c3 but c3 doesn't say that"; tuning refusal rate (too eager / too lax); choosing between RAGAS / HHEM / Lynx; deciding whether to add CoVe or Self-RAG-style IsSup gating; wiring NeMo Guardrails or Guardrails AI.

NOT for: prompt injection from retrieved chunks, PII leakage, jailbreaks, content policy — see `rag-security-guardrails`.

## The cite-or-refuse contract

> Every claim → cited chunk OR explicit refusal. No third option.

Enforce mechanically — prompts that say "always cite" without structural enforcement leak. Decision flow:

```
retrieve k chunks
  ├─ all sims < τ_sim?              → REFUSE (constructive)
  └─ rerank
       ├─ all rerank < τ_rerank?    → REFUSE (constructive)
       └─ generate with enum-bound chunk_ids
            └─ post-hoc NLI(chunk → claim)
                 ├─ any claim unsupported? → drop/rewrite; if none survive → REFUSE
                 └─ all supported          → RETURN cited answer
```

Refusal is a first-class output, not an exception. Track refusal rate as a product metric.

## The 8 generation decisions

| # | Decision | Default | Quantified rationale | Escape hatch |
|---|---|---|---|---|
| 1 | Citation enforcement | **Structured output with `enum`-bound chunk IDs** (OpenAI `strict: true`, Anthropic tool schemas, Gemini `responseSchema`). Decoder physically cannot emit a chunk ID that wasn't retrieved. | Prompt-only JSON ~82% valid vs >92% under strict schemas. EACL 2026: verbal LLM confidence ECE > 0.40 — structural > behavioral. | Open-weights/vLLM: use `outlines` / `lm-format-enforcer` regex on the sampler, or logit-bias citation-bracket tokens. |
| 2 | Post-hoc verifier | **NLI/entailment check (chunk → claim)** on every claim before responding. Drop or rewrite unsupported claims. Cheap second-pass small model (HHEM-2.1-Open sub-100ms). | Catches valid-JSON-wrong-semantics: `chunk_ids: ["c3"]` while the claim isn't in c3. Asking the model "are you sure?" is uncalibrated (ECE > 0.40). | Hard latency budget (<300ms p99): drop to embedding-similarity-only check (Guardrails AI `provenance_v0`). |
| 3 | Refusal signals | **OR of three signals** — (a) no chunk passes τ_sim, (b) no chunk passes τ_rerank, (c) entailment model labels draft "not supported". | AND-ing collapses to "never refuse"; single signal misses retriever-vs-generator failure split. See `references/refusal-calibration.md`. | Best-effort search UI where any answer beats none: relax to signal (a) only and surface confidence. |
| 4 | Refusal calibration | **Per-query-class thresholds.** Cluster traffic, fit τ_sim and τ_rerank per cluster. Target 5-15% refusal rate. | Per-class calibration: 2-3× precision lift vs single global threshold. 0% refusal = hallucinating OOD; >30% = retrieval is the bottleneck. | Cold-start (no traffic to cluster): start with a single global τ and a manual review queue; promote to per-class once a few hundred refusals are labeled. |
| 5 | Refusal UX | **Constructive refusal** — "I don't have enough information about X. Closest related material is [Y, doc#12], which discusses Z but does not address X." Never blank "I cannot answer." | Preserves trust + gives user a path forward (rephrase, expand scope, escalate). Reduces churn vs hard wall. | Compliance/regulated contexts where naming "closest related" leaks sensitive info: short, neutral refusal + escalation link. |
| 6 | Faithfulness eval | **Layered: RAGAS faithfulness + Vectara HHEM-2.1-Open + Patronus Lynx.** Production-derived eval set sampled from real traffic, not static gold. | RAGAS 0.95 human agreement on WikiEval (GPT-Score 0.72, GPT-Ranking 0.54). HHEM = fast local gate. Lynx (8B/70B) = ceiling. Hamel: 70% on production-derived > 95% on static. | Sub-100ms gate budget: HHEM-only and accept the recall loss; run RAGAS+Lynx offline on the sampled batch. |
| 7 | Guardrails stack | **Layered, not single-shot:** system prompt → structured output schema → enum-bound chunk IDs → post-hoc entailment validator → refusal with constructive fallback. | NeMo `self_check_hallucination` lifted unanswerable interception 65% → 90% (gpt-3.5-turbo), 0% → 70% (text-davinci-003), 60% → 80% (gemini-1.0-pro). Guardrails AI ProvenanceLLM + structured output reached 98% factual accuracy with <2% hard-bounce SLA. | Latency-critical p99 ≤ 1s: collapse to schema + embedding-similarity provenance only; sample 10% of traffic into the full stack offline. |
| 8 | Verify-loops | **CoVe for longform / claims-list answers** (factored, not joint). **Self-RAG-style `IsSup` gating on every retrieval-grounded segment.** Don't run CoVe on every query. | CoVe lifts MultiSpanQA F1 +23% (0.39 → 0.48), Wikidata list precision 2.1×, longform FactScore +15.5 pts. Cost: 4 sequential LLM calls (~3-4×). | Single-hop short-answer trivia: skip CoVe (verify question ≈ original). Latency-bound paths: gate CoVe on uncertainty signal (low rerank top-1, OR signal (c) triggered). |

## Citation enforcement (structured-output recipe)

The schema below is the canonical contract. `chunk_ids[].enum` is rebuilt per-request from the retrieved IDs — the decoder cannot emit a string outside that set.

```json
{
  "type": "object",
  "properties": {
    "refusal": {"type": "boolean"},
    "refusal_reason": {"type": "string"},
    "answer": {"type": "string"},
    "claims": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "text": {"type": "string"},
          "chunk_ids": {
            "type": "array",
            "items": {"enum": ["c1", "c2", "c3", "c4", "c5"]},
            "minItems": 1
          }
        },
        "required": ["text", "chunk_ids"],
        "additionalProperties": false
      }
    }
  },
  "required": ["refusal", "answer", "claims"],
  "additionalProperties": false
}
```

Pair with `response_format: {type: "json_schema", strict: true}` (OpenAI), `tool_choice` + JSON schema (Anthropic), or `responseSchema` (Gemini). Set `additionalProperties: false` everywhere — without it, the model invents fields.

Full schema + post-hoc validator pseudocode + NeMo / Guardrails AI integration snippets: `references/cite-or-refuse-contract.md`.

## Refusal signals + threshold tuning

Refusal fires when **any** of these is true (OR, not AND):

1. `max_i sim(q, chunk_i) < τ_sim[class(q)]`
2. `max_i rerank(q, chunk_i) < τ_rerank[class(q)]`
3. `entailment(merge(chunks), draft) ∈ {"not_supported", "partial"}`

τ thresholds are **per-query-class**, not global. Refusal Tokens ([arXiv:2412.06748](https://arxiv.org/abs/2412.06748)) + post-hoc temperature scaling (τ=2) drop adjusted ECE 0.13 → 0.08.

Recipe (cluster → fit τ → target 5-15% per class), constructive refusal templates, and domain-specific healthy rates (general 5-15%, regulated/medical 15-25%, narrow-domain support 2-8%): `references/refusal-calibration.md`.

## Faithfulness eval

Default stack — layered, each catches a different failure mode:

| Layer | Tool | Latency | Catches |
|---|---|---|---|
| Online gate | **HHEM-2.1-Open** (T5 entailment classifier) | <100ms local | Confident hallucination at request time |
| Online judge | **RAGAS Faithfulness** (atomic-claims NLI) | ~2 LLM calls | Subtle unsupported claims in multi-claim answers |
| Offline ceiling | **Patronus Lynx 8B / 70B** | Batched | Long-form, subtle entailment errors HHEM misses |

Production thresholds (converged across practitioners): faithfulness ≥ 0.95 for legal/medical/financial; 0.85-0.95 B2B SaaS default; <0.75 = reliably fabricating. Citation accuracy ≥ 0.98 (substring-verifiable). A 0.85 real-time faithfulness gate cuts user-reported wrong answers ~60%.

**Diagnostic matrix** (when something regresses, look here first):

| Recall@K | Faithfulness | Diagnosis |
|---|---|---|
| ↓ | flat | **Retriever regression** (index drift, embedding model swap, BM25 stop-word change) |
| flat | ↓ | **Generator regression** (model version, system prompt drift, temperature, schema change) |
| ↓ | ↓ | Upstream: chunking, ingestion, or query rewriter |
| flat | flat, refusal rate ↑ | Threshold drift or traffic shift — re-cluster query classes |

Always sample the eval set from real production traces — static benchmarks lie. 70% on production-derived > 95% on static gold.

## Anti-patterns

- **Asking the model "how sure are you?"** Verbal LLM confidence ECE > 0.40 (EACL 2026). Use structural checks (entailment, threshold, schema) — never verbal self-reports.
- **Prompting "always cite" without enforcement.** Smaller models emit plausible-looking `[doc_42#3]` references that don't exist. Use enum-bound structured output.
- **Single-shot guardrails.** One LLM-judge or one schema check fails open. Layer: schema → enum → entailment → refusal.
- **Blank refusals.** "I cannot answer" with no closest-related-material destroys trust. Always constructive.
- **Single global threshold.** Misses 2-3× precision lift available from per-query-class calibration. Cluster first, threshold per cluster.
- **CoVe on every query.** ~3-4× cost for no benefit on single-hop trivia; can hurt when verification Q ≈ original. Gate CoVe on response type or uncertainty.
- **End-to-end eval only.** Generator absorbs retriever failures and hides them in green pass-rates. Eval retrieval (Recall@K, MRR) and generation (faithfulness, citation accuracy) separately.
- **Static gold eval as the only gate.** Refresh from production traces continuously; treat high pass rates as a too-easy test set, not a goal.

## Additional resources

- `references/cite-or-refuse-contract.md` — read when wiring the structured-output schema, writing the post-hoc entailment validator, or integrating NeMo Guardrails / Guardrails AI for the first time.
- `references/refusal-calibration.md` — read when refusal rate is outside the 5-15% band, choosing per-query-class thresholds, or designing constructive-refusal templates for a new domain.
- Companion skill `rag-security-guardrails` — read when the question is about prompt injection from retrieved chunks, PII leakage in generated answers, jailbreak resistance, or content-policy filtering.
- Source deep-dives in this repo: `v2/generate-grounding.md` (citation patterns, constrained decoding, eval); `v2/generate-multihop.md` (CoVe, Self-RAG IsSup, escape-hatch refusal); `FINAL.md` §9 (grounded generation), §10 (multi-hop verify loops).
