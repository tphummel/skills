---
name: gitea
description: Interact with a self-hosted Gitea instance via its REST API. Use this skill whenever the user wants to do anything with Gitea — creating or updating pull requests, checking PR status, listing branches, reading commit history, merging PRs, adding PR comments, or reading/writing wiki pages. Trigger on requests like "update the PR description", "what PRs are open?", "merge the PR", "what branches exist?", "add a comment to the PR", "what's the latest commit on main?", "add a wiki page for X", "what does the wiki say about Y", "update the wiki", or any time the user mentions a PR number, a repo wiki, or asks about repo state on Gitea.
---

# Gitea API Skill

## Authentication

The base URL and API token are already available as environment variables — no need to fetch them from anywhere:

```
GITEA_BASE_URL   e.g. https://gitea.hummel.casa
GITEA_API_KEY    the API token
```

All requests use:
```
Authorization: token $GITEA_API_KEY
Accept: application/json
Content-Type: application/json   (for POST/PATCH/PUT)
```

Base URL for the API: `$GITEA_BASE_URL/api/v1`

## Known repos

| Short name | Full path |
|------------|-----------|
| terraform  | `hummel.casa/terraform` |
| ansible    | `hummel.casa/ansible` |
| dotfiles   | `hummel.casa/dotfiles` |

For other repos, ask the user or infer from context.

---

## Pull Requests

### List open PRs
```bash
curl -s -H "Authorization: token $GITEA_API_KEY" -H "Accept: application/json" \
  "$GITEA_BASE_URL/api/v1/repos/{owner}/{repo}/pulls?state=open&limit=50" | jq .
```

### Get a specific PR
```bash
curl -s -H "Authorization: token $GITEA_API_KEY" -H "Accept: application/json" \
  "$GITEA_BASE_URL/api/v1/repos/{owner}/{repo}/pulls/{index}" | jq .
```
PRs are addressed by their number (`index`), not an internal ID.

### Create a PR
```bash
curl -s -X POST \
  -H "Authorization: token $GITEA_API_KEY" \
  -H "Accept: application/json" \
  -H "Content-Type: application/json" \
  -d '{
    "title": "PR title",
    "body": "Description here",
    "head": "feature-branch",
    "base": "main"
  }' \
  "$GITEA_BASE_URL/api/v1/repos/{owner}/{repo}/pulls"
```

### Update PR title or description
```bash
curl -s -X PATCH \
  -H "Authorization: token $GITEA_API_KEY" \
  -H "Accept: application/json" \
  -H "Content-Type: application/json" \
  -d '{"title": "New title", "body": "New description"}' \
  "$GITEA_BASE_URL/api/v1/repos/{owner}/{repo}/pulls/{index}"
```
Both fields are optional — only include the ones you want to change.

### Merge a PR
```bash
curl -s -X POST \
  -H "Authorization: token $GITEA_API_KEY" \
  -H "Accept: application/json" \
  -H "Content-Type: application/json" \
  -d '{"Do": "merge", "merge_message_field": "Merge PR #N"}' \
  "$GITEA_BASE_URL/api/v1/repos/{owner}/{repo}/pulls/{index}/merge"
```
`Do` can be `"merge"`, `"rebase"`, or `"squash"`. Returns 204 on success.

### Add a comment to a PR
```bash
curl -s -X POST \
  -H "Authorization: token $GITEA_API_KEY" \
  -H "Accept: application/json" \
  -H "Content-Type: application/json" \
  -d '{"body": "Comment text here"}' \
  "$GITEA_BASE_URL/api/v1/repos/{owner}/{repo}/issues/{index}/comments"
```
PR comments use the `/issues/{index}/comments` endpoint (PRs and issues share the same comment system in Gitea).

### List PR comments
```bash
curl -s -H "Authorization: token $GITEA_API_KEY" -H "Accept: application/json" \
  "$GITEA_BASE_URL/api/v1/repos/{owner}/{repo}/issues/{index}/comments" | jq .
```

---

## Branches

### List branches
```bash
curl -s -H "Authorization: token $GITEA_API_KEY" -H "Accept: application/json" \
  "$GITEA_BASE_URL/api/v1/repos/{owner}/{repo}/branches?limit=50" | \
  jq '[.[] | {name: .name, sha: .commit.id[:8], date: .commit.timestamp, author: .commit.author.name}] | sort_by(.date) | reverse'
```
Include the commit author and sort most-recent-first — "what branches exist" almost always means "which ones are stale," and both details matter for that judgment.

### Get a specific branch
```bash
curl -s -H "Authorization: token $GITEA_API_KEY" -H "Accept: application/json" \
  "$GITEA_BASE_URL/api/v1/repos/{owner}/{repo}/branches/{branch}" | jq .
```

### Delete a branch
```bash
curl -s -X DELETE \
  -H "Authorization: token $GITEA_API_KEY" -H "Accept: application/json" \
  "$GITEA_BASE_URL/api/v1/repos/{owner}/{repo}/branches/{branch}"
```
Returns 204 on success. Confirm with the user before deleting.

