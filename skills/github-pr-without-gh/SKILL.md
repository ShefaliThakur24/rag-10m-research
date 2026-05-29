---
name: github-pr-without-gh
description: Performs the complete GitHub pull request workflow using only curl against the GitHub REST API when the gh CLI is not installed or unavailable. Covers opening a PR, posting conversation comments, posting inline review comments on diff lines, submitting a formal review (COMMENT or REQUEST_CHANGES; APPROVE fails when the PAT belongs to the PR author), merging via squash/merge/rebase, and branch cleanup. Includes a fine-grained PAT permission decoder for 403 and 422 errors and a git credential fill recipe to reuse an existing keychain-stored token without prompting the user for a fresh one. Use when "gh CLI not installed", "GitHub PR via curl", "GitHub REST API", "post inline PR comment", "submit PR review", "merge PR", or "fine-grained PAT permissions" come up.
---

## When to use this skill

Use when:

- `gh` CLI is not installed or not on PATH.
- The user wants to drive a PR (open, comment, review, merge) via `curl` against the GitHub REST API.
- A PR action just failed with `403 Resource not accessible by personal access token` or `422 Can not approve your own pull request` and the user needs the fine-grained PAT permission decoder.
- The user wants to reuse an existing keychain-stored PAT instead of being prompted for a new one.

Do NOT use when:

- `gh` IS available — just use `gh pr create`, `gh pr review`, `gh pr merge`. They're shorter and handle auth automatically.
- The user wants to do graph-style operations (cross-repo search, project boards). Those are GraphQL and out of scope here.

## Quick reference

