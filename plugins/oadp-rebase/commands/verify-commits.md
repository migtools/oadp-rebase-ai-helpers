---
description: Verify that all downstream carry commits are preserved after a rebase
argument-hint: <repo> <branch> [--pr <pr-number>]
---

## Name
oadp-rebase:verify-commits

## Synopsis
```
/oadp-rebase:verify-commits <repo> <branch> [--pr <pr-number>]
```

## Description
The `oadp-rebase:verify-commits` command verifies that ALL downstream `UPSTREAM: <carry>` commits from the destination branch are preserved in a rebase PR. This is the single most critical check in the entire OADP rebase process.

### Why This Matters
During rebasing, downstream patches can be silently dropped. This has happened historically and causes serious regressions:
- Security patches lost
- OpenShift-specific fixes removed
- Konflux CI integration broken
- Custom Dockerfiles deleted

### Commit Prefix Conventions
- `UPSTREAM: <carry>`: Downstream patch that MUST be preserved across every rebase. These are manually authored and contain important downstream modifications.
- `UPSTREAM: <drop>`: Auto-generated commit from hook scripts (go-replace, go-mod-tidy, normalize-dockerfiles). These are regenerated each rebase and it is SAFE to lose them.
- No prefix: Upstream commits from the source tag/branch.

### What Gets Checked
1. All `UPSTREAM: <carry>` commits from dest branch exist in the rebase PR
2. Downstream-only files are not deleted (Dockerfile.ubi, konflux.Dockerfile, release-announcement, .tekton/, etc.)
3. The rebase PR does not contain unexpected file deletions
4. Hook-generated `UPSTREAM: <drop>` commits are present (validates hooks ran successfully)

## Implementation

### Phase 1: Collect Expected Commits
1. Clone or access the destination downstream repository
2. List all `UPSTREAM: <carry>` commits on the destination branch:
   ```bash
   gh api repos/{org}/{repo}/commits?sha={branch}&per_page=100 | \
     jq -r '.[].commit.message' | grep "UPSTREAM: <carry>"
   ```
   Or locally:
   ```bash
   git log --oneline --grep="UPSTREAM: <carry>" origin/{branch}
   ```
3. Store this as the "expected" list

### Phase 2: Collect Actual Commits
1. If `--pr` is specified, fetch the PR branch commits:
   ```bash
   gh pr view {pr-number} --repo {org}/{repo} --json commits
   ```
2. Otherwise, check the rebase branch directly:
   ```bash
   git log --oneline --grep="UPSTREAM: <carry>" rebase-bot-{branch}
   ```
3. Store this as the "actual" list

### Phase 3: Compare
1. For each expected carry commit, verify it exists in the actual list (match by commit message subject, not hash - hashes change during cherry-pick)
2. Report any missing commits with their full commit messages
3. Report any extra carry commits (these might be new additions - flag but don't fail)

### Phase 4: Check Downstream Files
1. Verify these downstream-only files exist in the rebase branch (if they existed in dest):
   - `Dockerfile.ubi` or `Dockerfile`
   - `konflux.Dockerfile`
   - `.tekton/` directory
   - `release-announcement` (if applicable)
   - Any other files that only exist downstream

### Phase 5: Report
1. Generate a summary:
   - Total carry commits expected vs found
   - Any missing commits (CRITICAL - blocks merge)
   - Any missing downstream files (CRITICAL - blocks merge)
   - Hook-generated commits present (informational)
2. If all checks pass, output a clear "VERIFIED" status
3. If any checks fail, output detailed failure information

## Return Value
- **PASS**: All carry commits preserved, all downstream files present
- **FAIL**: Lists missing commits and/or files with remediation steps

## Examples

1. **Verify velero rebase PR**:
   ```
   /oadp-rebase:verify-commits velero oadp-1.6 --pr 503
   ```

2. **Verify kubevirt-velero-plugin rebase**:
   ```
   /oadp-rebase:verify-commits kubevirt-velero-plugin oadp-1.6 --pr 60
   ```

3. **Verify without PR (check rebase branch directly)**:
   ```
   /oadp-rebase:verify-commits velero-plugin-for-aws oadp-1.6
   ```

## Arguments
- `$1`: Repository name (e.g., `velero`, `velero-plugin-for-aws`, `kubevirt-velero-plugin`)
- `$2`: OADP branch (e.g., `oadp-1.6`, `oadp-dev`)
- `--pr <number>`: Optional PR number to check against

## See Also
- `/oadp-rebase:rebase` - Run the rebase process
- `/oadp-rebase:manual-rebase` - Handle repos needing manual intervention
