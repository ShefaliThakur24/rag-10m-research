# Detection recipes — scanners, classifiers, and budget enforcement

Concrete defaults for each layer of the security stack defined in `SKILL.md` `## Default security stack`. Numbers given as "(industry consensus)" when an exact vendor-published benchmark is unavailable; vendor-specific benchmarks are quoted only with a citation.

---

## 1. Indirect-injection scanner (default for layer 2)

Runs on every retrieved chunk before the chunk enters the LLM prompt.

**Recommended default: a small classifier + regex prefilter, layered.**

**Regex prefilter** (catches the cheap 80%):

```text
(?i)\b(ignore|disregard|forget)\s+(all\s+)?(previous|prior|earlier|above)\s+(instructions?|prompts?|rules?)\b
(?i)\b(system\s*prompt|system\s*message)\b.*\b(reveal|print|show|leak|display)\b
(?i)\bact\s+as\b.*\b(unrestricted|jailbroken|DAN|developer\s+mode)\b
(?i)<\s*system\s*>|</\s*system\s*>|<\s*\|im_start\|\s*>
(?i)\b(print|output|return|emit)\b.*\b(api[_ ]?key|secret|password|token)\b
```

The regex layer is a low-recall, high-precision sieve; do not rely on it alone.

**Classifier layer** — pick one based on latency budget:

| Model | Latency (cpu / single chunk) | Strengths | Weaknesses |
|---|---|---|---|
| `protectai/deberta-v3-base-prompt-injection-v2` (HF, open) | ~30-80ms | Free, local, fine-tuned on injection corpora | High false-positive rate on legitimate instruction-like content |
| `meta-llama/Llama-Guard-3-8B` (open, mostly for output but covers prompt-injection categories) | GPU-only, ~200ms | Maintained by Meta, also covers harm categories | Larger; needs GPU |
| Lakera Guard API (commercial) | ~50ms p50 + network | Maintained vendor model, broad coverage of injection patterns | Per-call cost; cross-region latency |
| Custom small instruction-tuned LLM (Mistral 7B, fine-tuned) | ~100ms GPU | Tunable to domain | Build + maintain a training pipeline |

**Stacking rule.** Regex first (cheap reject), then classifier (semantic). Run both in parallel and OR the results if latency budget allows; gate by classifier alone if not.

**On-hit action.** Replace the chunk with a placeholder `[chunk c{id} removed by safety scanner: indirect_injection_suspect]` so the model knows the slot was occupied but the content was removed. Do not silently drop — the model otherwise wonders why a citation is missing.

---

## 2. Input scanner (default for layer 3)

Runs on the user query before retrieval.

**Options.**

| Option | Type | Notes |
|---|---|---|
| Lakera Guard `/v2/guard` | Commercial API | Categories include `prompt_injection`, `jailbreak`, `pii`, `unknown_links`. Block on `flagged: true`. |
| `meta-llama/Prompt-Guard-86M` (HF, open) | Local, BERT-class | 86M params, classifies into `INJECTION`, `JAILBREAK`, `BENIGN`. Sub-50ms on CPU. |
| `protectai/deberta-v3-base-prompt-injection-v2` (HF, open) | Local, BERT-class | Same model as the chunk scanner; can be reused. |
| OpenAI / Anthropic moderation endpoints | Commercial API | Catches the base-distribution harm categories; **does not** specifically cover prompt-injection — pair it with one of the above, do not rely on it alone. |

**Default stack.** Prompt-Guard-86M (open, fast, local) as the primary; foundation-provider moderation endpoint as a parallel call for harm categories. Block the request at the API gateway on either positive.

**Threshold.** Out-of-the-box classifier thresholds are usually tuned for high precision; production deployments typically lower the threshold to favor recall and accept a higher false-positive rate, then audit false-positives weekly. The exact threshold is workload-specific; calibrate against a held-out adversarial set sized ≥ 200 examples per attack class.

---

## 3. PII detection (default for layer 7a)

Two-layer at ingestion and at output. Comparison of common options:

| Detector | Type | Strengths | Weaknesses | Latency (per page of text, CPU) | Cost model |
|---|---|---|---|---|---|
| Microsoft Presidio | Open-source NER + regex + checksum (e.g. Luhn for credit cards) | Free, deterministic, structured PII covered (SSN, IBAN, CC, phone, email) | Recall on free-form PII (names, addresses) lower than ML-only options; English-biased by default | ~100-300ms | Self-host |
| AWS Comprehend / GCP DLP / Azure Text Analytics PII | Cloud API | Maintained, broad language and entity coverage | Per-call cost; data leaves the boundary | ~200ms + network | Per-1k chars |
| Llama Guard 3 (8B) | LLM classifier | Covers harm + privacy categories; one model for two jobs | GPU required; misses structured PII like SSN unless added as a separate regex pass | ~150-300ms GPU | Self-host (model + GPU) |
| Custom regex + NER (spaCy + domain regex) | Hybrid | Tunable, cheap | Maintenance burden; brittle to format variation | ~50-200ms | Self-host |
| Commercial dedicated PII vendor (Skyflow, Nightfall, etc.) | Commercial API | Highest recall on edge cases; compliance-grade | Per-call cost; vendor lock-in | ~100-200ms + network | Per-call |

