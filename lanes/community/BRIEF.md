# Lane: community

## Scope

You read **community signal**: Reddit threads (r/LocalLLaMA, r/MachineLearning, r/LangChain, r/Rag), Hacker News stories+top comments, individual practitioner blogs (Eugene Yan, Hamel Husain, Phil Schmid, Simon Willison, Jason Liu, Douwe Kiela), conference talk write-ups.

## Hard write rule

You write ONLY to `lanes/community/**`. You commit ONLY on branch `lane/community`.

## Acceptance bar (HIGH - signal:noise is worst in your scope, so the bar is strictest)

- Source MUST surface a **novel observation, gotcha, or benchmark** that hasn't appeared in the papers or engineering lanes. Use `git log --all --oneline | rg <topic>` to check before claiming novelty.
- "X is hard" without specifics -> reject yourself, don't even commit it.
- A practitioner's stated numbers count, but tag `confidence` <= 0.5 unless they show their methodology.
- Forum debates and opinions -> only useful as signal about what's contested in practice. Record as a claim only if you can name a specific implementation issue or counter-example. Otherwise -> notes.md.
- Out-of-scope items -> one-liner in `lanes/community/notes.md`.

## Tools

- Reddit JSON: append `.json` to any reddit URL (e.g. `https://www.reddit.com/r/LocalLLaMA/top/.json?t=year&limit=100`)
- HN Algolia: `https://hn.algolia.com/api/v1/search?query=<terms>&tags=story`
- `WebFetch` for individual blogs
- `WebSearch` for discovery

## Batch loop

Per batch (~10 sources):

1. `cd /Users/shefalithakur/cursor-exp/rag-10m-research-community` (your worktree, branch `lane/community`)
2. `git fetch && git rebase main`
3. Read `AGENT_BRIEF_SHARED.md`, this file, and `skills/`.
4. Check `review/feedback/community-*.md`; address first.
5. Fetch a batch from one community source (Reddit subreddit JSON, HN query, or a practitioner's index page).
6. Skim ~20 items, pick the 5-10 with concrete content.
7. For each kept item:
   - Append source row to `lanes/community/sources.jsonl` (id `C-S-NNNN`).
   - Append claim rows ONLY if the claim survives the acceptance bar (most won't). Aim for high signal per claim.
8. `git add lanes/community/`
9. `git commit -m "[lane/community] <summary> (+N claims, +M sources)"`

## Stop conditions

Same as papers lane.

## Initial focus order

1. **Practitioner blogs first**: Eugene Yan, Hamel Husain, Jason Liu - they distill community signal into structured posts, much higher signal per minute than forums.
2. **r/Rag and r/LocalLLaMA top-of-year** for production gotchas and which tools people actually ship.
3. **HN stories with substantive comment threads** on vector DB / RAG topics - the top comments often have hard-won numbers.
4. r/MachineLearning and r/LangChain - more noise; sample sparingly.

## Self-improvement

Same protocol. Likely candidates here: "extract benchmark numbers from a blog post body", "score a Reddit thread for signal vs heat".
