---
name: Update Dependency Command
description: Update go.mod replace directives for repos that depend on downstream velero
---

## Name
oadp-rebase:update-dependency

## Synopsis
```
/oadp-rebase:update-dependency <repo> <branch> [--velero-commit <commit>]
```

## Description
The `oadp-rebase:update-dependency` command updates go.mod replace directives in repositories that depend on the downstream openshift/velero fork. This is needed for repos like hypershift-oadp-plugin that are not part of the automated rebasebot flow.

### Background
Several OADP repositories use `go.mod` replace directives to redirect the upstream velero module to `github.com/openshift/velero` (the downstream fork). The left-hand side of the replace must match the module path in the project's `require` block - currently `github.com/vmware-tanzu/velero` for most projects, but this may change to `github.com/velero-io/velero` as upstream transitions. Always check the project's `go.mod` to determine the correct module path before editing. After rebasing velero to a new upstream version, these replace directives need to be updated to point to the latest commit on the downstream velero branch.

### Go Pseudo-Version Format
Go modules use pseudo-versions for non-tagged commits:
```
v0.10.2-0.{YYYYMMDDHHMMSS}-{12-char-commit-hash}
```
Example: `v0.10.2-0.20260504131308-8373027be9ec`

The `go mod edit -replace` command with a commit hash will automatically generate the correct pseudo-version after `go mod tidy`.

## Implementation

### Phase 1: Find Latest Velero Commit
1. If `--velero-commit` not provided, fetch the latest commit on the downstream velero branch:
   ```bash
   gh api repos/openshift/velero/commits?sha={branch}&per_page=1 | jq -r '.[0].sha'
   ```

### Phase 2: Update go.mod
1. Clone the target repo (or work in existing checkout)
2. Update the replace directive (use the module path from the project's `require` block - check `go.mod` first):
   ```bash
   # Check which module path the project uses:
   grep "velero" go.mod | head -5
   # Then use the matching path (vmware-tanzu or velero-io):
   go mod edit -replace github.com/vmware-tanzu/velero=github.com/openshift/velero@{commit}
   ```
3. Run `go mod tidy` to resolve the pseudo-version
4. Run `go mod vendor` (if vendor directory exists)
5. Run `go vet ./...` to verify compilation

### Phase 3: Create PR
1. Create a branch, commit, push, and create PR
2. Commit message: `UPSTREAM: <carry>: update velero dependency to {version}`

## Return Value
- PR URL for the dependency update

## Examples

1. **Update hypershift-oadp-plugin velero dependency**:
   ```
   /oadp-rebase:update-dependency hypershift-oadp-plugin oadp-1.6
   ```

## Arguments
- `$1`: Repository name (e.g., `hypershift-oadp-plugin`)
- `$2`: OADP branch (e.g., `oadp-1.6`)
- `--velero-commit <hash>`: Optional specific commit hash to use

## See Also
- `/oadp-rebase:rebase` - Main rebase command (handles most repos automatically)
- `/oadp-rebase:status` - Check current dependency versions
