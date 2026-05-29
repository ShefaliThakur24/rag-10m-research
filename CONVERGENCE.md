# Convergence trigger (Shape A done -> publish + stop)

Tuned for a 2-hour session. The reviewer evaluates these signals on every tick after the first synth tick has landed.

## Hard trigger

Stop and publish when **all** of these hold simultaneously:

1. **Coverage**: each of the 7 pipeline stages has at least one `current-recommendation` entry in `doc/01-approach.md` with confidence >= 0.6 (relaxed from 0.7 for a 2-hour run):
   - ingest
   - chunk
   - embed
   - index
   - retrieve
   - rerank
   - generate + guardrail
2. **Per-lane novelty**: each lane's last 2 batches added < 15% new claims relative to its existing claim count.
3. **Cross-lane novelty**: the most recent synth tick added < 10% new claims to `synth/claims.jsonl`.
4. **Contradiction backlog**: `synth/contradictions.jsonl` has no `status: open` entries older than 2 synth ticks.
5. **Reviewer pass rate** over last 20 commits >= 70% (relaxed from 80% for 2-hour run).

## Soft trigger (wall-clock)

Stop and publish unconditionally when **session wall-clock >= 120 min**. Session start = the timestamp of the initial scaffold commit on `main`. Reviewer checks `git log --reverse --format='%aI' main | head -1`.

If the hard trigger has not fired by +110 min, the reviewer should commit `[shape-a] converged-on-timeout` and dispatch publish anyway. The doc may still have gaps; `doc/03-rejected.md` should list what's known to be incomplete.

## When triggered

The reviewer:

1. Writes a final commit on `main`: `[shape-a] converged` (or `[shape-a] converged-on-timeout`).
2. Writes a one-line entry to `CHANGELOG.md` summarizing the run.
3. Dispatches the publish subagent (per `publish/BRIEF.md`).
4. Signals lanes to stop by writing a commit on `main` containing `STOP-ALL-LANES` in the message body.
5. Stops itself (the dispatcher in chat kills the /loop sleepers).
