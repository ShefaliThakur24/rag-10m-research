# PAT + permission failure decoder

When a `curl` call to the GitHub API fails with a 401/403/404/422, map the exact error string to a cause here **before** asking the user to regenerate a token. Most failures are missing one specific fine-grained permission, not "the token is bad".

## Fine-grained PAT — minimum permissions for the PR workflow

| Permission       | Level            | What breaks without it                                                |
|------------------|------------------|-----------------------------------------------------------------------|
| Contents         | Read and write   | `git push` and `git clone` (over HTTPS, using the PAT)                |
| Pull requests    | Read and write   | `POST .../pulls`, comments, reviews, `PUT .../merge`                  |
| Metadata         | Read             | Auto-included. Required to even see the repo (404 otherwise).         |

Optional, only if you trigger them:

| Permission   | Level           | When you need it                                       |
|--------------|-----------------|--------------------------------------------------------|
| Issues       | Read and write  | Closing/linking issues from PR body keywords            |
| Workflows    | Read and write  | Pushing changes to `.github/workflows/*.yml`            |
| Actions      | Read            | Reading CI run status / logs from the API               |
| Checks       | Read            | Reading required-checks status before merge            |

Fine-grained PATs are **per-repo**: a PAT scoped to `acme/foo` returns 404 (not 403) on `acme/bar`. Re-issue with the right repo set, don't change permissions.

Classic PATs: the equivalent scope is `repo` (full). Classic tokens with only `public_repo` will 404 on private repos.

## Decoder table

| Status | Body (substring)                                              | Diagnosis                                                                                       | Fix                                                                                            |
|--------|---------------------------------------------------------------|-------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| 401    | `Bad credentials`                                             | `$GH_TOKEN` empty, expired, or revoked.                                                          | Re-run `git credential fill`; if still empty, regenerate PAT. Confirm with `GET /user`.        |
| 403    | `Resource not accessible by personal access token`            | Fine-grained PAT is missing a permission for this endpoint. Most often **Pull requests: write**.| Edit the PAT, add the missing permission, save. Token string stays the same.                   |
| 403    | `must have admin rights to Repository`                        | Branch is protected; PAT lacks admin or you haven't satisfied required reviews.                  | Either get required reviews, or for emergencies have an admin merge it manually.               |
| 403    | `You have exceeded a secondary rate limit`                    | Too many comments/reviews in a short window.                                                     | Back off (60s+) and retry. Don't loop creating comments.                                       |
| 404    | `Not Found` on a real repo                                    | Fine-grained PAT not scoped to this repo, OR repo is private + PAT has no access.                | Re-issue PAT with the repo selected, or add this repo to the existing PAT's repo set.          |
| 422    | `Can not approve your own pull request`                       | PAT user authored the PR; GitHub forbids self-approval.                                          | Use `event: COMMENT` (or `REQUEST_CHANGES`) with the verdict inline in the body.               |
| 422    | `No commits between {base} and {head}`                        | Branch wasn't pushed, or `head` doesn't match the remote branch name.                            | `git push` first, then verify `git ls-remote origin $HEAD` resolves.                           |
| 422    | `A pull request already exists for ...`                       | A PR for `head -> base` is already open.                                                         | `GET /repos/$O/$R/pulls?head=$O:$HEAD&base=$BASE` to find it and reuse its number.             |
| 422    | `pull_request_review_thread.line must be part of the diff`    | Inline comment targeted a line not present in the diff at that `commit_id`.                      | Pick a line that actually changed, or pass an older `commit_id` that did contain it.           |
| 405    | `Pull Request is not mergeable`                               | Conflicts, failing required checks, or missing required reviews.                                  | `GET .../pulls/$PR` → inspect `mergeable_state` (`dirty`, `blocked`, `behind`). Resolve, retry.|
| 409    | `Head branch was modified. Review and try the merge again`    | Race: someone pushed to head after you started the merge.                                         | Re-fetch HEAD SHA and retry, or pass `sha` in merge body to lock to the SHA you reviewed.      |

