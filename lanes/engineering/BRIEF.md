# Lane: engineering

## Scope

You read **engineering practitioners explaining real systems**: GitHub repos (README, docs, key issues/PRs), vendor engineering blogs (Pinecone, Vespa, Anthropic, Databricks, Cohere, AWS, Hugging Face, LlamaIndex, LangChain, Voyage), and enterprise case studies (Glean, Notion AI, Perplexity, You.com, Bloomberg, GitHub Copilot).

## Hard write rule

You write ONLY to `lanes/engineering/**`. You commit ONLY on branch `lane/engineering`.

## Acceptance bar (reviewer enforces)

- Source MUST include a **verifiable artifact**: either (a) a GitHub repo URL with stars and last-commit timestamp, or (b) a named production system at a real company with a public engineering write-up.
- Claims about a tool's behavior MUST cite the specific repo path or docs URL where the behavior is documented (e.g. "Qdrant uses HNSW with M=16 default per docs.qdrant.tech/concepts/search/#hnsw").
- Vendor marketing claims without numbers -> reject. Vendor benchmarks ARE acceptable but mark `confidence` <= 0.6 and look for independent confirmation.
- Out-of-scope items -> one-liner in `lanes/engineering/notes.md`.

## Tools

- `WebFetch` for blog posts, docs pages, README files at `https://raw.githubusercontent.com/<owner>/<name>/HEAD/README.md`
- GitHub API: `https://api.github.com/repos/<owner>/<name>` for stars/last-commit metadata. Rate-limited (60/hr unauthenticated) - spread requests; cache locally in your notes.
- `WebSearch` for discovery

## Batch loop

Per batch (~10 sources):

1. `cd /Users/shefalithakur/cursor-exp/rag-10m-research-engineering` (your worktree, branch `lane/engineering`)
2. `git fetch && git rebase main`
3. Read `AGENT_BRIEF_SHARED.md`, this file, and `skills/`.
4. Check `review/feedback/engineering-*.md`; address first.
5. Pick 5-10 unread sources from `sources/seed.yaml#engineering` or leads.
6. For each source:
   - Fetch the README/blog post/case study. For repos also fetch GitHub API metadata.
   - Append 1 source row to `lanes/engineering/sources.jsonl` (id format: `E-S-NNNN`). Include stars, last-commit, organization.
   - Append 1-5 claim rows to `lanes/engineering/claims.jsonl` (id format: `E-C-NNNN`). Verbatim quote required.
7. `git add lanes/engineering/`
8. `git commit -m "[lane/engineering] <summary> (+N claims, +M sources)"`

## Stop conditions

Same as papers lane (see AGENT_BRIEF_SHARED.md section 8).

## Initial focus order

Within your 2-hour budget, prioritize:

1. **Vendor blogs with concrete numbers**: Anthropic contextual retrieval (their own published before/after numbers), Pinecone scale write-ups, Vespa engineering blog (often has p95 latency at billion-scale), Cohere rerank docs (have explicit nDCG numbers).
2. **Eval frameworks**: explodinggradients/ragas (metric definitions + LLM judge sensitivity caveats), TruLens, DeepEval. Read their metric implementations, not their marketing.
3. **Vector DB internals**: Qdrant, Weaviate, Milvus, Vespa, pgvector docs. Get the HNSW parameters they default to, the quantization options they offer, the consistency model.
4. **Frameworks** (LlamaIndex, LangChain): these are aggregators - useful for surveying "what techniques are commonly used together" but light on hard numbers. Spend less time here.
5. **Enterprise case studies**: Glean, Perplexity, Bloomberg - for production-scale signal on what actually works.

## Self-improvement

Same skill-proposal protocol. Likely candidates here: "extract HNSW params from vendor docs", "compute repo health score from API metadata", "diff two vendor benchmarks for the same metric".
