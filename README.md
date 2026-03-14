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