## Concrete failures actually observed (use as reference patterns)

### 1. `git push` worked, `POST /pulls` 403'd

```
HTTP/2 403
{"message":"Resource not accessible by personal access token", "documentation_url":"..."}
```

**Diagnosis.** The PAT had **Contents: Read and write** (so `git push` succeeded) but was missing **Pull requests: Read and write**. PR creation, comments, reviews, and merge all go through the Pull requests permission, which is separate from Contents.

**Fix.** Edit the existing PAT at https://github.com/settings/personal-access-tokens → toggle Pull requests to Read and write → save. The token string is unchanged; just re-run the failed `curl` and it works. No need to update any keychain entry.

### 2. Review subagent's `APPROVE` failed with 422

```
HTTP/2 422
{"message":"Validation Failed","errors":[{"resource":"PullRequestReview","code":"custom","message":"Can not approve your own pull request"}]}
```

**Diagnosis.** The PAT belonged to the same user who opened the PR. GitHub treats this as self-approval, which is always rejected — independent of permissions or branch protection settings.

**Workaround.** Submit the review with `event: COMMENT` and put the verdict in the body:

```bash
curl -sS -X POST "${AUTH[@]}" \
  "$API/repos/$OWNER/$REPO/pulls/$PR/reviews" \
  -d '{"event":"COMMENT","body":"## Verdict: ship it\n\nApproval not possible (self-PR)."}'
```

This still appears in the Files-changed tab as a review (state `COMMENTED`) and CodeOwners / required-reviews logic will count it as a non-approving review. If protected branches require an APPROVE, a different reviewer (different PAT) must submit it.

### 3. Initial `git push` failed even though API calls succeeded

```
remote: Permission to OWNER/REPO.git denied to <user>.
fatal: unable to access 'https://github.com/OWNER/REPO.git/': The requested URL returned error: 403
```

**Diagnosis.** PAT scopes don't always map 1:1 to git-over-HTTPS, particularly when the credential helper is caching an older PAT under a different username. The API calls used the new PAT (passed via header) but `git push` used the cached one.

**Two fixes, both reliable.**

a. Inline the token in the push URL — bypasses the credential helper entirely:

```bash
git push https://x-access-token:$GH_TOKEN@github.com/$OWNER/$REPO.git HEAD:$HEAD
```

`x-access-token` is the magic username GitHub accepts when the password is a PAT.

b. Refresh the keychain entry via `git credential fill` / `approve`:

```bash
printf 'protocol=https\nhost=github.com\nusername=x-access-token\npassword=%s\n' "$GH_TOKEN" \
  | git credential approve
```

After this, plain `git push` and `git clone` work without inlining the token.

## Sanity-check commands

Always run these first when something fails — they isolate auth from endpoint-specific bugs.

```bash
curl -sS -o /dev/null -w "%{http_code}\n" "${AUTH[@]}" "$API/user"
curl -sS "${AUTH[@]}" "$API/user" | jq -r .login
curl -sS "${AUTH[@]}" "$API/repos/$OWNER/$REPO" | jq '{name, permissions, private}'
```

- 200 + your login on `/user` → token is valid, not expired.
- 200 + `permissions.push: true` on the repo → Contents: write is wired up.
- 404 on the repo → fine-grained PAT not scoped to this repo (re-issue, don't re-permission).

## Security reminders

- **Never paste `$GH_TOKEN` into a committed file**, a PR body, a code comment, or a chat message. Refer to it by env var name only.
- **Never echo `$GH_TOKEN`** in a shell trace; prefer `set +x` around any curl that interpolates it, or pass it via `-H @-` from a heredoc.
- **Prefer `git credential fill`** to read an existing keychain entry over asking the user for a new token. Treat regenerating a PAT as a last resort.
- If a token is exposed (committed, logged, pasted), revoke it immediately at https://github.com/settings/tokens — don't try to scrub history.
