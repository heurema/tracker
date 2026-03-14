# tracker

GitHub Issues read layer for the heurema org. Cross-repo issue tracking with
human-readable Markdown tables and structured agent JSON output.

**Read-only.** For writing issues (bug reports, feature requests), use the
[Reporter plugin](https://github.com/heurema/reporter).

## Installation

```bash
claude plugin add devtools/tracker
```

## Commands

### /tracker:issues

List GitHub Issues across heurema org repos with optional filters.

```
/tracker:issues [--state open|closed|all] [--label <label>] [--assignee <user>] [--repo <repo>] [--owner <org>] [--json]
```

**Examples:**

```bash
# All open issues across the org
/tracker:issues

# Open bugs only
/tracker:issues --state open --label bug

# Issues assigned to a specific user
/tracker:issues --assignee vi

# Issues in a single repo
/tracker:issues --repo signum

# Closed issues with a specific label
/tracker:issues --state closed --label enhancement

# Agent-friendly JSON output
/tracker:issues --json

# JSON with filters for agent composition
/tracker:issues --state open --label needs-review --json
```

**Filters:**

| Flag | Default | Description |
|------|---------|-------------|
| `--state` | `open` | Issue state: `open`, `closed`, or `all` |
| `--label` | — | Filter by label name (comma-separated) |
| `--assignee` | — | Filter by GitHub username |
| `--repo` | — | Limit to single repo name (scoped to `--owner`) |
| `--owner` | `heurema` | GitHub org or user |
| `--json` | off | Switch to structured JSON output for agents |

All filters are optional and compose with AND logic.

### Dual Output Paths

**Human path** (default): Markdown table with columns Repo, Issue#, Title, State,
Labels, Assignees, Updated. Designed for terminal reading and copy-paste.

**Agent path** (`--json`): GraphQL query via `gh api graphql --paginate --slurp`
returning structured JSON with full issue details: number, title, state, url,
repository.nameWithOwner, labels, assignees, createdAt, updatedAt. Pagination
is handled automatically — all pages are merged into a single output.

### /tracker:triage

Generate a daily digest of GitHub Issues with grouping and hot issue highlighting.

```
/tracker:triage [--since <duration>] [--group-by repo|label|priority] [--owner <org>]
```

**Examples:**

```bash
# Default daily digest — open issues updated in last 24h, grouped by repo
/tracker:triage

# Weekly digest
/tracker:triage --since 7d

# Group by label
/tracker:triage --group-by label

# Group by priority (uses critical/high/medium/low labels)
/tracker:triage --group-by priority

# Two-week digest grouped by priority
/tracker:triage --since 14d --group-by priority

# Digest for a different org
/tracker:triage --owner myorg --since 48h
```

**Flags:**

| Flag | Default | Description |
|------|---------|-------------|
| `--since` | `24h` | Time window for filtering by `updatedAt`. Supports: `24h`, `48h`, `7d`, `1w`, `14d`, `30d` |
| `--group-by` | `repo` | Grouping strategy: `repo`, `label`, or `priority` |
| `--owner` | `heurema` | GitHub org or user scope |

**Grouping behavior:**

- `repo` — one section per repository (e.g., `## signum`, `## forge`)
- `label` — one section per label; issues with multiple labels appear in each matching section; unlabeled issues appear under `## (unlabeled)`
- `priority` — sections for `critical`, `high`, `medium`, `low` based on label name matching; issues with no priority label appear under `## (no priority)`. Groups are sorted critical → high → medium → low.

**Hot issue highlighting:**

Each group highlights hot issues with a `**[HOT]**` marker:
- Most commented (top 3 by comment count within the group)
- Most recently updated (within last 6 hours)
- Newly created within the `--since` window

The digest uses the GraphQL API to fetch `comments.totalCount`, `createdAt`, and `updatedAt`
fields for accurate hot detection.

**Output format:**

```
# Tracker Triage Digest

**Owner:** heurema  **Since:** 24h (2026-03-13)  **Group by:** repo  **Total:** 12 issues

---

## signum

**4 issues** (updated since 24h)

- [#42] [Fix pipeline timeout](url) — updated 2026-03-14, 7 comments **[HOT]**
- [#39] [Add retry logic](url) — updated 2026-03-13, 2 comments
```

### /tracker:doctor

Run diagnostics to verify tracker is correctly configured.

```
/tracker:doctor
```

Checks:
1. `gh` CLI is installed and available in PATH
2. `gh auth status` shows an authenticated user
3. Auth token includes `repo` scope (required for private repo access)

Each check reports `[OK]` or `[FAIL]` with a remediation hint.

## Repo Scope Config

Tracker ships with a `.local.md` file listing the repos it should search by default.
Edit this file to add or remove repos from your scope:

```yaml
repos:
  - signum
  - reporter
  - tracker
  - forge
  - anvil
  - herald
  - sentinel
  - arbiter
  - glyph
  - genesis
  - oracle
  - sentinel-trading
  - freqtrade
  - content-ops
  - skillpulse
```

## Inter-Plugin Composition

### issue-context skill

The `issue-context` skill enables other plugins to fetch and summarize GitHub issue context
without implementing their own gh CLI calls. It is auto-invoked when Claude detects issue
references or context-fetching intent — not via slash commands.

**Use cases:**

- Reporter plugin: check for duplicate issues before filing a new one
- Signum plugin: attach issue context during contract generation
- Any plugin: resolve an issue reference embedded in natural language

**Issue reference formats supported:**

| Format | Example | Resolution |
|--------|---------|------------|
| `#NNN` | `#123` | Default owner + repo context from `.local.md` |
| `repo#NNN` | `signum#45` | Default owner + named repo |
| `owner/repo#NNN` | `heurema/signum#45` | Fully qualified reference |
| Free text | `"auth fails on login"` | Search across org |

**Example trigger phrases:**

```
"What is the status of signum#42?"
"Get issue context for heurema/reporter#17"
"Look up issue #99 — is it still open?"
"Search for issues about pipeline timeout"
```

**Output (single issue):**

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

The skill always returns JSON — never prose. All errors are returned as structured JSON
with an `error` field and `hint` for remediation.

## Write Path

Tracker is read-only. To create issues, use the Reporter plugin:

```
/report bug        # file a bug report
/report feature    # request a feature
/report question   # ask a question
```

Reporter handles `gh issue create` — tracker only reads.

## Requirements

- [gh CLI](https://cli.github.com/) installed and authenticated
- `repo` scope on your GitHub token (`gh auth refresh --scopes repo`)
- Read access to target repos in the heurema org
