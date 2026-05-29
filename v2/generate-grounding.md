# Grounded Generation + Cite-or-Refuse Contracts

Cite-or-refuse is easy to draw on a whiteboard and hard to ship. At 10M+ documents, the cost of a single confidently-wrong answer compounds across millions of queries, so the generation stage has to enforce a contract: every claim is either tied to a retrieved chunk, or the system explicitly abstains. This section operationalizes that contract — citation generation, constrained decoding, refusal calibration, faithfulness evaluation, and the guardrail layer that ties them together.

## 1. Citation Generation Patterns

Three patterns dominate production, and the right choice depends on the granularity of the claim.

**Footnote citation (Perplexity-style).** Answers carry numbered references inline (`...86 billion neurons [1]...`) and a source card list at the end. Perplexity averages ~21.87 inline citations per response and ties each numbered reference to a single retrieved URL; the citations are not appended post-hoc, they are structurally assigned during context assembly — the orchestration prompt embeds source IDs alongside each retrieved passage, and the LLM is constrained to cite from that pool. Best when sources are doc-level and the user wants a clickable trail. Failure mode: cheap to fake — the LLM can emit `[3]` next to an unrelated claim.

**Inline citation (`[doc#chunk]`).** A finer-grained version where each claim names both the document and chunk index. This is what most enterprise RAG stacks settle on because chunks (not docs) are the unit of retrieval; doc-level citations leak too much "somewhere in this 80-page PDF" ambiguity. Cost: longer outputs, and the prompt has to enumerate `chunk_id` for every retrieved passage.

**Anchor citation (quoted span → chunk).** The answer contains a verbatim quoted span and that span is mapped to a specific chunk offset. This is the highest-faithfulness pattern because verification reduces to a substring match in the source. It is what Anthropic's Citations API enforces natively: with `citations.enabled=true`, Claude returns structured `cite` blocks with `cited_text`, `document_index`, and `start_char_index`/`end_char_index` — and because `cited_text` is parsed out by the API rather than generated as output tokens, citations are guaranteed to be valid pointers into the supplied documents (no hallucinated quote strings). Anthropic reports the feature is "significantly more likely to cite the most relevant quotes" than prompt-based approaches.

A representative prompt fragment for inline citation when you do not have a native API:

```text
For every sentence you write, append [doc_id#chunk_id] referring
to the chunk(s) that support it. If no retrieved chunk supports
the sentence, do NOT write it. If no chunk supports any part of
the answer, respond exactly: "INSUFFICIENT_EVIDENCE".
Do not invent doc_ids or chunk_ids. Only use IDs from CONTEXT.
```

The "do not invent IDs" instruction is load-bearing — without it, smaller models will happily emit plausible-looking `[doc_42#3]` references that don't exist.

## 2. Constrained Decoding for Citation Correctness

Free-text citation generation is leaky. Three layered defenses:

**Structured output / JSON schema.** OpenAI Structured Outputs (`response_format: {type: "json_schema", strict: true}`) and Anthropic tool-use schemas compile the schema into a context-free grammar and mask invalid tokens at decode time — schema compliance is 100% by construction. Gemini's `responseSchema` does the same. Practitioners report ~82% JSON validity in GPT-5.1-class prompt-only JSON mode versus >92% complex-schema reliability under `strict: true` (with `additionalProperties: false` so the model cannot invent fields). For citations, the schema enforces that every claim object carries a non-empty `chunk_ids: [string]` array drawn from an enum of retrieved IDs:

```json
{
  "type": "object",
  "properties": {
    "claims": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "text": {"type": "string"},
          "chunk_ids": {
            "type": "array",
            "items": {"enum": ["c1","c2","c3","c4","c5"]},
            "minItems": 1
          }
        },
        "required": ["text","chunk_ids"],
        "additionalProperties": false
      }
    }
  },
  "required": ["claims"],
  "additionalProperties": false
}
```

The `enum` is the critical trick — the decoder physically cannot emit a chunk ID that wasn't retrieved.

**Logit biasing.** Where schema constraints aren't available (open-weights, vLLM, etc.), bias the logits of the citation-bracket tokens (`[`, doc-id tokens, `]`) upward at decode time, or use outlines/lm-format-enforcer to apply a regex like `\[c\d+(#\d+)?\]` directly to the sampler.

**Post-hoc citation verification.** Even with structured output, semantic correctness is not free: the LLM can produce valid JSON with `chunk_ids: ["c3"]` while the claim has nothing to do with chunk c3. A cheap verifier post-pass runs an NLI/entailment check (chunk → claim) and drops or rewrites unsupported claims. EACL 2026 work on systematic error taxonomies finds verbal LLM confidence is severely miscalibrated (Expected Calibration Error > 0.40), which is why post-hoc structural checks beat asking the model "how sure are you?".

## 3. Refusal Calibration

Refusal is a knob, not a switch. The Pareto frontier is **false-refusal rate vs. hallucination rate**, and the operating point is a product decision.

**When to refuse.** Three signals, OR'd together:
- No retrieved chunk's similarity exceeds threshold τ_sim (set per query-class; per-query-class calibration captures 2–3× the precision lift of a single global threshold).
- No chunk passes the cross-encoder rerank threshold τ_rerank.
- An LLM-judge or entailment model labels the draft answer "not supported".

