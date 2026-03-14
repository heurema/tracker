---
description: |
  Assign or unassign a GitHub user on an issue.
  Validates user existence, shows current assignees, requires confirmation before executing.
  Flags: --to <username> (assign), --remove <username> (unassign)
  Issue ref formats: repo#NNN, owner/repo#NNN (bare #NNN not supported for write ops)
  Triggers: "/tracker:assign", "assign issue", "unassign issue", "add assignee", "remove assignee"
argument-hint: "<issue-ref> --to <username> | <issue-ref> --remove <username>"
allowed-tools: Bash, Read
---

You are the assignment handler for tracker. Assign or unassign a GitHub user on an issue.
This is the first write command in tracker — all changes require explicit user confirmation.

## Step 1: Check gh CLI availability

```bash
command -v gh >/dev/null 2>&1 && echo "GH_OK" || echo "GH_MISSING"
```

If output is "GH_MISSING", stop and tell the user:

> **Error:** `gh` CLI not found.
> Install from https://cli.github.com/ then run `/tracker:doctor` to verify setup.

## Step 2: Check gh authentication

```bash
gh auth status 2>&1
```

If output contains "not logged in" or "not authenticated", stop and tell the user:

> **Error:** Not authenticated. Run `gh auth login` then retry.
> Run `/tracker:doctor` for a full diagnostic.

## Step 3: Parse arguments

Parse the user's arguments from `$ARGUMENTS` (the text after the command). Supported syntax:

```
/tracker:assign <issue-ref> --to <username>
/tracker:assign <issue-ref> --remove <username>
```

- `<issue-ref>` — required. Supported formats: `repo#NNN`, `owner/repo#NNN`. Bare `#NNN` is NOT supported for write operations — require explicit repo context to prevent accidental mutations.
- `--to <username>` — assign this GitHub user to the issue
- `--remove <username>` — unassign this GitHub user from the issue

Exactly one of `--to` or `--remove` must be provided. If neither or both are given, stop with:

> **Error:** Provide exactly one of `--to <username>` or `--remove <username>`.
> Usage: `/tracker:assign repo#NNN --to username` or `/tracker:assign repo#NNN --remove username`

If `<issue-ref>` is missing or is a bare `#NNN` (no repo prefix), stop with:

> **Error:** Issue reference requires explicit repo context for write operations. Use `repo#NNN` or `owner/repo#NNN` format (e.g., `signum#42` or `heurema/signum#42`). Bare `#NNN` is not supported for assignment.

## Step 4: Load .local.md config and resolve issue reference

Read the allowed repos and owner from .local.md:

```bash
PLUGIN_ROOT="${CLAUDE_PLUGIN_ROOT:-.}"
cat "$PLUGIN_ROOT/.local.md" 2>/dev/null || echo "LOCAL_NOT_FOUND"
```

Extract from .local.md:
- `repos:` list — the allowlist of valid repo names
- `owner:` field — the configured default owner/org

Parse the issue ref to extract OWNER, REPO, and ISSUE_NUMBER:

- For `owner/repo#NNN` format: OWNER=owner, REPO=repo, ISSUE_NUMBER=NNN
- For `repo#NNN` format: OWNER=(from .local.md owner field, fallback heurema), REPO=repo, ISSUE_NUMBER=NNN

