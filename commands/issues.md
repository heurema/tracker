---
description: |
  List GitHub Issues across heurema org repos.
  Dual output: Markdown table for humans, JSON for agents.
  Filters: --state, --label, --assignee, --repo, --owner, --json
  Triggers: "/tracker:issues", "show issues", "list open issues"
argument-hint: "[--state open|closed|all] [--label <label>] [--assignee <user>] [--repo <repo>] [--owner <org>] [--json]"
allowed-tools: Bash, Read
---

You are the issues viewer for tracker. List GitHub Issues across the heurema org using filters provided by the user.

## Step 1: Check gh CLI availability

```bash
command -v gh >/dev/null 2>&1 && echo "GH_OK" || echo "GH_MISSING"
```

If output is "GH_MISSING", stop and tell the user:

> **Error:** `gh` CLI not found — command not found.
> Install from https://cli.github.com/ then run `/tracker:doctor` to verify setup.

## Step 2: Check gh authentication

```bash
gh auth status 2>&1
```

If output contains "not logged in" or "not authenticated", stop and tell the user:

> **Error:** Not authenticated. Run `gh auth login` then retry.
> Run `/tracker:doctor` for a full diagnostic.

## Step 3: Parse arguments

Parse the user's arguments from `$ARGUMENTS` (the text after the command). Supported flags:

- `--state <value>` — open, closed, or all (default: open)
- `--label <value>` — filter by label (comma-separated for multiple)
- `--assignee <value>` — filter by GitHub username
- `--repo <value>` — limit to single repo name (scoped to --owner)
- `--owner <value>` — org or user (default: heurema)
- `--json` — output structured JSON instead of Markdown table

Extract defaults: STATE=open, OWNER=heurema, JSON_MODE=false

**Validate --state**: If `--state` is provided, it must be one of: `open`, `closed`, `all`. Any other value → stop and tell the user:

> **Error:** Invalid --state value `<value>`. Use: open, closed, or all.

## Step 4: Load repo scope

Read allowed repos from .local.md config. This is the allowlist of repos to search within.

```bash
PLUGIN_ROOT="${CLAUDE_PLUGIN_ROOT:-.}"
cat "$PLUGIN_ROOT/.local.md" 2>/dev/null || echo "REPOS_NOT_FOUND"
```

If .local.md is available, extract the `repos:` list as the allowlist.

**Allowlist enforcement** (fail-closed):
- If `--repo` is specified and the allowlist is loaded, validate that the requested repo is in the allowlist. If not, stop with the error below.
- If `--repo` is specified but `.local.md` is missing (REPOS_NOT_FOUND), stop with the error below — do not allow arbitrary repo access without a configured allowlist.
- If `--owner` is specified and differs from the `owner:` value in `.local.md`, stop with the error below — the allowlist is scoped to the configured owner only.

> **Error:** Repo `<repo>` is not in the tracker allowlist, or `--owner` doesn't match the configured owner (or allowlist not configured). Edit `.local.md` to update, or omit `--repo`/`--owner` to use defaults.

**Note:** When `--repo` is omitted, the query searches all repos visible to your gh auth token within the owner org. The allowlist restricts `--repo` targeting only; it does not filter org-wide results. This is by design for v0.1.0.

## Step 5a: Human path (no --json flag)

Build the gh search command with active filters and run it:

```bash
# Build search query
OWNER="${OWNER:-heurema}"
STATE="${STATE:-open}"
REPO_FILTER="${REPO:-}"

# Build command as array for safe argument handling
CMD=(gh search issues)

# Add repo/owner scope
if [ -n "$REPO_FILTER" ]; then
  CMD+=(--repo "${OWNER}/${REPO_FILTER}")
else
  CMD+=(--owner "${OWNER}")
fi

# Add state filter (gh search issues only accepts open|closed, not all)
if [ "$STATE" = "open" ] || [ "$STATE" = "closed" ]; then
  CMD+=(--state "${STATE}")
fi

# Add label filter (quoted)
if [ -n "$LABEL" ]; then
  CMD+=(--label "${LABEL}")
fi

# Add assignee filter (quoted)
if [ -n "$ASSIGNEE" ]; then
  CMD+=(--assignee "${ASSIGNEE}")
fi

CMD+=(--json number,title,state,url,repository,labels,assignees,updatedAt --limit 1000)

"${CMD[@]}" 2>&1
```

If the command fails with "command not found", display:

> **Error:** gh CLI not found — command not found. Install from https://cli.github.com/

Format the JSON results as a Markdown table with columns:

| Repo | Issue# | Title | State | Labels | Assignees | Updated |
|------|--------|-------|-------|--------|-----------|---------|

- Repo: extract `repository.nameWithOwner`, strip "heurema/" prefix for brevity
- Issue#: number as `#NNN`
- Title: truncate to 60 chars if needed, link to url
- State: open or closed
- Labels: join label names with `, `
- Assignees: join login names with `, ` (empty = unassigned)
- Updated: format updatedAt as `YYYY-MM-DD`

Show total count above the table: `**N issues** (state: STATE, owner: OWNER)`

If zero results, display: `No issues found matching the given filters.`

## Step 5b: Agent path (--json flag present)

Build and run the GraphQL query with pagination:

```bash
OWNER="${OWNER:-heurema}"
STATE="${STATE:-open}"

# Build search query string — all filters compose (never overwrite)
SEARCH_QUERY="is:issue"

# Scope: repo-specific or org-wide
if [ -n "$REPO" ]; then
  SEARCH_QUERY="${SEARCH_QUERY} repo:${OWNER}/${REPO}"
else
  SEARCH_QUERY="${SEARCH_QUERY} org:${OWNER}"
fi

# State filter
if [ "$STATE" = "open" ] || [ "$STATE" = "closed" ]; then
  SEARCH_QUERY="${SEARCH_QUERY} is:${STATE}"
fi

# Label filter — split comma-separated values into individual qualifiers
if [ -n "$LABEL" ]; then
  IFS=',' read -ra LABELS <<< "$LABEL"
  for lbl in "${LABELS[@]}"; do
    lbl=$(echo "$lbl" | xargs)  # trim whitespace
    SEARCH_QUERY="${SEARCH_QUERY} label:\"${lbl}\""
  done
fi

# Assignee filter (preserved regardless of --repo)
if [ -n "$ASSIGNEE" ]; then
  SEARCH_QUERY="${SEARCH_QUERY} assignee:${ASSIGNEE}"
fi

# Pass query as a GraphQL variable to avoid shell injection
gh api graphql --paginate --slurp \
  -f searchQuery="$SEARCH_QUERY" \
  -f query='
query($endCursor: String, $searchQuery: String!) {
  search(query: $searchQuery, type: ISSUE, first: 100, after: $endCursor) {
    issueCount
    pageInfo { hasNextPage endCursor }
    nodes {
      ... on Issue {
        number
        title
        state
        url
        repository { nameWithOwner }
        labels(first: 10) { nodes { name } }
        assignees(first: 5) { nodes { login } }
        createdAt
        updatedAt
      }
    }
  }
}' 2>&1
```

Output the raw JSON as-is. Do not wrap or reformat. This is the structured agent output.

## Rules

- Always check gh availability before running any gh commands
- Surface "command not found" errors clearly — do not swallow them
- Human path: Markdown table, readable and concise
- Agent path: raw JSON, no extra prose
- Filters compose with AND logic — all active filters must match
- Never modify any issues — this command is read-only
