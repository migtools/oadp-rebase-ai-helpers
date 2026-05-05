---
name: Verify Downstream Commits
description: Detailed guide for verifying that downstream carry commits are preserved after rebase
---

# Verify Downstream Commits

This skill provides detailed instructions for the most critical safety check in the OADP rebase process: verifying that all downstream `UPSTREAM: <carry>` commits are preserved.

## Why This is Critical

In past rebases, downstream patches have been silently dropped, leading to:
- Security patches disappearing from production builds
- OpenShift-specific integrations breaking
- Konflux CI Dockerfiles being deleted
- Release-specific files vanishing

The root causes of dropped commits include:
1. **Stale rebase branch**: Rebasebot uses an old rebase branch as base instead of current dest
2. **Cherry-pick failures**: A carry commit silently fails to apply and is skipped
3. **Hook script errors**: A hook script removes files or commits that shouldn't be removed

## Verification Procedure

### Step 1: Inventory Expected Commits

Before or immediately after a rebase, collect all carry commits from the destination branch:

```bash
# Using gh CLI
gh api repos/{org}/{repo}/commits?sha={branch}&per_page=100 \
  --jq '.[].commit.message' | grep "^UPSTREAM: <carry>"

# Using git (if you have the repo cloned)
git fetch origin {branch}
git log --oneline --grep="UPSTREAM: <carry>" origin/{branch}
```

Save this list - you will compare against it.

### Step 2: Collect Actual Commits on Rebase Branch

```bash
# From the PR
gh pr view {pr-number} --repo {org}/{repo} --json commits \
  --jq '.commits[].messageHeadline' | grep "UPSTREAM: <carry>"

# From the rebase branch directly
git fetch oadp-rebasebot rebase-bot-{branch}
git log --oneline --grep="UPSTREAM: <carry>" FETCH_HEAD
```

### Step 3: Compare Lists

For each expected carry commit, verify it exists in the actual list:
- Match by **commit message subject line**, NOT by commit hash (hashes change during cherry-pick)
- A missing commit is a **CRITICAL failure** - do not proceed with the merge

### Step 4: Verify Downstream-Only Files

Check that these files exist in the rebase branch (if they existed in dest):

```bash
# Common downstream-only files
git ls-tree --name-only -r FETCH_HEAD | grep -E "(Dockerfile\.ubi|konflux\.Dockerfile|release-announcement|\.tekton/)"
```

Files to check:
- `Dockerfile.ubi` or `Dockerfile` (downstream build)
- `konflux.Dockerfile` (Konflux CI build)
- `release-announcement` (release notes template)
- `.tekton/` directory (Konflux pipeline definitions)
- Any repo-specific downstream files

### Step 5: Verify Hook-Generated Commits

Check that `UPSTREAM: <drop>` commits exist (proves hooks ran successfully):

```bash
git log --oneline --grep="UPSTREAM: <drop>" FETCH_HEAD
```

Expected drop commits vary by repo but typically include:
- `UPSTREAM: <drop>: go mod tidy` (or similar)
- `UPSTREAM: <drop>: normalize Dockerfiles`
- `UPSTREAM: <drop>: update velero go.mod replace` (for repos with velero dependency)

## Known Problem: Stale Rebase Branch

### Symptoms
- Many downstream-only files deleted in the PR
- README.md regressing to upstream version
- PR shows an unexpectedly large diff
- Missing Dockerfile.ubi, konflux.Dockerfile, .tekton/ directory

### Root Cause
When rebasebot detects "dest already contains source" (upstream tag unchanged), it may use an existing stale rebase branch as the starting point for hooks instead of the current dest branch. If the stale branch predates the addition of downstream files, those files won't be present.

### Detection
```bash
# Check the PR diff for deleted downstream files
gh pr diff {pr-number} --repo {org}/{repo} | grep "^-" | head -50

# Check if downstream files exist in the PR branch
gh api repos/{org}/{repo}/git/trees/{pr-branch-sha}?recursive=1 \
  --jq '.tree[].path' | grep -E "(konflux|Dockerfile|tekton)"
```

### Fix
See the manual-intervention skill for detailed steps to fix this.

## Verification Checklist

Use this checklist for every rebase PR:

- [ ] All `UPSTREAM: <carry>` commits from dest branch are present in rebase PR
- [ ] No downstream-only files are deleted (Dockerfile.ubi, konflux.Dockerfile, .tekton/, etc.)
- [ ] Hook-generated `UPSTREAM: <drop>` commits are present
- [ ] PR diff does not show unexpected file deletions
- [ ] For velero: konflux.Dockerfile has correct version string
- [ ] For oadp-operator: CRDs are updated (from velero CRD copy hook)
- [ ] For oadp-operator: bundle is regenerated (from make bundle hook)
