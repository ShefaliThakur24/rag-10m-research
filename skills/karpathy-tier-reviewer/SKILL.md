---
name: karpathy-tier-reviewer
description: Reviews research and technical deliverables (PRs, design docs, research notes, ML synthesis documents) against an 8-clause Karpathy-style rubric covering quote accuracy, citation freshness, prose density, tradeoff specificity, audience fit, internal consistency, image necessity, and schema compliance. Produces PASS/FAIL/N/A per clause, a single APPROVE/REQUEST_CHANGES/COMMENT-WITH-NOTES verdict, and inline or conversation comments using canonical templates. Use when the user asks to review a research document, review an ML design doc, perform a rigorous technical review, do a Karpathy-style review, run a rubric-based PR review, or evaluate a research deliverable. Not for code review of programming PRs.
---

# Karpathy-tier reviewer

Eight clauses. Every sentence in the artifact earns its tokens or it gets cut. The rubric is opinionated by design; "it depends" answers fail clause 4.

## When to use this skill

Triggers:

- "review research document"
- "review ML design doc"
- "rigorous technical review"
- "Karpathy-style review"
- "rubric-based PR review"
- "research deliverable review"
- A diff or PR that changes prose-heavy markdown: design docs, research notes, synthesis documents, claim files, appendix material.

NOT for code review of programming PRs. If the diff is source code with no prose-deliverable component, decline and route the request to a code-review skill.

## The 8 clauses (one-line each)

1. Quote accuracy — every direct quote and number traces to a cited source; unverifiable claims get hedged or cut.
2. Citation freshness — sources are dated; stale citations get refreshed or marked deprecated; no "according to a recent paper" without a year.
3. Prose style — terse, dense, no filler, no hype, no marketing voice; every sentence earns its tokens.
4. Tradeoff specificity — "X beats Y" comes with the regime: data scale, latency budget, model class, recall floor.
5. Audience fit — calibrated to the stated audience; for an advanced production audience, foundational definitions belong in an appendix.
6. Internal consistency — terminology, units, model versions, and numbers match across the document.
7. Image necessity — diagram exists iff prose would exceed ~150 words or the relationship is fundamentally spatial. No decorative images.
8. Schema compliance — machine-readable artifacts parse and follow their declared schema. One bad line fails the file.

Full clause text, failure modes seen in practice, and worked examples: `references/rubric-clauses.md`.

## Review procedure

1. Identify the unit under review: a single file, a commit, or a PR. Read it end to end before commenting. Do not start writing notes on page one.
2. Walk the 8 clauses in order. For each clause:
   - Assign PASS, FAIL, or N/A.
   - Write one line of justification. Quote the offending text or name what is missing. No vague "needs work".
3. Aggregate to a verdict:
   - APPROVE — all clauses PASS or N/A.
   - REQUEST_CHANGES — any clause FAIL that blocks merge. Default-block clauses: 1, 3, 4, 8. Clauses 2, 5, 6, 7 block only when the failure is severe (e.g., contradiction across the doc, not a single citation format drift).
   - COMMENT-WITH-NOTES — minor or stylistic clause failures the author should see but that do not block merge.
4. Decide comment placement (see "Comment placement decision" below).
5. Post the verdict block verbatim at the top of a single conversation comment, then the per-clause table, then specific notes. The block must be copy-pasteable.
6. Post inline comments for clause failures pinned to a specific line. One inline comment per failure, not per clause.

Read `references/comment-templates.md` before posting the first comment.

## Verdict format

Reproduce this block verbatim at the top of the conversation comment. Fill bracketed fields. Do not add emoji, hedges, or closing pleasantries.

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
```

## Comment placement decision

- Inline (review comment pinned to a diff line): a specific line, paragraph, table row, or image breaks a clause. The fix is local. One inline per failure.
- Conversation (single top-level PR or file comment): the failure is cross-cutting — audience drift across the whole doc, contradiction between two distant sections, schema breakage across an entire jsonl file, prose style failing throughout. The verdict block always goes here.

Default: one conversation comment carrying the verdict block, plus zero to N inline comments for line-level failures. Never split the verdict across multiple comments.

## Additional resources

- Read `references/rubric-clauses.md` when you need the full clause definition, failure modes seen in practice, or worked pass/fail examples for a clause.
- Read `references/comment-templates.md` before posting your first comment; copy the literal inline-comment and conversation-comment shapes from there and fill them.
