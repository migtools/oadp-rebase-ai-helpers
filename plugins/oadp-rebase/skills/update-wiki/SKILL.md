---
name: Update Wiki Command
description: Generate rebase status markdown and update the oadp-operator wiki page
---

## Name
oadp-rebase:update-wiki

## Synopsis
```
/oadp-rebase:update-wiki [--branch <oadp-branch>]
```

## Description
The `oadp-rebase:update-wiki` command generates the current rebase status using the `rebase-status` tool and pushes the output to the corresponding wiki page at `https://github.com/openshift/oadp-operator/wiki/`.

### Wiki Pages
Each OADP branch has its own wiki page:
- `oadp-1.6` → `Rebase-status-‐-oadp‐1.6`
- `oadp-dev` → `Rebase-status-‐-oadp‐dev`
- `oadp-1.5` → `Rebase-status-‐-oadp‐1.5`

The wiki page naming uses Unicode hyphens (U+2010 `‐`) as separators in the URL slug, not regular ASCII hyphens.

### What rebase-status Checks
The tool reads all `rebase-configs/*_{branch}.env.sh` files and for each repository checks:
- Whether the downstream branch is up-to-date with upstream
- Open/merged rebase PRs from oadp-rebasebot
- Dependency sync status (e.g., does the plugin use the correct downstream velero commit?)
- Upstream container image availability on Quay
- Productization status (image-references)

## CRITICAL: Automation Principles

**Execute end-to-end without asking unnecessary questions.** The only reason to stop is if a step fails (build error, wiki push error, missing GitHub token).

## Implementation

### Phase 1: Set Up Tools

1. **Set up the oadp-rebase repository** (re-use if already cloned):
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

2. **Build the rebase-status tool**:
   ```bash
   cd "$OADP_REBASE_DIR/tools/rebase-status"
   go build -o rebase-status .
   ```
   If `go` is not available on the host, check if there's a pre-built binary. If not, report the error — the tool requires Go to build.

3. **Verify GitHub token** is available. The tool needs `GITHUB_TOKEN` for API calls:
   ```bash
   # gh CLI token works
   export GITHUB_TOKEN=$(gh auth token)
   ```

### Phase 2: Generate Markdown

1. **Run rebase-status with markdown output**:
   ```bash
   cd "$OADP_REBASE_DIR"
   ./tools/rebase-status/rebase-status --format markdown {branch} > /tmp/rebase-status-{branch}.md
   ```
   The `--config-dir` flag is auto-detected when running from the oadp-rebase repo root.

2. **Review the output** — quickly verify it looks reasonable (has wave tables, repo entries, no error messages in the markdown).

### Phase 3: Update Wiki

GitHub wikis are git repositories. Clone, update the page, push.

1. **Clone the wiki repo** (re-use if already cloned):
   ```bash
   WIKI_DIR="/tmp/oadp-operator-wiki"
   if [ -d "$WIKI_DIR/.git" ]; then
     cd "$WIKI_DIR"
     git pull --ff-only
   else
     gh repo clone openshift/oadp-operator.wiki "$WIKI_DIR"
     cd "$WIKI_DIR"
   fi
   ```

2. **Determine the wiki page filename**. GitHub wiki files use the page title as filename with `.md` extension. The URL slug uses Unicode hyphens but the actual filename uses regular characters:
   ```bash
   # Map branch to wiki page filename
   # The wiki page names use Unicode hyphen ‐ (U+2010) in the URL
   # but the actual .md files on disk may use different naming
   # List existing files to find the correct one:
   ls -la "$WIKI_DIR/" | grep -i "rebase-status"
   ```
   Expected filenames (check actual names in the cloned wiki):
   - `Rebase-status-‐-oadp‐1.6.md` (or similar with Unicode hyphens)
   - If the file doesn't exist yet, create it with the appropriate name

3. **Write the generated markdown** to the wiki page file:
   ```bash
   cp /tmp/rebase-status-{branch}.md "$WIKI_DIR/{wiki-page-filename}"
   ```

4. **Commit and push**:
   ```bash
   cd "$WIKI_DIR"
   git add -A
   git commit -m "Update rebase status for {branch} - $(date +%Y-%m-%d)"
   git push
   ```

### Phase 4: Report

Print the wiki page URL:
```
Wiki updated: https://github.com/openshift/oadp-operator/wiki/Rebase-status-‐-oadp‐{branch}
```

## Return Value
- **On success**: URL of the updated wiki page
- **On failure**: Error details (build failure, missing token, push failure)

## Examples

1. **Update wiki for oadp-1.6**:
   ```
   /oadp-rebase:update-wiki --branch oadp-1.6
   ```

2. **Update wiki for oadp-dev**:
   ```
   /oadp-rebase:update-wiki --branch oadp-dev
   ```

3. **Update wiki for oadp-dev (default)**:
   ```
   /oadp-rebase:update-wiki
   ```

## Arguments
- `--branch <branch>`: OADP branch to generate status for (default: `oadp-dev`)

## See Also
- `/oadp-rebase:status` - Check rebase status interactively (terminal output)
- `/oadp-rebase:rebase` - Run a rebase
