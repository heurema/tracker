---
description: |
  Diagnostic check for tracker plugin. Verifies gh CLI is installed,
  authenticated, and has required repo scope.
  Triggers: "/tracker:doctor", "check tracker setup", "diagnose tracker"
argument-hint: ""
allowed-tools: Bash
---

You are the diagnostics runner for tracker. Run three checks and report pass/fail with remediation hints.

## Run all checks

### Check 1: gh CLI in PATH

```bash
command -v gh >/dev/null 2>&1 && echo "CHECK1_OK: $(which gh)" || echo "CHECK1_FAIL"
```

- If "CHECK1_OK": print `[OK]  gh CLI found at <path>`
- If "CHECK1_FAIL": print `[FAIL] gh CLI not found in PATH` and show remediation:
  > Install gh CLI: https://cli.github.com/
  > macOS: `brew install gh`
  > Linux: see https://github.com/cli/cli/blob/trunk/docs/install_linux.md

If CHECK1 fails, skip checks 2 and 3 (they depend on gh being installed).

### Check 2: gh auth status — authenticated user

```bash
gh auth status 2>&1
```

Check if output contains a logged-in user (look for "Logged in" or "github.com" with a username).

- If authenticated: print `[OK]  gh auth: logged in`
- If not authenticated: print `[FAIL] gh auth: not authenticated` and show remediation:
  > Run: `gh auth login`
  > Select "GitHub.com" and follow prompts.

If CHECK2 fails, skip check 3.

### Check 3: repo scope present

```bash
gh auth status 2>&1
```

Inspect the output for the `repo` scope in the token scopes list.

- If "repo" appears in scopes: print `[OK]  gh auth: repo scope present`
- If "repo" not in scopes: print `[FAIL] gh auth: repo scope missing` and show remediation:
  > Re-authenticate with repo scope:
  > `gh auth login --scopes repo`
  > Or refresh: `gh auth refresh --scopes repo`

## Summary

After all checks, print a summary line:

- All passed: `All checks passed. tracker is ready.`
- Any failed: `Some checks failed. See above for remediation steps.`

## Rules

- Run checks in order: 1 → 2 → 3
- Stop at first failure if subsequent checks depend on it
- Print exact [OK] / [FAIL] prefixes — they are machine-readable
- Never modify auth state or install anything — diagnostics only