| Operation                          | Method + endpoint                                                   | Recipe                                                              |
|------------------------------------|---------------------------------------------------------------------|---------------------------------------------------------------------|
| Open PR                            | `POST /repos/{owner}/{repo}/pulls`                                  | [curl-recipes.md#open-pr](references/curl-recipes.md#open-pr)       |
| Conversation comment (PR overall)  | `POST /repos/{owner}/{repo}/issues/{pr}/comments`                   | [curl-recipes.md#conversation-comment](references/curl-recipes.md#conversation-comment) |
| Inline diff comment                | `POST /repos/{owner}/{repo}/pulls/{pr}/comments`                    | [curl-recipes.md#inline-comment](references/curl-recipes.md#inline-comment) |
| Formal review                      | `POST /repos/{owner}/{repo}/pulls/{pr}/reviews`                     | [curl-recipes.md#review](references/curl-recipes.md#review)         |
| Merge PR                           | `PUT /repos/{owner}/{repo}/pulls/{pr}/merge`                        | [curl-recipes.md#merge](references/curl-recipes.md#merge)           |
| Delete remote branch               | `DELETE /repos/{owner}/{repo}/git/refs/heads/{branch}`              | [curl-recipes.md#delete-branch](references/curl-recipes.md#delete-branch) |
| Get PR (for head commit SHA)       | `GET /repos/{owner}/{repo}/pulls/{pr}`                              | [curl-recipes.md#get-pr](references/curl-recipes.md#get-pr)         |

## Auth (one-liner)

Pull the existing PAT out of the macOS / libsecret keychain via `git credential fill` — do NOT ask the user for a new token, and do NOT paste the token into any committed file.

```bash
GH_TOKEN=$(printf 'protocol=https\nhost=github.com\n\n' \
  | git credential fill \
  | awk -F= '/^password=/{print $2}')
```

Then use it as a bearer token on every API call:

```bash
curl -sS -H "Authorization: Bearer $GH_TOKEN" \
        -H "Accept: application/vnd.github+json" \
        -H "X-GitHub-Api-Version: 2022-11-28" \
        https://api.github.com/user | jq -r .login
```

If `git credential fill` returns nothing, the keychain has no entry. Either run `git push` once interactively to populate it, or fall back to `read -s GH_TOKEN`. Never echo `$GH_TOKEN`.

## Workflow

Assume `OWNER=...`, `REPO=...`, `HEAD=feat/my-branch`, `BASE=main`, `GH_TOKEN=...` are set.

### 1. Push the branch

`git push` may fail even when the API works — PAT scopes don't always map 1:1 to git-over-HTTPS. The reliable form:

```bash
git push https://x-access-token:$GH_TOKEN@github.com/$OWNER/$REPO.git HEAD:$HEAD
```

Or, after `git credential fill` succeeds once, plain `git push -u origin $HEAD` will work because the helper stores the credential.

### 2. Open the PR — capture the number

```bash
PR_JSON=$(curl -sS -X POST \
  -H "Authorization: Bearer $GH_TOKEN" \
  -H "Accept: application/vnd.github+json" \
  https://api.github.com/repos/$OWNER/$REPO/pulls \
  -d "$(jq -n --arg t "Title" --arg b "Body" --arg h "$HEAD" --arg base "$BASE" \
       '{title:$t, body:$b, head:$h, base:$base}')")
PR=$(echo "$PR_JSON" | jq -r .number)
HEAD_SHA=$(echo "$PR_JSON" | jq -r .head.sha)
echo "PR #$PR at $(echo "$PR_JSON" | jq -r .html_url)"
```

Full payload + sample response: [curl-recipes.md#open-pr](references/curl-recipes.md#open-pr).

### 3. Comment

Conversation comment (on the PR as a whole — shows in the Conversation tab):

```bash
curl -sS -X POST -H "Authorization: Bearer $GH_TOKEN" \
  https://api.github.com/repos/$OWNER/$REPO/issues/$PR/comments \
  -d '{"body":"LGTM, shipping."}'
```

Inline comment (on a specific line of the diff — needs `commit_id`, `path`, `line`, `side`):

```bash
curl -sS -X POST -H "Authorization: Bearer $GH_TOKEN" \
  https://api.github.com/repos/$OWNER/$REPO/pulls/$PR/comments \
  -d "$(jq -n --arg sha "$HEAD_SHA" \
       '{commit_id:$sha, path:"src/foo.py", line:42, side:"RIGHT", body:"nit: rename this"}')"
```

`side` is `RIGHT` for the new file, `LEFT` for the old file. Line is the line number in the file at `commit_id`. Full recipe: [curl-recipes.md#inline-comment](references/curl-recipes.md#inline-comment).

### 4. Submit a formal review

```bash
curl -sS -X POST -H "Authorization: Bearer $GH_TOKEN" \
  https://api.github.com/repos/$OWNER/$REPO/pulls/$PR/reviews \
  -d '{"event":"COMMENT","body":"Verdict: ship it. (self-PR, cannot APPROVE)"}'
```

`event` is one of `COMMENT`, `REQUEST_CHANGES`, `APPROVE`. **Do not send `APPROVE` if the PAT belongs to the PR author** — GitHub returns `422 Can not approve your own pull request`. Use `COMMENT` and put the verdict in the body. See [pat-troubleshooting.md#self-approve](references/pat-troubleshooting.md#self-approve).

### 5. Merge

Squash is the default-safe choice (linear history, single commit on `base`):

```bash
curl -sS -X PUT -H "Authorization: Bearer $GH_TOKEN" \
  https://api.github.com/repos/$OWNER/$REPO/pulls/$PR/merge \
  -d '{"merge_method":"squash","commit_title":"feat: thing (#'"$PR"')"}'
```

`merge_method` is `squash | merge | rebase`. A 405 `Pull Request is not mergeable` means CI/conflicts/required reviews still block it.

### 6. Cleanup

Delete the remote branch and the local branch:

```bash
curl -sS -X DELETE -H "Authorization: Bearer $GH_TOKEN" \
  https://api.github.com/repos/$OWNER/$REPO/git/refs/heads/$HEAD
git checkout $BASE && git pull && git branch -d $HEAD
```

## Common 403 / 422 errors

| Status | Message (substring)                                         | Most likely cause                                              |
|--------|-------------------------------------------------------------|----------------------------------------------------------------|
| 401    | `Bad credentials`                                           | `GH_TOKEN` empty/expired. Re-run `git credential fill`.        |
| 403    | `Resource not accessible by personal access token`          | Fine-grained PAT missing a permission (see decoder below).     |
| 403    | `must have admin rights`                                    | Branch is protected and PAT lacks admin or required reviews.   |
| 422    | `Can not approve your own pull request`                     | PAT belongs to PR author; use `event: COMMENT` instead.        |
| 422    | `No commits between {base} and {head}`                      | Branch hasn't been pushed, or `head` arg is wrong.             |
| 405    | `Pull Request is not mergeable`                             | Conflicts, failing required checks, or missing required reviews. |
| 404    | `Not Found` on a real repo                                  | PAT doesn't have access to that repo (fine-grained PAT scoped to wrong repos). |

Fine-grained PAT permissions actually needed:

- **Contents**: Read and write (for `git push`)
- **Pull requests**: Read and write (for PR create, comment, review, merge)
- **Metadata**: Read (auto-included; required to even see the repo)

Full decoder: [pat-troubleshooting.md](references/pat-troubleshooting.md).

## Additional resources

- **Read [`references/curl-recipes.md`](references/curl-recipes.md)** when you need the full JSON payload, exact headers, and a sample response for an operation. Each section has a `## ` anchor matching the Quick reference table above.
- **Read [`references/pat-troubleshooting.md`](references/pat-troubleshooting.md)** when an API call returned 401/403/404/422 and you need to map the error string to the missing permission or the workaround. Start there before asking the user to regenerate a token.
