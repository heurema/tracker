---
description: |
  Daily digest of GitHub Issues across heurema org repos.
  Groups issues by repo, label, or priority. Highlights 'hot' issues:
  most commented, most recently updated, newly created within --since window.
  Filters: --since, --group-by, --owner
  Triggers: "/tracker:triage", "daily digest", "triage issues", "show hot issues"
argument-hint: "[--since <duration>] [--group-by repo|label|priority] [--owner <org>]"
allowed-tools: Bash, Read
---

You are the triage digest generator for tracker. Produce a daily digest of GitHub Issues
grouped by repo, label, or priority with hot issue highlighting.

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

- `--since <duration>` — time window for filtering issues by updatedAt (default: 24h). Supports: `24h`, `48h`, `7d`, `1w`, `14d`, `30d`.
- `--group-by <value>` — how to group results: `repo`, `label`, or `priority` (default: repo)
- `--owner <value>` — org or user (default: heurema)

Extract defaults: SINCE=24h, GROUP_BY=repo, OWNER=(from .local.md, fallback heurema)

**Validate --group-by**: If `--group-by` is provided, it must be one of: `repo`, `label`, `priority`. Any other value → stop and tell the user:

> **Error:** Invalid --group-by value `<value>`. Use: repo, label, or priority.

**Validate --since**: If `--since` is provided, it must match the pattern `<number>h`, `<number>d`, or `<number>w` (e.g., 24h, 7d, 1w). Any other value → stop and tell the user:

> **Error:** Invalid --since value `<value>`. Use format: NNh (hours), NNd (days), or NNw (weeks). Examples: 24h, 7d, 1w.

## Step 4: Parse --since into ISO date

Convert the `--since` duration to an ISO 8601 date string for use in GitHub search queries.

```bash
SINCE="${SINCE:-24h}"

# Parse duration: support NNh, NNd, NNw formats
if echo "$SINCE" | grep -qE '^[0-9]+h$'; then
  HOURS=$(echo "$SINCE" | tr -d 'h')
  SINCE_DATE=$(date -u -v -${HOURS}H '+%Y-%m-%dT%H:%M:%SZ' 2>/dev/null || date -u -d "${HOURS} hours ago" '+%Y-%m-%dT%H:%M:%SZ' 2>/dev/null)
elif echo "$SINCE" | grep -qE '^[0-9]+d$'; then
  DAYS=$(echo "$SINCE" | tr -d 'd')
  SINCE_DATE=$(date -u -v -${DAYS}d '+%Y-%m-%dT%H:%M:%SZ' 2>/dev/null || date -u -d "${DAYS} days ago" '+%Y-%m-%dT%H:%M:%SZ' 2>/dev/null)
elif echo "$SINCE" | grep -qE '^[0-9]+w$'; then
  WEEKS=$(echo "$SINCE" | tr -d 'w')
  DAYS=$(( WEEKS * 7 ))
  SINCE_DATE=$(date -u -v -${DAYS}d '+%Y-%m-%dT%H:%M:%SZ' 2>/dev/null || date -u -d "${DAYS} days ago" '+%Y-%m-%dT%H:%M:%SZ' 2>/dev/null)
else
  # This branch should not be reached if --since was validated in Step 3.
  # If it is reached, it means validation was skipped — fail explicitly.
  echo "SINCE_INVALID"
fi

# Extract date portion only for GitHub search qualifier (updated:>YYYY-MM-DD)
SINCE_DATE_ONLY=$(echo "$SINCE_DATE" | cut -c1-10)
echo "SINCE_DATE=$SINCE_DATE"
echo "SINCE_DATE_ONLY=$SINCE_DATE_ONLY"
```

## Step 5: Load repo scope

Read allowed repos from .local.md config. This is the allowlist of repos to search within.

```bash
PLUGIN_ROOT="${CLAUDE_PLUGIN_ROOT:-.}"
cat "$PLUGIN_ROOT/.local.md" 2>/dev/null || echo "REPOS_NOT_FOUND"
```

If .local.md is available, extract the `repos:` list and `owner:` value as the allowlist.

**Default OWNER from config:** If the user did not pass `--owner`, use the `owner:` value from `.local.md` (if present) instead of the hardcoded default. Only fall back to "heurema" if `.local.md` is missing or has no `owner:` field.

**Allowlist enforcement** (fail-closed):
- If `--owner` is specified and differs from the `owner:` value in `.local.md`, stop with the error below — the allowlist is scoped to the configured owner only.
- If `.local.md` is missing (REPOS_NOT_FOUND), proceed with owner-scoped search (no allowlist restriction).

