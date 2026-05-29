# curl recipes for the GitHub PR REST API

Every recipe assumes these shell vars are set:

```bash
OWNER=...                # e.g. octocat
REPO=...                 # e.g. hello-world
PR=...                   # PR number, set after open-pr
HEAD=feat/my-branch      # head branch name
BASE=main                # base branch name
GH_TOKEN=...             # from `git credential fill` — see SKILL.md Auth
API=https://api.github.com
AUTH=(-H "Authorization: Bearer $GH_TOKEN" -H "Accept: application/vnd.github+json" -H "X-GitHub-Api-Version: 2022-11-28")
```

Use `curl -sS "${AUTH[@]}" ...` to splat the three required headers. Never log `$GH_TOKEN`.

---

## open-pr

`POST /repos/{owner}/{repo}/pulls`

```bash
PR_JSON=$(curl -sS -X POST "${AUTH[@]}" \
  "$API/repos/$OWNER/$REPO/pulls" \
  -d "$(jq -n \
        --arg title "feat: add widget" \
        --arg body  "## Summary\n- adds widget\n- tests included" \
        --arg head  "$HEAD" \
        --arg base  "$BASE" \
        '{title:$title, body:$body, head:$head, base:$base, draft:false}')")

PR=$(echo "$PR_JSON" | jq -r .number)
HEAD_SHA=$(echo "$PR_JSON" | jq -r .head.sha)
NODE_ID=$(echo "$PR_JSON" | jq -r .node_id)
URL=$(echo "$PR_JSON" | jq -r .html_url)
echo "Opened PR #$PR -> $URL (head sha $HEAD_SHA)"
```

Sample response (truncated):

```json
{
  "number": 1347,
  "node_id": "PR_kwDOA...",
  "html_url": "https://github.com/octocat/hello-world/pull/1347",
  "state": "open",
  "head": { "ref": "feat/my-branch", "sha": "6dcb09b5b57875f334f61aebed695e2e4193db5e" },
  "base": { "ref": "main",           "sha": "abc123..." },
  "mergeable": null,
  "mergeable_state": "unknown"
}
```

`mergeable` is `null` immediately after open — GitHub computes it async. Re-`GET` the PR a moment later to check.

Cross-fork PRs: set `head` to `OTHER_OWNER:branch-name`.

---

## get-pr

`GET /repos/{owner}/{repo}/pulls/{pr}` — use this to read `head.sha` (needed for inline comments) and `mergeable` / `mergeable_state`.

```bash
curl -sS "${AUTH[@]}" "$API/repos/$OWNER/$REPO/pulls/$PR" \
  | jq '{number, state, mergeable, mergeable_state, head_sha: .head.sha, base: .base.ref}'
```

---

## conversation-comment

`POST /repos/{owner}/{repo}/issues/{pr}/comments` — note this is the **issues** endpoint; on GitHub, PRs are issues for the purposes of conversation-level comments. Body is plain Markdown.

```bash
curl -sS -X POST "${AUTH[@]}" \
  "$API/repos/$OWNER/$REPO/issues/$PR/comments" \
  -d '{"body":"Thanks for the review — pushed a fix in `abc123`."}'
```

Sample response (truncated):

```json
{
  "id": 1,
  "html_url": "https://github.com/octocat/hello-world/pull/1347#issuecomment-1",
  "body": "Thanks for the review — pushed a fix in `abc123`.",
  "user": { "login": "you" }
}
```

---

## inline-comment

`POST /repos/{owner}/{repo}/pulls/{pr}/comments` — comment on a specific line of the diff.

Required body fields:

- `commit_id` — the SHA the comment is attached to. Almost always `HEAD_SHA` from `open-pr` / `get-pr`.
- `path` — file path relative to repo root, forward slashes.
- `line` — line number in the file at `commit_id`.
- `side` — `RIGHT` (new file / addition) or `LEFT` (old file / deletion). For a normal "I'm commenting on the added line" case, use `RIGHT`.
- `body` — Markdown.

Single-line example:

```bash
curl -sS -X POST "${AUTH[@]}" \
  "$API/repos/$OWNER/$REPO/pulls/$PR/comments" \
  -d "$(jq -n --arg sha "$HEAD_SHA" '{
        commit_id: $sha,
        path:      "src/foo.py",
        line:      42,
        side:      "RIGHT",
        body:      "nit: this branch is dead — `frobnicate()` already returns early above."
      }')"
```

Multi-line range (e.g. lines 40–42 of `src/foo.py`):

```bash
curl -sS -X POST "${AUTH[@]}" \
  "$API/repos/$OWNER/$REPO/pulls/$PR/comments" \
  -d "$(jq -n --arg sha "$HEAD_SHA" '{
        commit_id:  $sha,
        path:       "src/foo.py",
        start_line: 40,
        start_side: "RIGHT",
        line:       42,
        side:       "RIGHT",
        body:       "consider extracting this block into a helper."
      }')"
```

Common 422s here:

- `pull_request_review_thread.line must be part of the diff` — the line isn't in the diff for that `commit_id`. Either use a line that actually changed, or pass an older `commit_id` that does include it.
- `pull_request_review_thread.path diff too large` — file is collapsed; use `position` (diff hunk offset) instead of `line`.

Reply to an existing inline thread: add `in_reply_to: <comment_id>` and drop `commit_id` / `path` / `line` / `side`.

---

## review

`POST /repos/{owner}/{repo}/pulls/{pr}/reviews` — submits a formal review (the green/red/grey badge in the Files tab).

