# Cite-or-refuse contract (reference)

> **The contract:** every claim → cited chunk OR explicit refusal. No third option.

The contract is enforced **mechanically**, not by instruction. Prompts that say "always cite" leak. The structured-output schema below makes fabricated citations decode-impossible; the post-hoc entailment validator catches semantically wrong (but schema-valid) citations; the refusal branch handles the case where no chunk supports any claim.

## 1. Structured-output schema (canonical)

The `chunk_ids[].enum` is rebuilt per-request from the retrieved IDs. Compile with `strict: true` (OpenAI Structured Outputs) or equivalent (Anthropic tool schemas, Gemini `responseSchema`). Set `additionalProperties: false` everywhere — without it, the model invents fields.

```json
{
  "type": "object",
  "properties": {
    "refusal": {
      "type": "boolean",
      "description": "true iff no claim survives the entailment check or all signals tripped"
    },
    "refusal_reason": {
      "type": "string",
      "description": "Constructive, names the gap and closest-related material. Empty when refusal=false."
    },
    "answer": {
      "type": "string",
      "description": "Free-text answer. Empty string when refusal=true."
    },
    "claims": {
      "type": "array",
      "description": "Atomic claims. Each must be supported by at least one chunk_id. Empty array when refusal=true.",
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
  "required": ["refusal", "refusal_reason", "answer", "claims"],
  "additionalProperties": false
}
```

The `enum` is the load-bearing trick — the decoder physically cannot emit a chunk ID that wasn't retrieved. Without it, smaller models invent plausible-looking `[doc_42#3]` references.

For open-weights / vLLM without strict schema support: use `outlines` or `lm-format-enforcer` to apply the schema as a regex over the sampler, or logit-bias the citation-bracket tokens.

## 2. Post-hoc entailment validator (pseudocode)

Even with structured output, semantic correctness isn't free: the model can emit valid JSON with `chunk_ids: ["c3"]` while the claim has nothing to do with c3. The validator runs an NLI check (chunk → claim) and drops or rewrites unsupported claims.

```python
def validate_and_finalize(response, chunks_by_id, entailer):
    """
    entailer: callable(premise: str, hypothesis: str) -> {"label": str, "score": float}
              label in {"entailment", "neutral", "contradiction"}
              e.g. Vectara HHEM-2.1-Open (~50ms local) or Patronus Lynx
    """
    if response["refusal"]:
        return response

    surviving = []
    for claim in response["claims"]:
        premise = "\n\n".join(chunks_by_id[cid] for cid in claim["chunk_ids"])
        verdict = entailer(premise, claim["text"])
        if verdict["label"] == "entailment" and verdict["score"] >= 0.5:
            surviving.append(claim)

    if not surviving:
        return constructive_refusal(
            query=response.get("_query"),
            top_chunks=list(chunks_by_id.values())[:3],
            reason="No claim in the draft answer is entailed by retrieved chunks."
        )

    response["claims"] = surviving
    response["answer"] = rewrite_from_claims(surviving)
    return response


def constructive_refusal(query, top_chunks, reason):
    closest = top_chunks[0]
    return {
        "refusal": True,
        "refusal_reason": (
            f"I don't have enough information to answer that. "
            f"The closest related material is {closest['doc_id']}#{closest['chunk_id']}, "
            f"which discusses {closest['summary']} but does not address {query}."
        ),
        "answer": "",
        "claims": [],
    }
```

Why structural beats verbal: EACL 2026 work on systematic error taxonomies finds verbal LLM confidence is severely miscalibrated (Expected Calibration Error > 0.40). Asking the model "are you sure?" is a worse signal than running a 100ms entailment classifier.

## 3. NeMo Guardrails integration

NeMo provides `self_check_hallucination` (a SelfCheckGPT variant — sample k responses at temp=1, NLI-check the original against the samples) and `self_check_facts` when retrieved chunks are available. Adding the hallucination rail lifted unanswerable-question interception:

- gpt-3.5-turbo: 65% → 90%
- text-davinci-003: 0% → 70%
- gemini-1.0-pro: 60% → 80%

```yaml
# config.yml
rails:
  output:
    flows:
      - self check facts
      - self check hallucination

models:
  - type: main
    engine: openai
    model: gpt-4o-mini
```

```colang
# rails/output.co
define flow self check facts
  $accuracy = execute self_check_facts
  if $accuracy < 0.5
    bot refuse to respond about hallucination
    stop

define flow self check hallucination
  $is_hallucination = execute self_check_hallucination
  if $is_hallucination
    bot refuse to respond about hallucination
    stop
```

NeMo also ships native integration with Patronus AI's **Lynx** (8B and 70B variants) for hallucination detection in RAG — drop in as a custom action when you need ceiling-quality entailment.

## 4. Guardrails AI integration

Guardrails AI is a validator chain. `ProvenanceLLM` runs sentence-by-sentence entailment against retrieved sources; pair with structured output for schema-level enforcement.

```python
from guardrails import Guard
from guardrails.hub import ProvenanceLLM

guard = Guard.from_pydantic(
    output_class=RAGAnswer,         # pydantic model mirroring the schema above
    prompt=GROUNDED_PROMPT,
).use(
    ProvenanceLLM(
        validation_method="sentence",
        llm_callable="gpt-4o-mini",
        on_fail="filter",           # drop unsupported sentences; "exception" to hard-bounce
    ),
)

result = guard(
    llm_api=openai.chat.completions.create,
    model="gpt-4o",
    messages=[...],
    response_format={"type": "json_schema", "strict": True, "json_schema": schema},
    metadata={"sources": retrieved_chunks},
)
```

`provenance_v0` uses embedding similarity (sub-100ms, cheap); `provenance_v1` uses a second LLM (higher recall, ~1 LLM call/sentence). In a published sales-engagement deployment, layered guardrails (RAG + structured output + provenance validators + bounce-back loops) drove factual accuracy to **98%** with a <2% hard-bounce SLA.

## 5. Layered architecture (net)

The production stack is **layered, not single-shot** — each layer catches a different failure mode:

1. **System prompt** sets the contract in natural language ("do not invent doc_ids; only cite IDs from CONTEXT").
2. **Structured-output schema** constrains the **shape** — 100% JSON validity by construction under `strict: true`.
3. **Enum-bound chunk IDs** prevent **fabricated references** — decoder cannot emit out-of-set IDs.
4. **Post-hoc entailment validator** (HHEM / Lynx / RAGAS-faithfulness) gates the **response** — catches valid-JSON-wrong-semantics.
5. **Refusal logic** triggers on threshold misses with a **constructive fallback** — converts hallucinations into recoverable refusals.

Compounding effect is what gets a 10M-doc RAG from "occasionally wrong" to near-zero-hallucination. One extra small-model call per response; "confidently wrong" becomes a tracked, bounded, observable failure mode.
