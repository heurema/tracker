# Changelog

## 0.3.0

- New `issue-context` skill: inter-plugin composition skill that resolves issue references and returns structured JSON summaries
- Resolves `#123`, `repo#45`, and `owner/repo#45` reference formats
- Fetches full issue details via `gh issue view` (title, state, labels, assignees, URL, timestamps, last 3 comments)
- Search mode via `gh search issues` for free-text queries
- Structured JSON error responses for all failure conditions
- `issue-context` capability added to `plugin.json`

## 0.2.0

- New `/tracker:triage` command: daily digest of GitHub Issues with Markdown section output
- Grouping support: `--group-by repo|label|priority` (default: repo)
- `--since` flag for time-based filtering (default: 24h); supports `24h`, `48h`, `7d`, `1w`, `14d`, `30d`
- Hot issue highlighting: most commented, most recently updated, newly created within `--since` window
- Priority grouping via label matching: `critical` / `high` / `medium` / `low`
- `triage` capability added to `plugin.json`

## 0.1.0

- Dual-path `/tracker:issues` command: human Markdown table + agent GraphQL JSON
- Filter support: `--state`, `--label`, `--assignee`, `--repo`, `--owner`
- `/tracker:doctor` diagnostic command with gh CLI, auth, and repo scope checks
- Repo scope config via `.local.md` (heurema org ~15 repos allowlist)
