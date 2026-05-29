# Publish brief

You are spawned by the reviewer at convergence (or on-demand) to produce the final deliverable.

## Environment note (DEGRADED PATH)

Google Docs MCP is NOT installed in this environment. You take the fallback path: assemble a single concatenated `EXPORT.md` at repo root, with all images inlined and internal links rewritten.

If a Google Docs MCP becomes available before convergence, swap to it: convert each top-level doc file to a Google Doc, preserve appendix anchors, upload images. Return Doc URLs to chat.

## Your task (one pass)

1. `cd /Users/shefalithakur/cursor-exp/rag-10m-research` (operate from main).
2. Concatenate (in order):
   - `doc/00-problem.md`
   - `doc/01-approach.md`
   - `doc/02-tradeoffs.md`
   - `doc/03-rejected.md`
   - `appendix/concepts.md`
   - `appendix/evals.md`
   - `appendix/glossary.md`
3. Rewrite cross-file links so they work in a single doc: convert `../appendix/concepts.md#anchor` and `appendix/concepts.md#anchor` -> `#anchor` (anchors are already unique because concept anchors are alphabetical slugs).
4. Inline images: keep `![alt](doc/images/<file>.png)` references (markdown viewers resolve relative to EXPORT.md location at repo root, so update paths to `doc/images/<file>.png` everywhere).
5. Prepend a one-page executive summary you write yourself based on `doc/01-approach.md` (no more than 30 lines: problem, top 3 recommendations, top 3 caveats, where to look next).
6. Append metadata footer: total claims in `synth/claims.jsonl`, total sources in `synth/sources.jsonl`, run duration, number of synth ticks, number of reviewer pass/fail decisions.
7. Write to `EXPORT.md` at repo root.
8. `git add EXPORT.md && git commit -m "[publish] EXPORT.md generated at convergence"` on main.
9. Print to stdout: `EXPORT_PATH=/Users/shefalithakur/cursor-exp/rag-10m-research/EXPORT.md` and the metadata footer.
10. Exit.

## What you do NOT do

- Push to Google Docs (MCP unavailable - the fallback IS the deliverable for this run).
- Modify the source `doc/`, `appendix/`, or `synth/` files. EXPORT.md is a derivation, not a replacement.
- Add new content. If a section is empty (`_TBD_`), keep it empty - the exec summary should call out gaps explicitly.
