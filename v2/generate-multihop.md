# Multi-Hop Reasoning and Verify-Loops for Production RAG

The compositionality bound established earlier (GPT-4 fails 3×3 multiply, accuracy collapses past ~3 implicit hops) is the *symptom*. The *fix* is mechanical: never let a single forward pass own a multi-hop answer. Decompose, retrieve between steps, verify before returning. This section operationalizes that for a 10M+ corpus where one bad hop poisons the rest.

## 1. Multi-Hop Benchmarks: What They Actually Stress

Pick benchmarks by failure mode, not headline number. The four below isolate distinct weaknesses.

| Benchmark | Hops | What it stresses | Current SOTA | Notes |
|---|---|---|---|---|
| **HotpotQA** (distractor) | 2 | Supporting-fact selection from 10 paragraphs; answer + sup-fact joint EM | Beam Retrieval (single model) **72.69 EM / 85.04 F1 Ans, 50.53 EM / 77.54 F1 Joint** | [hotpotqa.github.io](https://hotpotqa.github.io/) — saturated for 2-hop bridge questions; sup-fact joint metric is the real signal |
| **HotpotQA** (fullwiki) | 2 | Open-domain retrieval over all of Wikipedia | AISO **67.46 EM / 80.52 F1** | Retrieval-bound, not reasoning-bound |
| **2WikiMultiHopQA** | 2–4 | Comparison + composition + bridge; explicit reasoning chains from Wikidata | HippoRAG 2 R@5 **90.4** ; HGRAG QA F1 **78.3** | [aclanthology 2020.coling-main.580](https://aclanthology.org/2020.coling-main.580/) — chain annotations make it the cleanest decomposition eval |
| **MuSiQue-Ans** | 2–4 | Composed from single-hop questions to *minimize shortcuts* and pretraining leakage | HippoRAG 2 R@5 **74.7** ; HGRAG F1 **53.8** | [arxiv 2108.00573](https://arxiv.org/abs/2108.00573) — strongest "did your retriever actually traverse?" signal |
| **StrategyQA** | implicit (2–3) | Yes/no questions with *unstated* decomposition steps | Zero-shot-PS+ **65.4** vs Zero-shot-CoT 63.8 (Wang 2023, GPT-3.5-class) | [aclanthology 2023.acl-long.147](https://aclanthology.org/2023.acl-long.147.pdf) — measures whether the model *infers* the hop structure |

**Production rule:** if you're shipping over 10M+ docs, MuSiQue-Ans is the one to gate on. HotpotQA-distractor lies — gold paragraphs in a 10-doc context is not your retrieval problem. Self-evaluate on MuSiQue + your own held-out 2–4 hop set.

## 2. Query Decomposition Techniques

All five techniques below convert "answer in one shot" into "decompose, retrieve per hop, compose."

| Technique | Mechanism (1 line) | Headline delta vs single-pass |
|---|---|---|
| **Self-Ask** (Press et al. 2022) | Model emits explicit `Follow up:` / `Intermediate answer:` lines; optional search-engine plug-in for sub-Qs | Closes ~40% of compositionality gap on Bamboogle/Compositional Celebs vs CoT; gap *does not* shrink with scale alone [arxiv 2210.03350](https://arxiv.org/abs/2210.03350) |
| **Decomposed Prompting** (Khot et al. 2022) | Task-specific decomposers route sub-tasks to specialized handlers; modular and recursive | Wins where sub-tasks need different tools (math vs lookup); designed to be composable |
| **Least-to-Most** (Zhou et al. 2022) | Few-shot examples teach sequential decomposition: easiest sub-Q first, feed answer forward | Strong on compositional generalization (SCAN-style splits); fragile on the boundary between decomposition and execution prompts |
| **Plan-and-Solve / PS+** (Wang et al. 2023) | Zero-shot: emit a plan first, then execute; PS+ adds explicit calc/extraction guards | StrategyQA **65.4 vs 63.8** Zero-shot-CoT; calculation errors 5% vs 7%, missing-step errors 7% vs 12% [aclanthology 2023.acl-long.147](https://aclanthology.org/2023.acl-long.147.pdf) |
| **IRCoT** (Trivedi et al. 2023) | *Each CoT sentence becomes a retrieval query*; alternate generate-one-step / retrieve / append | Recall@k **+11–21 pts**; QA F1 **+7.1 HotpotQA, +13.2 2Wiki, +5–9 MuSiQue**; **factual errors in CoT cut by 50% (HotpotQA) and 40% (2Wiki)** [aclanthology 2023.acl-long.557](https://aclanthology.org/2023.acl-long.557.pdf) |
| **HippoRAG / HippoRAG 2** (Gutiérrez et al. 2024–25) | OpenIE → entity graph + Personalized PageRank for one-shot multi-hop traversal (no per-hop LLM call) | Single-step HippoRAG matches IRCoT QA at **10–30× lower cost, 6–13× lower latency**; HippoRAG 2 R@5: **MuSiQue 74.7, 2Wiki 90.4, HotpotQA 96.3** [proceedings.neurips.cc/HippoRAG](https://proceedings.neurips.cc/paper_files/paper/2024/file/6ddc001d07ca4f319af96a3024f6dbd1-Paper-Conference.pdf), [arxiv 2502.14802](https://arxiv.org/html/2502.14802v2) |

**Pattern that wins at 10M+ scale:** structure-augmented retrieval (HippoRAG-class graph traversal) for the *retrieval* hops, IRCoT-style interleaving for the *reasoning* hops, with a hard cap of 4–5 retrieve-reason rounds. Pure prompt-only Self-Ask saturates around 2 hops; you need retrieval *between* steps once your corpus exceeds a few million.

## 3. Chain-of-Verification (CoVe)

Dhuliawala et al. 2023 [arxiv 2309.11495](https://arxiv.org/abs/2309.11495) defines a 4-step verify-loop:

1. **Draft** — generate an initial answer.
2. **Plan verification questions** — emit independent fact-check questions targeting the draft's claims.
3. **Answer verifications independently** — critical: each verification Q is answered in a *fresh context* without the draft, to avoid the model rationalizing its own hallucinations.
4. **Revise** — regenerate the final answer conditioned on the verification answers.

Numbers (Llama-65B base):

| Task | Baseline | CoVe (factored) | Delta |
|---|---|---|---|
| MultiSpanQA F1 | 0.39 | **0.48** | **+23%** |
| Wikidata list precision | 0.17 | **0.36** | **2.1×** |
| Longform FactScore | 55.9 | **71.4** | +15.5 pts |

**"Factored" beats "joint."** Answering verification questions *separately* (factored) beats answering them in one batched prompt (joint), because joint prompting lets the draft contaminate verification.

**When CoVe helps:** list-style and entity-heavy answers (Wikidata-flavor), longform where each sentence is a verifiable claim.
**When CoVe hurts:** (a) latency-sensitive paths — 4 sequential LLM calls, ~3–4× cost; (b) low draft quality — if the draft misses an entity entirely, no verification question gets generated for it; (c) tasks where the verification question is just as hard as the original (single-hop trivia). Don't run CoVe on every query — gate it on a confidence/uncertainty signal or on response type (claims-list vs short answer).

## 4. Self-RAG and Adaptive Retrieval

Self-RAG (Asai et al. 2023, [arxiv 2310.11511](https://arxiv.org/html/2310.11511v1)) bakes the retrieve/verify decisions *into the decoder* via four reflection-token classes:

- `Retrieve` — should we retrieve right now? (per segment, not per query)
- `IsRel` — is this passage relevant to the prompt?
- `IsSup` — is the generated segment fully / partially / not supported by the passage?
- `IsUse` — overall utility 1–5 of the response

**Training data construction:** a GPT-4 *critic* labels reflection tokens on instruction-tuning data → distilled into a 7B/13B critic, which then re-labels a larger corpus → generator trained with standard next-token loss over the augmented sequence. So a single LM emits both content and its own meta-tokens at inference; tree-decoding picks the segment with the best weighted IsRel × IsSup × IsUse score.

**Headline numbers** (Self-RAG-13B vs baselines):

| Task | Llama2-13B | Alpaca-13B | ChatGPT | **Self-RAG-13B** |
|---|---|---|---|---|
| PubHealth (acc) | — | 51.1 | — | **74.5** |
| ARC-Challenge | 29.4 | 57.6 | — | **73.1** |
| PopQA | 14.7 | 24.4 | — | **55.8** |
| TriviaQA | 47.0 | 66.9 | — | **69.3** |
| ASQA citation precision | — | — | — | **70.3** |

**Inference cost:** ~1.5–2× a vanilla RAG decode at 13B because of beam-style segment scoring, but you skip retrieval entirely on segments where `Retrieve=No` — net win on mixed-workload traffic where ~30–50% of segments don't need retrieval.

## 5. ReAct + Agentic Retrieve-on-Demand

ReAct (Yao et al. 2022, [arxiv 2210.03629](https://arxiv.org/pdf/2210.03629v3)) interleaves `Thought:` / `Action:` / `Observation:` traces. On HotpotQA EM: Standard 28.7, CoT 29.4, Act-only 25.7, **ReAct 27.4**, **ReAct+CoT (best-of) 35.1**; FEVER 60.9. The combined ReAct↔CoT fallback policy is what actually ships — pure ReAct under-uses parametric knowledge.

Production stacks built on this: **LangGraph** (state-machine over tools, checkpointing), **AutoGen** (multi-agent conversation), **OpenAI Assistants / Responses API** (managed tool loop). Same conceptual pattern; differences are in state persistence and supervisor topology.

**Failure modes at 10M+ scale:**
- *Cycles* — agent re-issues near-identical retrieval queries forever. Mitigation: query-hash dedup + cosine-sim rejection (>0.95) on consecutive sub-queries.
- *Runaway tool calls* — agent burns tokens on irrelevant lookups. Mitigation: per-trace tool-call budget (e.g. 8 retrievals max).
- *Infinite plan-revise* — verifier always rejects, generator always patches. Mitigation: monotone-improvement requirement (each revision must clear a stricter scoring threshold, else escalate).
- *Context bloat* — observation traces blow past the model's effective-context window long before the hard limit. Mitigation: summarize observations older than N steps; keep raw text only for the last 2 hops.

**Concrete guardrails to wire in day one:**

| Guardrail | Default | Source |
|---|---|---|
| Max graph iterations / recursion | **20–25** (LangGraph default 25) | [langchain-ai/langgraph#recursion_limit](https://langchain-ai.github.io/langgraph/) |
| Tool-call rate limit | ≤ 8 retrievals + 4 verifications per query | per ReAct paper trajectory analysis |
| Per-trace token budget | hard kill at 32k input + 4k output | cost ceiling |
| Escape-hatch refusal | if verification IsSup < threshold after 2 revisions, return "insufficient evidence" + cite top-k passages | mirrors Self-RAG's IsSup gate |
| Cycle detector | reject sub-query if cosine-sim > 0.95 to any prior sub-query in trace | prevents prompt-loop pathology |
| Wall-clock SLA | p99 ≤ 12s; preempt agent → fall back to single-pass RAG with high-recall retrieval | latency contract |

The escape-hatch refusal is the single highest-leverage guardrail: it converts "confidently wrong multi-hop answer" (the worst failure for a hallucination-sensitive system) into "I don't know, here are the candidate documents" (a recoverable failure that surfaces to users honestly).

## TL;DR for the Pipeline

1. **Decompose, don't compose.** Use IRCoT-style interleaving for retrieval + reasoning; HippoRAG 2 for cheap structured traversal where graph extraction is feasible.
2. **Verify, don't trust.** Apply factored CoVe to claim-list / longform answers; Self-RAG-style IsSup gating on every retrieval-grounded segment.
3. **Bound the loop.** ReAct gives you the trace; only guardrails (iter cap, cycle detector, escape-hatch refusal) keep it from torching tokens at scale.

Headline numbers worth memorizing: **IRCoT cuts CoT factual errors by ~50%**, **CoVe lifts MultiSpanQA F1 by 23%**, **HippoRAG 2 hits 74.7 / 90.4 / 96.3 R@5 on MuSiQue / 2Wiki / HotpotQA** at 10–30× lower cost than IRCoT.
