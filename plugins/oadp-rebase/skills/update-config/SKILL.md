---
name: Update Config Command
description: Update a rebase config file to target a new upstream version
---

## Name
oadp-rebase:update-config

## Synopsis
```
/oadp-rebase:update-config <repo> <branch> <new-upstream-version>
```

## Description
The `oadp-rebase:update-config` command updates a rebase configuration file in `rebase-configs/` to point to a new upstream tag or branch. This is the first step when performing a rebase to a newer upstream version.

### Config File Location
Config files are in the `rebase-configs/` directory of the oadp-rebase repository, named:
```
{org}_{repo}_{branch}.env.sh
```
Where `org` is `openshift` or `migtools`, repo name uses underscores instead of hyphens, and branch uses hyphens.

### Config File Variables
Each config file defines:
- `SOURCE_UPSTREAM_REPO`: `https://github.com/{org}/{repo}:{tag-or-branch}` - the upstream source
- `DESTINATION_DOWNSTREAM_REPO`: `{org}/{repo}:{branch}` - the downstream target
- `REBASE_REPO`: `oadp-rebasebot/{repo}:rebase-bot-{branch}` - the working branch for PRs
- `HOOK_SCRIPTS`: Post-rebase hook scripts to execute
- `EXTRA_REBASEBOT_ARGS`: Additional rebasebot flags
- `SKIP_REPO`: Set to `"true"` to skip this repo during wave execution

### Common Upstream Version Patterns
- **Velero**: `v1.18.1-rc.1` (velero-io/velero tag)
- **Velero plugins (aws, gcp, azure)**: `v1.14.1-rc.1` (velero-io/velero-plugin-for-* tag)
- **Kopia**: `v0.22.3-velero-patch` (project-velero/kopia tag)
- **kubevirt-velero-plugin**: `v0.9.0` (kubevirt/kubevirt-velero-plugin tag)

Note: Upstream repos moved from `vmware-tanzu` to `velero-io` on GitHub. Config files use `velero-io` URLs. The Go module path in go.mod may still use `github.com/vmware-tanzu/velero` - do not confuse these.

### Downstream-Only Repos
Some repos have `SOURCE == DESTINATION` (source and dest are the same repo/branch). These are downstream-only repositories that don't rebase from upstream - they only run hooks to update dependencies:
- `velero-plugin-for-legacy-aws`
- `oadp-operator`

For these repos, do NOT change the source - it should remain pointing to the downstream branch.

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
All config files are in `$OADP_REBASE_DIR/rebase-configs/`. All subsequent operations should use this directory.

### Phase 2: Locate Config
1. Map the repo name and branch to the config file:
   ```
   velero + oadp-1.6 -> rebase-configs/openshift_velero_oadp-1.6.env.sh
   velero-plugin-for-aws + oadp-1.6 -> rebase-configs/openshift_velero_plugin_for_aws_oadp-1.6.env.sh
   kubevirt-velero-plugin + oadp-1.6 -> rebase-configs/migtools_kubevirt_velero_plugin_oadp-1.6.env.sh
   ```
2. Read the current config file

### Phase 3: Verify Upstream Version
1. Check that the new upstream tag/branch exists:
   ```bash
   git ls-remote --tags https://github.com/{upstream-org}/{repo} {new-version}
   ```
2. If it doesn't exist, warn the user and abort

### Phase 4: Update Config
1. Update the upstream version variable (e.g., `UPSTREAM_VELERO_BRANCH`, `UPSTREAM_PLUGIN_TAG`, `UPSTREAM_KOPIA_TAG_BRANCH_FOR_VELERO`)
2. Update any comments referencing the version
3. Do NOT change `DESTINATION_DOWNSTREAM_REPO`, `REBASE_REPO`, or `HOOK_SCRIPTS` unless explicitly requested

### Phase 5: Validate
1. Source the updated config file to verify it parses correctly
2. Print the resulting configuration for review
3. Run `./run-oadp-rebase.sh --test --branch {branch} {repo}` to validate

## Return Value
- Updated config file path and a summary of changes made

## Examples

1. **Update velero to v1.18.1-rc.1 for oadp-1.6**:
   ```
   /oadp-rebase:update-config velero oadp-1.6 v1.18.1-rc.1
   ```

2. **Update AWS plugin to v1.14.1-rc.1 for oadp-1.6**:
   ```
   /oadp-rebase:update-config velero-plugin-for-aws oadp-1.6 v1.14.1-rc.1
   ```

3. **Update kubevirt-velero-plugin to v0.9.0 for oadp-1.6**:
   ```
   /oadp-rebase:update-config kubevirt-velero-plugin oadp-1.6 v0.9.0
   ```

## Arguments
- `$1`: Repository name (e.g., `velero`, `velero-plugin-for-aws`)
- `$2`: OADP branch (e.g., `oadp-1.6`, `oadp-dev`)
- `$3`: New upstream version tag or branch (e.g., `v1.18.1-rc.1`)

## See Also
- `/oadp-rebase:rebase` - Run the rebase after updating config
- `/oadp-rebase:status` - Check current upstream versions across all repos