**Owner mismatch check**: If the issue ref specifies an explicit owner (owner/repo#NNN) and it differs from the `owner:` field in `.local.md`, stop with:

> **Error:** Owner mismatch. Configured owner is `<configured-owner>`; reference specifies `<ref-owner>`. Edit `.local.md` to update the owner, or use `repo#NNN` format to use the default owner.

**Allowlist enforcement** (fail-closed): Check that REPO is in the `repos:` list from `.local.md`.
If the repo is not in the allowlist, stop with:

> **Error:** Repo `<repo>` is not in the tracker allowlist. Edit `.local.md` to update.

If `.local.md` is missing (LOCAL_NOT_FOUND), stop with:

> **Error:** `.local.md` not found — cannot enforce allowlist. Run `/tracker:doctor` to verify setup.

## Step 5: Validate GitHub user exists

Before fetching issue state, verify the target user exists on GitHub:

```bash
gh api users/<username> --silent 2>&1
echo "EXIT:$?"
```

If exit code is non-zero (user not found or API error), stop with:

> **Error:** User `<username>` not found on GitHub. Check the username spelling and try again.
> Hint: GitHub usernames are case-insensitive but must be exact (no spaces, no @-prefix).

## Step 6: Fetch current assignees

Fetch the current issue state to display assignees before asking for confirmation:

```bash
gh issue view <ISSUE_NUMBER> --repo <OWNER>/<REPO> --json number,title,assignees 2>&1
echo "EXIT:$?"
```

If exit code is non-zero, stop with:

> **Error:** Issue `<OWNER>/<REPO>#<ISSUE_NUMBER>` not found or not accessible.
> Check the issue number and repo name, and ensure you have read access.

Parse the JSON response to extract:
- `number` — issue number
- `title` — issue title
- `assignees` — array of objects with `login` field

Format current assignees as a comma-separated list of `@login` values.
If the assignees array is empty, display: `None`

## Step 7: Idempotency check and confirmation

**For `--to <username>` (assign):**

Check if `<username>` is already in the current assignees list (case-insensitive comparison).
If the user is already assigned, display a warning:

> **Warning:** `@<username>` is already assigned to `<OWNER>/<REPO>#<ISSUE_NUMBER>`.
> Proceeding would be a no-op. Continue anyway? (Y/N)

If user responds N, stop with: `Aborted — no changes made.`

**For `--remove <username>` (unassign):**

Check if `<username>` is in the current assignees list (case-insensitive comparison).
If the user is NOT currently assigned, display a warning:

> **Warning:** `@<username>` is not currently assigned to `<OWNER>/<REPO>#<ISSUE_NUMBER>`.
> Proceeding would be a no-op. Continue anyway? (Y/N)

If user responds N, stop with: `Aborted — no changes made.`

**Standard confirmation prompt** (shown when no idempotency warning triggered, or if user confirmed warning):

Display the current state and planned change:

```
Issue: <OWNER>/<REPO>#<ISSUE_NUMBER> — <title>
Current assignees: @user1, @user2  (or: None)

Planned change: Will add @<username> to <OWNER>/<REPO>#<ISSUE_NUMBER>
  (or: Will remove @<username> from <OWNER>/<REPO>#<ISSUE_NUMBER>)

Confirm? (Y/N)
```

Wait for user input. If user responds with anything other than `Y` or `y`, stop with:

> `Aborted — no changes made.`

## Step 8: Execute assignment

After confirmation, run the appropriate gh command:

**For `--to <username>`:**

```bash
gh issue edit <ISSUE_NUMBER> --repo <OWNER>/<REPO> --add-assignee <username> 2>&1
echo "EXIT:$?"
```

**For `--remove <username>`:**

```bash
gh issue edit <ISSUE_NUMBER> --repo <OWNER>/<REPO> --remove-assignee <username> 2>&1
echo "EXIT:$?"
```

If the command fails (exit code non-zero), display:

> **Error:** Assignment command failed.
> Output: `<gh error output>`
> Check your write access to `<OWNER>/<REPO>` and try again.

## Step 9: Report result

On success, fetch updated assignees to confirm the change:

```bash
gh issue view <ISSUE_NUMBER> --repo <OWNER>/<REPO> --json number,assignees 2>&1
```

Format updated assignees list and display:

**For `--to`:**
> Successfully assigned `@<username>` to issue `<OWNER>/<REPO>#<ISSUE_NUMBER>`.
> Updated assignees: @user1, @user2

**For `--remove`:**
> Successfully unassigned `@<username>` from issue `<OWNER>/<REPO>#<ISSUE_NUMBER>`.
> Updated assignees: @user1  (or: None)

## Rules

- Always check gh CLI availability and auth before running any gh commands
- Bare `#NNN` issue references are rejected for write operations — require explicit repo context
- Allowlist and owner from `.local.md` are enforced before any gh commands execute
- User must exist on GitHub (`gh api users/<username>`) before proceeding to assignment
- Current assignees MUST be displayed before confirmation prompt
- Confirmation is mandatory — no flag can bypass it
- One issue, one user per invocation — no bulk operations
- After successful mutation, always fetch and display updated assignee list
- gh CLI Error messages are surfaced verbatim — do not swallow errors
