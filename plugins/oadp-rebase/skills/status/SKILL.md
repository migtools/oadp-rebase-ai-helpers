---
name: Status Command
description: Check current rebase status across all OADP repositories for a branch
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

### Phase 1: Set Up the oadp-rebase Repository
Do NOT rely on the oadp-rebase repo being in the current working directory or home directory. Re-use a previous clone if available, otherwise clone fresh:
```bash
OADP_REBASE_DIR="/tmp/oadp-rebase-tools"
if [ -d "$OADP_REBASE_DIR/.git" ]; then
  cd "$OADP_REBASE_DIR"
  git fetch origin
  git reset --hard origin/oadp-dev
else
  git clone https://github.com/oadp-rebasebot/oadp-rebase.git "$OADP_REBASE_DIR" --branch oadp-dev --single-branch
  cd "$OADP_REBASE_DIR"
fi
```
This ensures the latest configs are used. All config files are in `$OADP_REBASE_DIR/rebase-configs/`.

### Phase 2: Load All Configs
1. Read all `rebase-configs/*_{branch}.env.sh` files from the cloned oadp-rebase repo
2. For each config, extract:
   - `SOURCE_UPSTREAM_REPO` (upstream version)
   - `DESTINATION_DOWNSTREAM_REPO` (downstream target)
   - `SKIP_REPO` flag
3. Determine if it's a hooks-only repo (source org/repo == dest org/repo)

### Phase 3: Check PRs
The rebase bot uses a GitHub App identity (`app/oadp-rebasebot-app`), NOT a regular user account. Do NOT use `--author "oadp-rebasebot"` — it will not match.

Instead, filter PRs by the **head branch name** `rebase-bot-{branch}`, which is the branch the bot always pushes to:

1. For each repository, check for open rebase PRs:
   ```bash
   gh pr list --repo {org}/{repo} --head "rebase-bot-{branch}" --state open --json number,title,url,createdAt --limit 3
   ```
2. Check recently merged rebase PRs:
   ```bash
   gh pr list --repo {org}/{repo} --head "rebase-bot-{branch}" --state merged --json number,title,url,mergedAt --limit 3
   ```

### Phase 4: Generate Report
1. Organize by wave (see wave ordering in `/oadp-rebase:rebase`)
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