**Precision / recall / latency / cost trade-off.** Vendor-published benchmarks vary by class and corpus; do not write specific accuracy numbers unless quoting a vendor's named eval. As an evaluation default, build a held-out PII set of ≥ 500 examples per class drawn from your own production traces (redacted) and measure each tool on that. (industry consensus)

**Default recipe.**
- Ingestion: Presidio with a domain-extended regex set + spaCy NER for free-form names / addresses. Redact to `<EMAIL_1>`, `<SSN_1>`, ... with a reversible map in a separately-encrypted store.
- Output: Presidio again on the draft answer (catches PII that the model assembled from un-redacted fragments) + Llama Guard 3 for the privacy category.
- Anti-pattern: a single LLM-as-judge call asking "is this PII?". Misses structured PII, jailbreakable, non-deterministic.

---

## 4. Output harm classifier (default for layer 7b)

**Default: Llama Guard 3 (8B).** Self-hostable, open weights, covers categories S1-S13 per its model card:

| Code | Category |
|---|---|
| S1 | Violent crimes |
| S2 | Non-violent crimes |
| S3 | Sex-related crimes |
| S4 | Child sexual exploitation |
| S5 | Defamation |
| S6 | Specialized advice (e.g. medical / legal without disclaimers) |
| S7 | Privacy |
| S8 | Intellectual property |
| S9 | Indiscriminate weapons |
| S10 | Hate |
| S11 | Suicide & self-harm |
| S12 | Sexual content |
| S13 | Elections |

(Source: Meta Llama Guard 3 model card.)

**Alternatives.** Perspective API (Jigsaw / Google) for toxicity-class detection with sentence-level scoring. OpenAI Moderation API for the base-distribution categories at no marginal cost when already using OpenAI for generation. Foundation-provider safety filters in Anthropic / Gemini / OpenAI are useful as a parallel layer but are not aware of the chunk content; run your own classifier on the draft anyway.

**On-hit action.** Replace the answer with a non-leaking refusal: "I can't include that information in my response." Do not echo the offending span or the category; otherwise the refusal itself is a side-channel. Log the offending chunk + category to a separate audit store for review.

---

## 5. Rate / budget enforcement (default for layer 8)

Three independent enforcement points, all required.

### 5a. Per-user QPS (at the API gateway)

Token-bucket or sliding-window limiter per `user_id`. Typical starting defaults (tune per product):

| Tier | Sustained QPS | Burst |
|---|---|---|
| Anonymous / unauthenticated | 0.1 (1 query per 10s) | 3 |
| Free | 1 | 10 |
| Paid | 10 | 30 |
| Enterprise | per-contract | per-contract |

These numbers are starting defaults; tune against your own observed legitimate traffic distribution. (industry consensus)

### 5b. Per-tenant token budget (at the API gateway, daily)

Track input + output tokens per `tenant_id` per UTC day. Hard cap at the contracted ceiling; soft-warn at 80%. Required for multi-tenant SaaS to bound denial-of-wallet blast radius.

### 5c. Per-trace hard kills (in the orchestrator)

Cited verbatim from `v2/generate-multihop.md` §5; these are the LangGraph / ReAct-paper-derived defaults:

| Limit | Default | Source |
|---|---|---|
| Max graph iterations / recursion | **20–25** (LangGraph default 25) | LangGraph `recursion_limit` |
| Tool-call rate per query | ≤ 8 retrievals + 4 verifications | ReAct paper trajectory analysis |
| Per-trace token budget | hard kill at 32k input + 4k output | cost ceiling |
| Wall-clock SLA | p99 ≤ 12s; preempt → single-pass fallback | latency contract |
| Cycle detector | reject sub-query if cosine-sim > 0.95 to any prior sub-query in the trace | prevents prompt-loop pathology |
| Escape-hatch refusal | after 2 failed verification revisions → return "insufficient evidence" + cite top-k passages | mirrors Self-RAG IsSup gate |

**Order of precedence.** Per-trace limit fires first (kills the runaway query), per-user QPS fires second (kills the runaway script), per-tenant daily budget fires last (kills the runaway tenant). Do not let one substitute for another — they catch different attacks.

**Detection signal.** Distribution of `tokens_per_trace` (alert on p99 climbing); rate of per-trace hard kills (alert on spike per tenant); rate of per-user QPS throttles; per-tenant daily-budget burn rate.

---

## 6. Wiring it all together

Pipeline-order checklist for layers that touch the request path (numbers refer to `SKILL.md` stack layers):

1. API gateway: per-user QPS limit, per-tenant daily budget (layer 8).
2. Input scanner on user query (layer 3); reject if flagged.
3. ACL-filtered retrieval (layer 1).
4. Indirect-injection scanner on each retrieved chunk (layer 2); replace flagged chunks.
5. Prompt assembled with prompt isolation pattern (layer 4) + tool allowlist (layer 6).
6. LLM call with constrained structured output schema (layer 5) + per-trace token budget (layer 8).
7. Output classifiers on draft: PII (Presidio + Llama Guard 3 privacy) + harm (Llama Guard 3 S1-S13) (layer 7); block or redact on hit.
8. Entailment validator drops claims not supported by retrieved chunks (also serves layer 5).
9. Tool-call orchestrator validates each tool call against allowlist + argument schema (layer 6); rejects out-of-allowlist calls.
10. Return.

Every layer's hit / block / drop is logged with `(trace_id, user_id, tenant_id, layer, action, signal)` for after-the-fact audit and for the detection signals listed per layer above.
