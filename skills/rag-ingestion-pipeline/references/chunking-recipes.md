# Chunking recipes

Per-source-type chunking defaults, with the regime where each splitter wins. The 7-decision table in `SKILL.md` names `token-aware ~512 tokens / 10-15% overlap on semantic boundaries` as the global default; this file is the override table when that default doesn't fit the source.

## Splitter selection (when each wins)

| Splitter | Wins when | Loses when | Quantified note |
|---|---|---|---|
| **Token-aware (default)** — split at ~512 tokens, snap to nearest paragraph / heading / sentence, 50-80 token overlap (10-15%) | Prose, docs, articles, manuals, support tickets, transcripts. Faithfulness-gated workloads (RAGAS-class metrics ≥ 0.95). | Code / log files (no paragraph structure). Source already self-contained at chunk granularity (e.g., one SEC filing = one chunk). | 512-token chunks score faithfulness **97.59 / relevancy 97.41** on `lyft_2021` doc-QA (ada-002 + GPT-3.5/Zephyr-7B); 1024-token drops to 94.26 / 95.56; 2048-token collapses to **80.37 / 91.11**. |
| **Sentence-window** — embed one sentence at a time, retrieve sentence ± N neighbors as context window | Highly precise factual lookup (FAQs, regulatory clauses) where the answer is one sentence and surrounding context must be returned verbatim. | Multi-sentence reasoning. BM25 signal weakens with short chunks. | Sentence-window is empirically strong on benchmarks dominated by single-sentence answers; weaker on aggregation/multi-hop. Pair with parent-document return so the generator sees enough context. |
| **Recursive character** — split by the largest separator that fits (`\n\n` → `\n` → `. ` → ` `), recurse if the chunk is still too big | Code (functions, classes), log files, configuration, anything without consistent paragraph boundaries. | Prose with strong paragraph structure (use token-aware instead — recursive over-splits or under-splits depending on file formatting). | LangChain's `RecursiveCharacterTextSplitter` default. Use language-aware separators (`def `, `class `, `}` for code). |
| **Semantic / topic-segmented** — split where embedding similarity between adjacent sentences drops below a threshold (e.g., 0.5 cosine) | Long heterogeneous documents (research papers, contracts, multi-section reports) where natural topic boundaries dominate sentence boundaries. | Cost — every sentence pair needs an embedding. Use only when the topic-shift structure is the actual signal. | Empirically lifts retrieval precision a few points over fixed-size token splits on mixed-topic corpora; cost is the binding constraint at 10M+ docs. Run as a post-pass on candidate chunks rather than every sentence. |
| **Sliding-window** — fixed-size token window stepped at fixed stride | Streaming text (transcripts, chat logs) where boundaries are time-based, not semantic. | Boundary-aware sources. Wastes embedding budget on near-duplicate overlapping chunks. | Overlap = stride/2 is the typical setting. Acceptable only when no semantic boundary exists. |

## Per-source recipes

### Code and log content

- **Splitter**: recursive character with **language-aware separators** (`\n\nclass `, `\n\ndef `, `\n\nfunction `, `\n}` for typical OO/functional code; `\n[ERROR]`, `\n[INFO]`, ISO-8601 timestamp regex for logs).
- **Chunk size**: 800-1500 chars for code (one function or method); 50-100 lines for logs (one event group).
- **Overlap**: minimal (≤5%) — code rarely benefits from overlap; function boundaries are clean.
- **Embedding model**: domain-specific code embedder beats generalist by a generation. **`voyage-code-3`** is +13.80% NDCG@10 average vs OpenAI 3-large across 32 code datasets, +14.64% at 1024 dim, +17.66% at 256 dim. Self-host alternative: `bge-large-en-v1.5` is adequate for English-language code with comments.
- **Metadata to attach**: file path, language, symbol name (function/class), line range.

### HTML

