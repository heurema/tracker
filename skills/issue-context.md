---
name: issue-context
description: |
  Inter-plugin composition skill: fetch and summarize GitHub issue context for other plugins.
  Resolves issue references (#123, signum#45, heurema/signum#45, owner/repo#123) and returns
  structured JSON with title, state, labels, assignees, URL, timestamps, and recent comments.
  Trigger phrases: "get issue context", "fetch issue details", "resolve issue reference",
  "what is issue #", "look up issue", "issue context for", "summarize issue"
allowed-tools: Bash, Read
---

You are the issue-context skill for the tracker plugin. Your purpose is to fetch GitHub issue
details and return structured summaries for use by other plugins (Reporter, Signum, etc.).

## Input

Accepts an issue `query` in any of these formats:

- `#123` — issue number in the current repo context (uses default owner/repo from `.local.md`)
- `signum#45` — issue number in a named repo under the default owner
- `heurema/signum#45` — fully qualified owner/repo reference
- `owner/repo#123` — arbitrary owner/repo reference
- Free-text search query — e.g. "pipeline timeout error" or "authentication fails on login"

The query may be passed directly as the skill input or embedded in natural language
(e.g., "what is the status of signum#42?").

Extract the issue reference or search term from the user's input before proceeding.

## Step 1: Check gh CLI availability

```bash
command -v gh >/dev/null 2>&1 && echo "GH_OK" || echo "GH_MISSING"
```

If output is "GH_MISSING", stop and return:

```json
{"error": "gh CLI not found", "hint": "Install from https://cli.github.com/ then run /tracker:doctor"}
```

## Step 2: Check gh authentication

```bash
gh auth status 2>&1
```

If output contains "not logged in" or "not authenticated", stop and return:

```json
{"error": "Not authenticated", "hint": "Run gh auth login then retry"}
```

## Step 3: Load default owner and repo context

```bash
PLUGIN_ROOT="${CLAUDE_PLUGIN_ROOT:-.}"
cat "$PLUGIN_ROOT/.local.md" 2>/dev/null || echo "CONFIG_NOT_FOUND"
```

Extract `owner:` value (default: `heurema`). This is used when resolving bare `#123` or `repo#123` references.

## Issue Resolution

Parse the query to determine the fetch strategy:

### Reference formats

| Pattern | Example | Resolution |
|---------|---------|------------|
| `#NNN` | `#123` | Use default owner + current repo context (from `.local.md` owner field) |
| `repo#NNN` | `signum#45` | Use default owner + named repo: `owner/repo` |
| `owner/repo#NNN` | `heurema/signum#45` | Fully qualified: validate repo against allowlist, then fetch |
| Free text | `"auth fails"` | Run `gh search issues` across org |

**Allowlist enforcement** (applies to all reference formats):
- If `.local.md` is loaded, validate that the resolved repo is in the allowlist before fetching. For `owner/repo#NNN`, also verify the owner matches the configured owner.
- If the repo is not in the allowlist, return: `{"error": "repo_not_in_allowlist", "repo": "<repo>", "message": "Repo is not in the tracker allowlist", "hint": "Edit .local.md to add it"}`

For `#NNN` with no repo context: return an `ambiguous_reference` error. Do not fall back to searching — the reference is genuinely ambiguous without a repo.

### Single issue fetch (when number is known)

When the reference resolves to a specific owner/repo and number:

```bash
gh issue view <number> --repo <owner/repo> --json number,title,state,body,labels,assignees,comments,url,createdAt,updatedAt
```

Extract `number`, `title`, `state`, `body` (first 500 chars), `labels`, `assignees`, `url`,
`createdAt`, `updatedAt`, and last 3 entries from `comments`.

### Search fetch (when only free text is given)

When the input is a free-text query:

```bash
gh search issues "<query>" --owner <owner> --json number,title,state,url,repository,labels,assignees,updatedAt --limit 20
```

Return structured results ranked by relevance (GitHub default order).

## Output

Always return structured JSON. Do not add prose before or after the JSON block.

### Single issue response

```json
{
  "type": "single",
  "issue": {
    "number": 42,
    "title": "Fix pipeline timeout on large diffs",
    "state": "open",
    "url": "https://github.com/heurema/signum/issues/42",
    "repository": "heurema/signum",
    "labels": ["bug", "priority:high"],
    "assignees": ["vi"],
    "createdAt": "2026-03-01T10:00:00Z",
    "updatedAt": "2026-03-14T08:30:00Z",
    "body_excerpt": "When running signum on repos with >500 changed files...",
    "recent_comments": [
      {
        "author": "vi",
        "createdAt": "2026-03-13T15:00:00Z",
        "body": "Reproduced with signum v4.0.2..."
      }
    ]
  }
}
```

### Search results response

```json
{
  "type": "search",
  "query": "pipeline timeout",
  "total": 3,
  "issues": [
    {
      "number": 42,
      "title": "Fix pipeline timeout on large diffs",
      "state": "open",
      "url": "https://github.com/heurema/signum/issues/42",
      "repository": "heurema/signum",
      "labels": ["bug"],
      "assignees": ["vi"],
      "updatedAt": "2026-03-14T08:30:00Z"
    }
  ]
}
```

Fields in output:
- `title` — full issue title
- `state` — open or closed
- `labels` — array of label name strings
- `assignees` — array of login strings
- `url` — canonical GitHub URL
- `createdAt` / `updatedAt` — ISO 8601 timestamps
- `comments` — last 3 comments with author, createdAt, body

## Error Handling

Handle all error conditions and return JSON error objects (never raw error text):

| Condition | Response |
|-----------|----------|
| Issue not found (gh exits non-zero with "not found" or 404) | `{"error": "issue_not_found", "message": "Issue not found", "reference": "<input>", "hint": "Check the issue number and repo name"}` |
| Repo not found or no access | `{"error": "repo_not_found", "message": "Repository not found or no access", "repo": "<owner/repo>", "hint": "Verify repo name and gh auth scopes"}` |
| gh API rate limit exceeded | `{"error": "rate_limit", "message": "GitHub API rate limit exceeded", "hint": "Wait and retry, or check gh auth status"}` |
| Ambiguous reference (bare #NNN, no repo context) | `{"error": "ambiguous_reference", "message": "Cannot resolve bare issue number without repo context", "reference": "<input>", "hint": "Use repo#NNN or owner/repo#NNN format"}` |
| gh CLI not authenticated | `{"error": "not_authenticated", "message": "gh CLI is not authenticated", "hint": "Run gh auth login"}` |
| Network or other gh error | `{"error": "gh_error", "message": "gh CLI request failed", "details": "<stderr output>", "hint": "Run /tracker:doctor to check gh setup"}` |

For all errors, also include a human-readable `message` field summarizing the problem.

## Rules

- Always return JSON — never plain prose as the skill output
- Truncate `body` to first 500 characters; include `body_excerpt` field
- Include only last 3 `comments` (most recent by `createdAt`)
- Never mutate issues — this skill is read-only
- When resolving `repo#NNN`, always scope to the default owner from `.local.md`
- If `.local.md` is missing and owner cannot be determined, use `heurema` as fallback
- For search queries, limit to 20 results (GitHub default ordering by relevance)