> **Error:** `--owner` doesn't match the configured owner in `.local.md`. Edit `.local.md` to update, or omit `--owner` to use defaults.

## Step 6: Fetch issues via GraphQL

Fetch all open issues updated within the --since window, with comment counts for hot issue detection.

**Note:** `gh search issues` lacks comment count fields, so we use `gh api graphql` for full metadata
including `comments.totalCount`. The GraphQL approach subsumes what `gh search issues --updated` would provide.

```bash
OWNER="${OWNER:-heurema}"
SINCE_DATE_ONLY="${SINCE_DATE_ONLY:-$(date -u -v -1d '+%Y-%m-%d' 2>/dev/null || date -u -d 'yesterday' '+%Y-%m-%d')}"

# Build GitHub search query
SEARCH_QUERY="is:issue is:open org:${OWNER} updated:>${SINCE_DATE_ONLY}"

# Execute GraphQL query to get issues with comment counts
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
        createdAt
        updatedAt
        repository { nameWithOwner }
        labels(first: 10) { nodes { name } }
        comments { totalCount }
      }
    }
  }
}' 2>&1
```

If the command fails with "command not found", display:

> **Error:** gh CLI not found — command not found. Install from https://cli.github.com/

Capture the JSON output into ISSUES_JSON.

**Error detection**: Check the GraphQL response for errors before treating it as data:
- If the output contains `"errors"` key at the top level, display the error message from the response and stop:
  > **Error:** GitHub API returned an error: `<error message>`. Check your auth with `/tracker:doctor`.
- If the command exits with non-zero status, display:
  > **Error:** gh API request failed. Run `/tracker:doctor` to check auth and scopes.
- Only if the response is valid JSON with zero `issueCount`, display:
  > No issues found updated since `<SINCE>` in the `<OWNER>` org.

## Step 7: Build Markdown digest

Using the JSON output from Step 6, build a Markdown digest. This step requires the AI agent
to parse the JSON and generate Markdown sections. The output format uses `##` headers per group,
issue counts, and hot issue highlighting.

**Parsing the response**: The GraphQL response is an array (due to --slurp) where each element
has `.data.search.nodes`. Flatten all nodes across all pages before processing.

**Hot issue detection criteria** (mark with a visual indicator like a bullet prefix or bold):
- Most commented: top 3 issues by `comments.totalCount` within the current group
- Most recently updated: issues with `updatedAt` within the last 6 hours relative to now
- Newly created: issues with `createdAt` within the --since window

**Grouping logic:**

For `--group-by repo` (default):
- Group issues by `repository.nameWithOwner`
- Strip owner prefix for group header (e.g., "heurema/signum" → "signum")
- Header: `## signum`

For `--group-by label`:
- Each issue appears in every group matching its labels
- Issues with no labels appear under `## (unlabeled)`
- Header: `## bug`, `## enhancement`, etc.

For `--group-by priority`:
- Map labels to priority tiers: `critical` → Critical, `high` → High, `medium` → Medium, `low` → Low
- Match label names containing these patterns (case-insensitive)
- Issues with no matching priority labels appear under `## (no priority)`
- Header: `## critical`, `## high`, `## medium`, `## low`, `## (no priority)`
- Sort groups: critical → high → medium → low → no priority

**Per-group section format:**

```
## <group-name>

**N issues** (updated since <SINCE>)

- [#NNN] [Title](url) — updated YYYY-MM-DD, N comments [HOT]
- [#NNN] [Title](url) — updated YYYY-MM-DD, N comments
```

Mark hot issues with `**[HOT]**` suffix when they meet any hot criterion.

**Header block** (before all groups):

```markdown
# Tracker Triage Digest

**Owner:** heurema  **Since:** 24h (2026-03-13)  **Group by:** repo  **Total:** N issues

---
```

**Footer** (after all groups):

```markdown
---
*Generated by /tracker:triage — read-only, no issues were modified.*
```

If zero results across all groups, display:

```
No open issues found updated since <SINCE> in the <OWNER> org.
```

## Rules

- Always check gh availability before running any gh commands
- Surface "command not found" errors clearly — do not swallow them
- Human path: Markdown sections, readable and scannable
- Hot issues: visual highlighting only — no persistent state written
- Grouping: issues may appear in multiple groups (label/priority modes)
- Never modify any issues — this command is read-only
- The `--since` default is 24h — always apply time filtering even if not specified
