# Changelog

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
