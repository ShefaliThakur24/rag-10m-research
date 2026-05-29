# Rubric clauses — full text and failure modes

Each clause has: definition, what to check, failure modes seen in practice on a real production RAG research repo (lane outputs + a 617-line synthesis document), and a worked pass/fail example.

The rubric was developed by reviewing per-lane research outputs (papers, engineering, community) and a final synthesis document under a strict "would-Karpathy-merge-this" bar. The failure modes below are from that session, not invented.

---

## Clause 1 — Quote accuracy

**Definition.** Every direct quote and every numeric claim traces to a cited source. The source must be a URL, paper ID, or section reference precise enough to verify in under a minute. If the claim cannot be verified within the review budget, the claim is hedged ("reportedly", "~", "approximately") or removed entirely.

**Check.**

- Pick the numeric claims and direct quotes in the diff. For each, follow the citation. The quoted string must appear verbatim in the source (whitespace normalization allowed; paraphrase not allowed).
- Bare numbers without a footnote / inline citation default to FAIL.
- "According to [Lab/Company]" with no link defaults to FAIL.

**Failure modes seen in practice.**

- Subagents fabricated numbers like `"Anthropic reports 49% retrieval failure reduction"` with no exact source pointer. The number was plausible but not findable in the cited blog post. Fix: require a URL plus a page-or-section reference next to the number, OR hedge to `"reportedly ~50% reduction (Anthropic, contextual retrieval post)"`.
- Quoted phrase appeared in the citation in a paraphrased form, not verbatim. Treated as FAIL; either re-quote exactly or drop the quote marks and paraphrase explicitly.

**Pass.** `"bge-base-en-v1.5 reports 63.55 nDCG@10 on MTEB Retrieval (MTEB leaderboard, accessed via paper Table 3)."` — number, model, metric, source, location.

**Fail.** `"Dense embeddings cut retrieval failures by roughly half in production."` — no source, no regime, no model.

---

## Clause 2 — Citation freshness

**Definition.** Each cited source has an explicit year. Sources older than ~24 months either get a short justification for why they still apply, get refreshed against a newer source, or get downgraded to low-confidence. No "a recent paper shows..." without a year.

**Check.**

- Every citation has `(Author, YEAR)` or an equivalent dated form.
- For sources older than ~24 months in a fast-moving area (embeddings, rerankers, LLMs): expect a one-line justification or a confidence downgrade.
- "Recent", "latest", "current" are flagged on sight unless paired with a year.

**Failure modes seen in practice.**

- A 2021 retrieval paper cited as state of the art in a 2026 document with no note that the field moved on. Fix: cite a 2024+ replacement, or add `"Still the canonical reference for the dual-encoder formulation; superseded for ranking quality by ..."`.
- "A recent benchmark" with no year and no link.

**Pass.** `"BM25 baseline numbers from Robertson & Zaragoza, 2009 — still the canonical sparse retrieval reference for English news corpora."`

**Fail.** `"A recent paper showed dense retrieval wins on long documents."` — no year, no link, no regime.

---

## Clause 3 — Prose style

**Definition.** Karpathy-style: terse, dense, technically loaded, no filler, no hype, no marketing voice, no "in conclusion", no "in today's world", no rhetorical questions, no transitional throat-clearing. Every sentence earns its tokens or it gets cut.

**Check.** Auto-flag the following patterns on sight:

- `revolutionary`, `groundbreaking`, `cutting-edge`, `state-of-the-art` (used as a marketing tag rather than a benchmark statement)
- `leverage` (as a verb)
- `in today's world`, `in the modern era`, `nowadays`
- `robust solution`, `powerful framework`, `seamless`, `synergy`
- `it is important to note that`, `it is worth mentioning`
- `in conclusion`, `to sum up`, `as we have seen`
- Filler adverbs: `very`, `quite`, `extremely`, `really` (as intensifiers, not measurements)
- Rhetorical questions that the next sentence answers

**Failure modes seen in practice.**

- Early lane outputs opened with `"In today's rapidly evolving AI landscape, retrieval-augmented generation has emerged as a revolutionary approach..."` — every word a token tax. The same content compressed to one technical sentence: `"RAG conditions an LLM on retrieved documents to ground its output."`
- Conclusions that restated the document.

**Pass.** `"Reranking with a cross-encoder buys ~5-12 nDCG@10 over the bi-encoder baseline at a 30-80ms latency cost per query (BGE reranker base, batch=1, L4)."`

**Fail.** `"Reranking is a powerful technique that can be leveraged to significantly improve retrieval quality in a robust manner."`

---

## Clause 4 — Tradeoff specificity

**Definition.** Any claim of the form "X is better than Y" must come with the regime where the claim holds: data scale, query distribution, latency budget, model class, recall floor, or cost ceiling. A vague "it depends" is a failure, not an answer. State the regime, or do not make the claim.

**Check.**

- "Better", "worse", "outperforms", "preferred" without an `applies_when` clause → FAIL.
- "It depends" answers → FAIL. Either resolve the dependence with regimes or remove the comparison.
- Tradeoff tables where the `wins_when` column says "general use" or is blank → FAIL.

**Failure modes seen in practice.**

