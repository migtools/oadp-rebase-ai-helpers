---
description: Check current rebase status across all OADP repositories for a branch
argument-hint: [--branch <oadp-branch>] [--wave <wave-number>]
---

## Name
oadp-rebase:status

## Synopsis
```
/oadp-rebase:status [--branch <oadp-branch>] [--wave <wave-number>]
```

## Description
The `oadp-rebase:status` command provides an overview of the current rebase state across all OADP repositories for a given branch. It checks what upstream version each repo is currently tracking, whether there are open rebase PRs, and identifies repos that need attention.

### Information Gathered
For each repository:
- Current upstream source version (from config file)
- Whether the downstream branch exists
- Open rebase PRs (if any)
- Last rebase PR (merged or closed)
- Whether the repo is skipped (`SKIP_REPO=true`)
- Whether it's a hooks-only repo (source == dest)

## Implementation

### Phase 1: Load All Configs
1. Read all `rebase-configs/*_{branch}.env.sh` files
2. For each config, extract:
   - `SOURCE_UPSTREAM_REPO` (upstream version)
   - `DESTINATION_DOWNSTREAM_REPO` (downstream target)
   - `SKIP_REPO` flag
3. Determine if it's a hooks-only repo (source org/repo == dest org/repo)

### Phase 2: Check PRs
1. For each repository, check for open PRs from `oadp-rebasebot`:
   ```bash
   gh pr list --repo {org}/{repo} --author "oadp-rebasebot" --state open
   ```
2. Check recently merged rebase PRs:
   ```bash
   gh pr list --repo {org}/{repo} --author "oadp-rebasebot" --state merged --limit 3
   ```

### Phase 3: Generate Report
1. Organize by wave
2. For each repo show:
   - Wave number
   - Repo name
   - Current upstream version
   - Rebase type (full / hooks-only / skipped)
   - PR status (open/merged/none)
3. Highlight repos that need attention

## Return Value
A formatted table showing the rebase status of all repositories, organized by wave.

## Examples

1. **Check all repos for oadp-1.6**:
   ```
   /oadp-rebase:status --branch oadp-1.6
   ```

2. **Check Wave 3 status**:
   ```
   /oadp-rebase:status --branch oadp-1.6 --wave 3
   ```

## Arguments
- `--branch <branch>`: OADP branch to check (default: `oadp-dev`)
- `--wave <number>`: Optional wave filter (1-5)

## See Also
- `/oadp-rebase:rebase` - Run a rebase
- `/oadp-rebase:update-config` - Update upstream version in config