- **Pre-pass**: strip nav / header / footer / ads with a readability library (`trafilatura`, `readability-lxml`, or `unstructured.io`'s HTML partitioner). Skipping this step floods the index with boilerplate that wins BM25 on every query.
- **Splitter**: token-aware at ~512 tokens, with **DOM-aware boundary preference**: `<h1>` > `<h2>` > `<h3>` > `<p>` > sentence.
- **Tables**: extract separately. Convert to markdown or JSON-row form. Embed each row (or each table) as its own chunk with the table caption as context.
- **Metadata**: source URL, last-modified, language, heading path (e.g., `Docs > API > Authentication`).

### PDF

- **Parse first** — the parse step is the bottleneck (every research lane in `doc/01-approach.md` and `FINAL.md` flagged it). Candidates: Unstructured.io (open-source, broad coverage), AWS Textract / Azure Document Intelligence (managed OCR + table-aware), Marker / Nougat (academic PDFs), Camelot / Tabula (tables only).
- **Splitter**: token-aware at ~512 tokens, **respect page and section boundaries**. Do not split mid-paragraph across page breaks.
- **Tables / figures**: extract separately; chunk on row-group + caption. Treat figure captions as their own chunks with image references in metadata.
- **Scanned PDFs**: OCR before chunking; OCR confidence is a useful metadata field for downstream confidence weighting.
- **Metadata**: source file, page range, section heading, table/figure ID if applicable.

### Markdown

- **Splitter**: token-aware at ~512 tokens with `#`-level heading boundaries. `MarkdownTextSplitter` (LangChain) and `MarkdownHeaderTextSplitter` are sensible starting points.
- **Heading inheritance**: prepend the heading chain to each chunk (e.g., `# Section / ## Subsection / ### Detail`). This is the cheapest manual approximation of contextual augmentation when LLM-based contextualization is not affordable.
- **Code blocks**: keep inline if short (< 100 tokens); split out as code chunks if longer (use the code recipe).
- **Metadata**: file path, heading path, language tag for any code blocks inside.

## Anthropic contextual chunk prompt (verbatim)

The default in `SKILL.md` says **enable contextual chunk augmentation**. The Anthropic Contextual Retrieval prompt is short and load-bearing — copy it directly.

```text
<document>
{{WHOLE_DOCUMENT}}
</document>

Here is the chunk we want to situate within the whole document:

<chunk>
{{CHUNK_CONTENT}}
</chunk>

Please give a short succinct context to situate this chunk within
the overall document for the purposes of improving search retrieval
of the chunk. Answer only with the succinct context and nothing else.
```

**Operational notes:**

- **Cache the document.** Use Anthropic prompt caching (`cache_control: { type: "ephemeral" }` on the `<document>` block) so the document is paid for once per indexing batch rather than once per chunk. This brings the marginal cost per chunk to ~$0.001 on `claude-3-haiku` / `claude-3-5-haiku` class models. Without caching, document re-tokenization dominates the bill.
- **Prepend, don't replace.** The model's output (50-100 token context) is **prepended** to the original chunk, not used in place of it. Both BM25 and the dense embedder index the concatenation: `context + "\n" + chunk_text`. The Anthropic eval (5.7% → 2.9% retrieval failure) is on the concatenated form.
- **Index both lexical and dense.** "Contextual Embeddings + Contextual BM25" means **both** indexes are built on the contextualized chunks. Augmenting only the dense index gives a smaller lift (5.7% → 3.7%, the -35% number); doing both gets the -49% number.
- **One-shot, no gleaning.** Single LLM call per chunk. No iterative refinement. At 10M docs × ~5 chunks/doc = 50M LLM calls; budget ~$10-50k on Haiku-class. Skip contextualization on chunks that are already long and self-contained (e.g., chunks > 1500 tokens — they rarely need outer-document context).
- **Re-run on re-embed.** When you change embedding models, re-run contextualization only if the document corpus has changed; the contextualized chunks are model-agnostic.

## Overlap sizing

- **Default**: 10-15% (~50-80 tokens for 512-token chunks).
- **Why overlap at all**: avoids splitting an answer span across a chunk boundary.
- **Too much overlap** (≥30%) bloats the index, inflates BM25 false positives (the same span scores high on N adjacent chunks), and wastes embedding cost.
- **Too little overlap** (<5%) loses answer spans that straddle boundaries. The sentence-window pattern (one-sentence chunks, parent-document return at query time) is a different design that obviates overlap entirely.

## Chunk-level metadata to always store

Independent of source type, attach this metadata at index time. Most reranker and citation patterns assume it exists.

- `doc_id` — stable document identifier (URL, file hash, primary key).
- `chunk_id` — stable chunk identifier within `doc_id` (e.g., `doc_42#chunk_007`).
- `chunk_index` — 0-based ordinal of the chunk in the document (for sentence-window / parent-doc retrieval).
- `source_uri` — original source path or URL.
- `last_modified` — for freshness-weighted ranking.
- `language` — for language-routing.
- `heading_path` / `section` — for boundary-aware citation.
- `char_start` / `char_end` — for anchor-style citation (quoted span → chunk offset).
- Any pre-extracted entities or document-level tags relevant to filter queries.

The `char_start` / `char_end` fields are the foundation of Anthropic Citations API-style anchor citations; without them the generator can cite at chunk granularity only, not at quoted-span granularity.