- `"Dense embeddings outperform sparse retrieval."` Fails on sight — no model, no corpus, no metric. Fix: `"For queries with paraphrase semantics on >1M document corpora, dense embeddings (e.g., bge-base-en-v1.5) outperform BM25 by ~8-15 nDCG@10 (MTEB Retrieval avg)."`
- `"Larger chunks are usually better for long-form answering."` Fails — "usually" with no regime. Fix: state corpus type and chunk-size range tested.

**Pass.** `"For sub-200ms p95 latency budgets on a single L4 GPU, bi-encoder retrieval + cross-encoder rerank on top-50 outperforms top-200 + rerank by 3-4 nDCG@10 at half the rerank cost."`

**Fail.** `"Cross-encoders are generally better than bi-encoders for ranking."`

---

## Clause 5 — Audience fit

**Definition.** The document is calibrated to a stated audience. For an advanced production audience (the default for ML design docs and research synthesis), skip 101 explanations of vector DBs, embeddings, chunking, transformer basics, attention mechanisms, or "what is RAG". Foundational definitions go in an appendix or get linked out.

**Check.**

- Any inline explanation of a concept the audience is assumed to know → FAIL or push to appendix.
- Concept explanations longer than ~4 sentences inline (not in an appendix) → FAIL.
- Tutorial scaffolding ("first, we'll cover...", "in this section we will...") → FAIL on advanced audiences.

**Failure modes seen in practice.**

- Lane outputs occasionally re-explained what an embedding is in the body of a design doc aimed at ML engineers. Fix: cut the paragraph or move it to `appendix/concepts.md` and link.

**Pass.** Doc body says `"Use a bi-encoder for first-stage retrieval (see appendix/concepts.md if unfamiliar)."`

**Fail.** Doc body says `"An embedding is a fixed-dimensional dense vector representation of a piece of text, typically produced by a transformer encoder. Embeddings allow us to..."`.

---

## Clause 6 — Internal consistency

**Definition.** Terminology, units, model versions, dataset names, and numbers match across the document. If chunk size is "512 tokens" in one section, it is not "~500 tokens" or "half a page" elsewhere. Citation format is consistent.

**Check.**

- Same entity referred to by one canonical name throughout (model versions, dataset names, paper authors).
- Units consistent: tokens vs characters vs words, ms vs seconds, USD vs cents.
- Numbers consistent across the body, the tables, and the appendix.
- Citation format consistent.

**Failure modes seen in practice.**

- The same paper cited as `"Chen et al. 2024"` in one section and `"Chen 2024"` in another. Reviewer flagged it as a single inline comment, did not block merge. Treat as COMMENT-WITH-NOTES unless it occurs throughout.
- `"chunk_size=512"` in the methodology section, `"~500-token chunks"` in the results section. Pick one and use it everywhere; this often hides a real bug.

**Pass.** Chunk size is `512 tokens` in body, tables, jsonl, and appendix; model is `bge-base-en-v1.5` everywhere.

**Fail (blocking).** Latency reported as `80ms` in body, `0.12s` in the comparison table — different number, not just different unit.

---

## Clause 7 — Image necessity

**Definition.** A diagram exists if and only if (a) the equivalent prose would exceed ~150 words, or (b) the relationship is fundamentally spatial (geometry, architecture topology, pipeline shape). No decorative images. No images of bullet lists. No screenshots of tables that should be markdown tables.

**Check.**

- For each new image: can a 3-bullet list replace it? If yes → FAIL.
- Is the image a screenshot of text that could be inline markdown? → FAIL.
- Does the caption state the takeaway in one paragraph? If absent → request caption.

**Failure modes seen in practice.**

- In the session that produced this rubric, 6 diagrams were kept (pipeline topology, latency-cost frontier, two-stage retrieval shape, etc.) and 2 proposed diagrams were rejected because the relationship was a flat 3-bullet list and a small ordered enumeration. Both became inline lists; doc improved.

**Pass.** A two-stage retrieve+rerank architecture diagram with a one-paragraph caption stating the latency-vs-recall tradeoff illustrated.

**Fail.** A diagram of "the three benefits of using a reranker" rendered as three boxes with arrows pointing nowhere.

---

## Clause 8 — Schema compliance

**Definition.** Machine-readable artifacts (`.jsonl`, `.yaml`, `.csv`, `.toml`) parse cleanly and conform to the declared schema. One bad line fails the whole file.

**Check.**

- Run the parser. `jq -c . file.jsonl > /dev/null`, `yq . file.yaml > /dev/null`, etc. Any parse error → FAIL on the file.
- Required fields present. Run a `jq -e 'has("id") and has("evidence") and has("confidence")'` (or equivalent) across all entries; any line missing required fields → FAIL.
- Enum fields hold declared values; numeric fields are numeric.
- Schema referenced from the document matches the file; if the schema lives in a separate file, the version pinned in the artifact matches it.

**Failure modes seen in practice.**

- Single line in a 200-line `claims.jsonl` had a stray trailing comma from a hand-edit, broke the whole file. Treat as blocking; the lane is asked to re-emit the file. No partial merges.
- `confidence` field encoded as `"high"` in one row, `0.8` in another. Pick numeric or enum; do not mix.

**Pass.** `jq -c . lanes/papers/claims.jsonl | wc -l` matches `wc -l`, every line has all required fields, every `confidence` is a float in `[0, 1]`.

**Fail (blocking).** Even one unparseable line, or even one row missing a required field.
