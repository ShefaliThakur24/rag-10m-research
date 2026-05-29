# Notes (lane: community)

Free-form working notes. Append-only.

## Deadline

- Session started: 2026-05-29 10:43 UTC (scaffold commit 4969ee3)
- Local wallclock at lane start: 2026-05-29 10:43 UTC (Fri, 16:13 IST)
- Hard deadline: 2026-05-29 12:43 UTC (= scaffold + 2h)
- Stop conditions: deadline reached OR 60 sources processed OR `STOP-ALL-LANES` in main OR `[shape-a] converged` in main.

## v2-relaunched-at 2026-05-29 11:46 UTC

v1 hung silently before any source/claim/commit was written (sources.jsonl and claims.jsonl empty, no lane commits on `lane/community`). v2 relaunch operating constraints:

- Effective working window: ~57 min until hard session deadline (12:43 UTC).
- Per-fetch hard limit: 45s (abandon, log to `## Failed fetches`).
- First batch: 5 sources, target first commit within 15 min.
- Skip Reddit on first batch (frequent 403/stall); prefer HN Algolia + practitioner blogs.

## Failed fetches

## Out-of-scope leads

## Open questions for synth

## Manual sourcing tick by chat-agent at +73min
- Eugene Yan eval-process, Hamel Husain field guide + evals-faq
- Tianpan retriever-eval-antipattern (Recall@K/Precision@K thresholds)
- Cite-or-refuse contract pattern