**Healthy refusal rate.** Practitioner consensus (Hamel Husain's evals FAQ, EACL 2026 taxonomy): if abstention rate is 0% the system is hallucinating on out-of-domain queries; if >30% retrieval/coverage is the bottleneck. 5–15% is a healthy band for most enterprise domains. Track it as a first-class metric.

**Refusal UX.** Blank "I cannot answer" is a worse experience than a *constructive* refusal: "I don't have enough information about X in the knowledge base. The closest related material I found is [Y, doc#12], which discusses Z but does not address X." This pattern (Hamel; Sapota playbook) preserves user trust and gives the user a path forward (rephrase, expand scope, escalate).

**Refusal Tokens (arXiv 2412.06748).** A test-time-tunable strategy: train the model with a special `[refuse]` token and threshold its emission probability, so refusal sensitivity is a single knob (no retraining). Post-hoc softmax temperature scaling (τ=2) drops adjusted ECE from 0.13 → 0.08.

## 4. Faithfulness Evaluation in Practice

Three metrics, three operating points:

| Metric | Mechanic | Cost / Latency | Use when |
|---|---|---|---|
| **RAGAS Faithfulness** | LLM extracts atomic claims, NLI-style entailment vs. retrieved context. Score = supported / total. Achieves **0.95 human agreement on WikiEval** (vs. GPT-Score 0.72, GPT-Ranking 0.54). | 2 LLM calls/answer | Online gate, broad RAG eval suites |
| **FActScore** | Atomic-fact decomposition + retrieval against a reference corpus (e.g., Wikipedia). Automated estimator within <2% error of human FActScore. | Heavier (per-fact retrieval) | Offline benchmark, long-form generation |
| **TruLens Groundedness** | RAG-triad: context relevance, answer relevance, groundedness. Same shape as RAGAS faithfulness, different prompt aggregation. | Comparable to RAGAS | Production observability dashboards |
| **Vectara HHEM-2.1-Open** | Small T5 classifier for context→claim entailment. Free, fast, open-source. | Local inference, sub-100ms | Real-time gates, cost-sensitive |

**Production thresholds.** Industry practitioners converge on:
- **Faithfulness ≥ 0.95** for legal/medical/financial (Sapota gate, agility-at-scale guidance);
- **0.85–0.95** as the B2B SaaS production default — catches confident hallucinations while keeping rejection rate sane;
- **< 0.75** = the bot is reliably fabricating. Sapota reports a real-time faithfulness gate at 0.85 reduces user-reported wrong answers by **~60%**.

Citation-accuracy targets are stricter than faithfulness because citations are easier to verify mechanically — production targets cluster around **citation accuracy ≥ 0.98** (verifiable: cited span must substring-match the chunk).

Diagnostic pattern: when recall@K drops but faithfulness stays flat → retriever regression. When recall@K is stable but faithfulness drops → generator regression (model version, system message, temperature drift).

## 5. Real-World Guardrail Systems

**NVIDIA NeMo Guardrails.** Provides `self_check_hallucination` (a SelfCheckGPT variant — sample k responses at temp=1, NLI-check the original against the samples) and a `self_check_facts` rail when retrieved chunks are available. In NVIDIA's own benchmark, adding the hallucination rail to gpt-3.5-turbo lifted unanswerable-question interception from **65% → 90%** (text-davinci-003: 0% → 70%; gemini-1.0-pro: 60% → 80%). NeMo also ships native integration with Patronus AI's **Lynx** (8B and 70B variants) for hallucination detection in RAG.

**Guardrails AI.** Modular validator chain. The `ProvenanceLLM` validator runs sentence-by-sentence entailment against retrieved sources; `provenance_v0` uses embedding similarity, `provenance_v1` uses a second LLM. Pairs with structured output validators to enforce schema-level citation contracts. In a published sales-engagement deployment, layered guardrails (RAG + structured output + provenance validators + bounce-back loops) drove factual accuracy to **98%** with a <2% hard-bounce SLA.

**LangChain's CitationOutputParser / Anthropic citation prompt patterns.** Anthropic's own guidance for non-native-citation contexts: explicitly instruct "*do not invent citations; only cite passages that appear verbatim in the provided documents*" and require the model to first extract relevant quotes, then write the answer referring to those quote IDs. Mattyyeung's "deterministic quoting" pattern (which Anthropic implicitly fine-tuned toward in the Citations API) is the canonical prompt-only form.

**Net architecture.** The production stack is *layered, not single-shot*: (1) structured-output schema constrains the shape, (2) enum-bound chunk IDs prevent fabricated references, (3) post-hoc entailment validator (Lynx / HHEM / RAGAS-faithfulness) gates the response, (4) refusal logic triggers on threshold misses with a constructive fallback. The compounding effect of these layers — each catching a different failure mode — is what gets a 10M-doc RAG from "occasionally wrong" to near-zero-hallucination.

**Bottom line for the pipeline:** treat grounded generation as a *contract*, not a prompt. Constrain the decoder, verify the output, calibrate the refusal, and instrument every layer. The cost is one extra small-model call per response; the payoff is that "confidently wrong" answers become a tracked, bounded, observable failure mode instead of an open-ended liability.
