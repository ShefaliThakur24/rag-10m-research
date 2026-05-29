# RAG-10M Research

A self-contained, multi-agent research repo converging on a production-grade design document for "10M+ documents, near-zero hallucination" RAG.

## How to read the output

The deliverable is the `doc/` tree. Read in this order:

1. [doc/00-problem.md](doc/00-problem.md) - the problem framing (success criteria, non-goals, target audience).
2. [doc/02-tradeoffs.md](doc/02-tradeoffs.md) - the explicit tradeoff matrices for each pipeline stage.
3. [doc/01-approach.md](doc/01-approach.md) - the recommended pipeline end-to-end with rationale.
4. [doc/03-rejected.md](doc/03-rejected.md) - approaches considered and rejected, one-line each.
5. [appendix/concepts.md](appendix/concepts.md) - 2-4 sentence refs for any technical term mentioned in the docs above.

Audience: senior ML / staff engineers shipping production RAG. The docs assume you know vector DBs, embeddings, chunking, basic RAG. They do **not** explain these.

## How the repo operates

5 agents work in parallel:

- **3 lane agents** (papers / engineering / community) gather sources, write per-lane `sources.jsonl` and `claims.jsonl`, commit to their own branch.
- **1 synth agent** runs every 30 min, reads the unified state from `main`, dedupes claims, updates `doc/` and `appendix/`, generates diagrams.
- **1 reviewer agent** runs every 5 min (Karpathy-tier rubric), evaluates new commits on lane/* and synth/main, ff-merges to `main` on pass or writes feedback file on fail.

Reviewer is the only writer to `main`, `review/`, and `skills/rag10m-*/`. Lanes never touch synth's files; synth never touches lanes' files. Zero merge conflicts by construction.

## Branches

- `main` - reviewed state. Reviewer ff-merges into this.
- `lane/papers`, `lane/engineering`, `lane/community` - per-lane working branches.
- `synth/main` - synth's working branch.

## Worktrees (sibling directories)

- `../rag-10m-research-papers/` (branch `lane/papers`)
- `../rag-10m-research-engineering/` (branch `lane/engineering`)
- `../rag-10m-research-community/` (branch `lane/community`)
- `../rag-10m-research-synth/` (branch `synth/main`)
- Reviewer operates directly from the main repo dir at `rag-10m-research/`.

## Key files

- [AGENT_BRIEF_SHARED.md](AGENT_BRIEF_SHARED.md) - standing orders for all 5 agents (audience+tone, write-isolation, schemas, image policy, skill-proposal protocol).
- [CONVERGENCE.md](CONVERGENCE.md) - the Shape A -> B trigger (tuned for a 2-hour run).
- [reviewer/BRIEF.md](reviewer/BRIEF.md) - the Karpathy-tier review rubric.
- [synth/BRIEF.md](synth/BRIEF.md) - per-tick synthesis algorithm.
- [lanes/papers/BRIEF.md](lanes/papers/BRIEF.md), [lanes/engineering/BRIEF.md](lanes/engineering/BRIEF.md), [lanes/community/BRIEF.md](lanes/community/BRIEF.md).
- [publish/BRIEF.md](publish/BRIEF.md) - convergence-time publish procedure.
- [sources/seed.yaml](sources/seed.yaml) - the pre-curated seed crawl list.

## Environment notes

- **MCPs**: arxiv, firecrawl, github, and Google Docs MCPs are NOT installed in this environment. Agents fall back to built-in `WebFetch` + `WebSearch` for crawling, and `publish` writes a concatenated `EXPORT.md` to repo root instead of pushing to Google Docs. See lane BRIEFs for explicit endpoints (arxiv HTTP API, GitHub HTML, Reddit `.json` suffix).
- **Skills location**: accepted skills are written to `skills/rag10m-<slug>/SKILL.md` inside this repo (not at the workspace root, because Cursor reserves `.cursor/` at workspace root). Lanes/synth read `skills/` at the start of each batch.

## Stop condition

- `CONVERGENCE.md` triggers, OR
- 2-hour wall-clock budget exhausted (lanes self-terminate, reviewer dispatches publish).

## Operating dashboard

Live operational state is in:

- `git log --all --oneline --decorate -20` - branch tips and recent activity.
- `review/log/` - one file per reviewed commit, audit trail.
- `review/feedback/` - any open feedback files (lanes address these on the next batch).
- `synth/open-questions.md` - questions the agents need a human to answer (you edit in place; next synth tick incorporates).
- `CHANGELOG.md` - human-readable summary of every synth tick.
- `skills/` - accepted skills (each subdir = one skill).

To course-correct a lane: stop the lane subagent, then re-launch with an updated prompt.

To stop everything: kill the two `/loop` sleeper PIDs (printed at bootstrap end), then stop the lane background subagents.