---

## Commits

### List commits on a branch
```bash
curl -s -H "Authorization: token $GITEA_API_KEY" -H "Accept: application/json" \
  "$GITEA_BASE_URL/api/v1/repos/{owner}/{repo}/commits?sha={branch}&limit=20" | \
  jq '[.[] | {sha: .sha[:8], message: .commit.message | split("\n")[0], author: .commit.author.name, date: .commit.author.date}]'
```

### Get a single commit
```bash
curl -s -H "Authorization: token $GITEA_API_KEY" -H "Accept: application/json" \
  "$GITEA_BASE_URL/api/v1/repos/{owner}/{repo}/git/commits/{sha}" | jq .
```

---

## Wiki

Wiki pages are addressed by a **slug** (`sub_url`), not the human-readable **title** — and the two are often different. Always get the slug from a `title`/`sub_url` pair in a list or create response before doing a GET/PATCH/DELETE; never guess a slug from the title yourself (e.g. spaces become `-`, but Gitea can add unexpected suffixes like a trailing `.-` in some cases — don't try to reproduce the encoding by hand).

Page content is base64-encoded on the wire (`content_base64`) in both directions.

### List wiki pages
```bash
curl -s -H "Authorization: token $GITEA_API_KEY" -H "Accept: application/json" \
  "$GITEA_BASE_URL/api/v1/repos/{owner}/{repo}/wiki/pages" | jq '[.[] | {title, sub_url}]'
```

### Get a wiki page
```bash
curl -s -H "Authorization: token $GITEA_API_KEY" -H "Accept: application/json" \
  "$GITEA_BASE_URL/api/v1/repos/{owner}/{repo}/wiki/page/{slug}" | jq '. + {content: (.content_base64 | @base64d)}'
```
Use the `sub_url` from the list/create response as `{slug}`, URL-encoded if it contains special characters.

### Create a wiki page
```bash
CONTENT_B64=$(printf '%s' "$MARKDOWN_CONTENT" | base64 -w0)
curl -s -X POST \
  -H "Authorization: token $GITEA_API_KEY" \
  -H "Accept: application/json" \
  -H "Content-Type: application/json" \
  -d "{\"title\": \"Page Title\", \"content_base64\": \"$CONTENT_B64\", \"message\": \"commit message\"}" \
  "$GITEA_BASE_URL/api/v1/repos/{owner}/{repo}/wiki/new"
```
Returns 201 with the page's real `sub_url` — use that for any follow-up edit/delete, don't assume it matches `title`.

### Edit a wiki page
```bash
CONTENT_B64=$(printf '%s' "$MARKDOWN_CONTENT" | base64 -w0)
curl -s -X PATCH \
  -H "Authorization: token $GITEA_API_KEY" \
  -H "Accept: application/json" \
  -H "Content-Type: application/json" \
  -d "{\"title\": \"Page Title\", \"content_base64\": \"$CONTENT_B64\", \"message\": \"commit message\"}" \
  "$GITEA_BASE_URL/api/v1/repos/{owner}/{repo}/wiki/page/{slug}"
```
**Always include `title` on PATCH, even if unchanged.** The API docs claim omitting it "keeps it unchanged," but in practice omitting it renames the page to a generic placeholder (like "unnamed") and changes its slug — silently orphaning the old URL. Fetch the current title first (via GET) if you don't already know it, and pass it back explicitly.

### Get wiki page revision history
```bash
curl -s -H "Authorization: token $GITEA_API_KEY" -H "Accept: application/json" \
  "$GITEA_BASE_URL/api/v1/repos/{owner}/{repo}/wiki/revisions/{slug}" | jq '.commits[] | {sha: .sha[:8], message: .commit.message, author: .author.name, date: .author.date}'
```

### Delete a wiki page
```bash
curl -s -X DELETE \
  -H "Authorization: token $GITEA_API_KEY" -H "Accept: application/json" \
  "$GITEA_BASE_URL/api/v1/repos/{owner}/{repo}/wiki/page/{slug}"
```
Returns 204 on success. Confirm with the user before deleting.

---

## Tips

- **PR numbers vs internal IDs**: Always use the human-visible PR number (e.g., `50`) in URLs, not the internal `id` field from API responses.
- **Pagination**: List endpoints default to 20 results. Add `?limit=50` for more, or follow the `Link` header for full pagination.
- **PR state**: `state=open`, `state=closed`, or `state=all`.
- **Terraform PRs go through Atlantis**: Merging a terraform PR triggers `terraform plan/apply` automatically — confirm with the user before merging.
- **Uninitialized wikis return 404, not an empty list**: `GET /wiki/pages` 404s until the wiki has had at least one page ever created (even if all pages are later deleted, it then correctly returns `[]`). A 404 here usually just means "no wiki content yet," not a broken request — check `has_wiki` on the repo object if unsure.