```bash
curl -sS -X POST "${AUTH[@]}" \
  "$API/repos/$OWNER/$REPO/pulls/$PR/reviews" \
  -d '{
        "event": "COMMENT",
        "body":  "## Verdict: ship it\n\nLGTM. Self-PR so cannot formally APPROVE."
      }'
```

`event` values:

- `COMMENT` — leaves the review without a verdict. Use this when the PAT belongs to the PR author (APPROVE will 422 otherwise).
- `REQUEST_CHANGES` — blocks the merge button on protected branches.
- `APPROVE` — green check. **Will 422 with `Can not approve your own pull request` if the PAT user authored the PR.**

You can also bundle inline comments into the review (so they all appear together) by passing a `comments` array:

```bash
curl -sS -X POST "${AUTH[@]}" \
  "$API/repos/$OWNER/$REPO/pulls/$PR/reviews" \
  -d "$(jq -n '{
        event: "REQUEST_CHANGES",
        body:  "Two blockers inline.",
        comments: [
          {path: "src/foo.py", line: 42, side: "RIGHT", body: "null check missing"},
          {path: "src/bar.py", line: 17, side: "RIGHT", body: "use the helper"}
        ]
      }')"
```

Sample response (truncated):

```json
{
  "id": 80,
  "state": "COMMENTED",
  "html_url": "https://github.com/octocat/hello-world/pull/1347#pullrequestreview-80",
  "submitted_at": "2026-05-29T12:00:00Z"
}
```

---

## merge

`PUT /repos/{owner}/{repo}/pulls/{pr}/merge`

Squash (linear history, single commit on `base`):

```bash
curl -sS -X PUT "${AUTH[@]}" \
  "$API/repos/$OWNER/$REPO/pulls/$PR/merge" \
  -d "$(jq -n --arg n "$PR" '{
        merge_method: "squash",
        commit_title: "feat: add widget (#\($n))",
        commit_message: "Squashed merge of #\($n)."
      }')"
```

Plain merge commit:

```bash
curl -sS -X PUT "${AUTH[@]}" \
  "$API/repos/$OWNER/$REPO/pulls/$PR/merge" \
  -d '{"merge_method":"merge"}'
```

Rebase (replays commits onto `base`, no merge commit):

```bash
curl -sS -X PUT "${AUTH[@]}" \
  "$API/repos/$OWNER/$REPO/pulls/$PR/merge" \
  -d '{"merge_method":"rebase"}'
```

Optional `sha` field — fail the merge if the PR head no longer matches that SHA (race-condition guard):

```bash
curl -sS -X PUT "${AUTH[@]}" \
  "$API/repos/$OWNER/$REPO/pulls/$PR/merge" \
  -d "$(jq -n --arg sha "$HEAD_SHA" '{merge_method:"squash", sha:$sha}')"
```

Sample 200 response:

```json
{ "sha": "6dcb09b5b57875f334f61aebed695e2e4193db5e", "merged": true, "message": "Pull Request successfully merged" }
```

Common failures:

- `405 Pull Request is not mergeable` — conflicts, failing required checks, or missing required reviews. `GET` the PR and inspect `mergeable_state`.
- `409 Head branch was modified. Review and try the merge again` — someone pushed to the head after you started; re-fetch and retry (or use the `sha` guard).

---

## delete-branch

`DELETE /repos/{owner}/{repo}/git/refs/heads/{branch}` — removes the remote ref. Local cleanup is plain git.

```bash
curl -sS -X DELETE "${AUTH[@]}" \
  "$API/repos/$OWNER/$REPO/git/refs/heads/$HEAD"

git checkout $BASE && git pull --ff-only && git branch -d $HEAD
```

A 204 No Content is success. A 422 `Reference does not exist` means it was already deleted (e.g. "delete branch on merge" is enabled in repo settings) — safe to ignore.

---

## Cheatsheet: the whole flow in 10 lines

```bash
GH_TOKEN=$(printf 'protocol=https\nhost=github.com\n\n' | git credential fill | awk -F= '/^password=/{print $2}')
AUTH=(-H "Authorization: Bearer $GH_TOKEN" -H "Accept: application/vnd.github+json" -H "X-GitHub-Api-Version: 2022-11-28")
API=https://api.github.com

git push https://x-access-token:$GH_TOKEN@github.com/$OWNER/$REPO.git HEAD:$HEAD
PR_JSON=$(curl -sS -X POST "${AUTH[@]}" "$API/repos/$OWNER/$REPO/pulls" -d "$(jq -n --arg h "$HEAD" --arg b "$BASE" '{title:"feat: x", body:"...", head:$h, base:$b}')")
PR=$(echo "$PR_JSON" | jq -r .number); HEAD_SHA=$(echo "$PR_JSON" | jq -r .head.sha)
curl -sS -X POST "${AUTH[@]}" "$API/repos/$OWNER/$REPO/issues/$PR/comments" -d '{"body":"shipping"}'
curl -sS -X POST "${AUTH[@]}" "$API/repos/$OWNER/$REPO/pulls/$PR/reviews"   -d '{"event":"COMMENT","body":"verdict: ship"}'
curl -sS -X PUT  "${AUTH[@]}" "$API/repos/$OWNER/$REPO/pulls/$PR/merge"    -d '{"merge_method":"squash"}'
curl -sS -X DELETE "${AUTH[@]}" "$API/repos/$OWNER/$REPO/git/refs/heads/$HEAD"
```
