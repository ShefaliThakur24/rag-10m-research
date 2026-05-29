# Comment templates

Two templates. Use both literally. Fill the bracketed fields. Do not paraphrase the headings, do not add emoji, do not add a closing pleasantry.

## Conversation comment (one per review)

Post this as a single top-level PR comment (or file-level comment when reviewing a single file). The verdict block at the top must be copy-pasteable.

```
**Verdict:** [APPROVE | REQUEST_CHANGES | COMMENT-WITH-NOTES]

| Clause | Result | Note |
|--------|--------|------|
| 1. Quote accuracy       | [PASS/FAIL/N/A] | <one line> |
| 2. Citation freshness   | [PASS/FAIL/N/A] | <one line> |
| 3. Prose style          | [PASS/FAIL/N/A] | <one line> |
| 4. Tradeoff specificity | [PASS/FAIL/N/A] | <one line> |
| 5. Audience fit         | [PASS/FAIL/N/A] | <one line> |
| 6. Internal consistency | [PASS/FAIL/N/A] | <one line> |
| 7. Image necessity      | [PASS/FAIL/N/A] | <one line> |
| 8. Schema compliance    | [PASS/FAIL/N/A] | <one line> |

**Blocking failures:** [comma-separated clause numbers, or "none"]

## Cross-cutting notes

- **Clause [N] — [clause name]:** <one paragraph, max 4 sentences, naming the failure and the fix the author should make. Quote the offending phrase if it recurs.>
- **Clause [N] — [clause name]:** <same shape>

## Inline comments

[count] inline comments posted; see annotations on the diff.
```

Rules:

- Omit the "Cross-cutting notes" section entirely when there are none.
- Omit the "Inline comments" line when zero inline comments were posted.
- Never list a clause in "Cross-cutting notes" that you also addressed inline; pick one venue per failure.
- If verdict is APPROVE and no cross-cutting notes exist, post just the verdict block.

## Inline comment (one per specific line-level failure)

Post on the offending diff line. One inline comment per failure, not per clause. Quote the exact offending text; suggest the fix in one sentence; do not write the fix yourself.

```
**Clause [N] — [clause name]: FAIL**

> [exact quoted offending line or phrase from the diff]

[One-sentence reason it fails the clause.]

**Fix:** [one-sentence suggestion. Imperative voice. No "could you maybe consider".]
```

Examples (do not copy verbatim; these show the shape):

```
**Clause 4 — Tradeoff specificity: FAIL**

> "Dense embeddings outperform sparse retrieval."

No regime stated: corpus size, query distribution, model, and metric are all missing.

**Fix:** State the regime, e.g. "for paraphrase queries on >1M-doc corpora, bge-base-en-v1.5 beats BM25 by ~8-15 nDCG@10".
```

```
**Clause 1 — Quote accuracy: FAIL**

> "Anthropic reports 49% retrieval failure reduction."

Number not findable in the linked post; no page or section reference.

**Fix:** Add the exact source location, or hedge to "reportedly ~50% reduction".
```

```
**Clause 7 — Image necessity: FAIL**

> doc/images/three-benefits.png

The relationship is a flat 3-item list; a diagram adds no spatial information.

**Fix:** Replace with an inline bulleted list.
```

## Voice

- Third person, technical, no hedging in the reviewer's own voice.
- Quote the offending text exactly with `>`. Do not paraphrase the author.
- One sentence of diagnosis, one sentence of fix. Stop there.
- No emoji, no "great work", no "in conclusion", no "hope this helps".
