---
name: Hook Scripts Reference
description: Complete reference for all post-rebase hook scripts used in OADP rebases
---

# Hook Scripts Reference

This skill documents every post-rebase hook script, what it does, when it's used, and how to debug failures.

## Hook Script Architecture

Hook scripts are shell scripts stored in the `rebasebot-hook-scripts/` directory of the oadp-rebase repository. They are referenced in config files via `git:` URLs:

```
git:https://github.com/oadp-rebasebot/oadp-rebase/oadp-dev:rebasebot-hook-scripts/{script-name}.sh
```

When rebasebot runs, it:
1. Clones the hook scripts from the `git:` URL
2. Executes each hook script in the order specified in the config
3. Each hook script makes its own git commit (with `UPSTREAM: <drop>` prefix)

### Environment Available to Hooks
- `REBASEBOT_SOURCE`: The source repo URL (e.g., `https://github.com/velero-io/velero:v1.18.1-rc.1`)
- `GIT_USERNAME`: Git committer name
- `GIT_EMAIL`: Git committer email
- Working directory: The root of the rebased repository

## Individual Hook Scripts

### go-replace_velero_{branch}.sh
**Purpose**: Adds a go.mod replace directive pointing the upstream velero module to `github.com/openshift/velero` at the downstream branch.

**Used by**: All repos that import velero (plugins, oadp-operator, non-admin, etc.)

**What it does**:
1. **Dynamically detects** which velero module path the project uses by scanning `go.mod` for either `github.com/vmware-tanzu/velero` or `github.com/velero-io/velero` in `require` blocks
2. Runs `go mod edit -replace {detected-module}=github.com/openshift/velero@{branch}`
3. Cleans up any stale replace line from the other module path (e.g., removes a `vmware-tanzu` replace if the project now uses `velero-io`)
4. Commits the change as `UPSTREAM: <drop>: ...`

**Branch-specific**: Each OADP branch has its own script (e.g., `go-replace_velero_oadp-1.6.sh`)

**Important**: The script NEVER hardcodes the module path. The upstream Velero project moved from `vmware-tanzu` to `velero-io` on GitHub, but the Go module path (`module` line in go.mod) may still be `github.com/vmware-tanzu/velero`. These are two independent identifiers. The replace directive must match the `require` path, not the GitHub URL.

**Debugging**: If this fails, the downstream velero branch may not exist yet (wave ordering issue).

### go-replace_kopia_{branch}.sh
**Purpose**: Adds a go.mod replace directive pointing `github.com/kopia/kopia` to `github.com/migtools/kopia` at the downstream branch.

**Used by**: Velero (which imports kopia for backup operations)

**What it does**:
1. Runs `go mod edit -replace github.com/kopia/kopia=github.com/migtools/kopia@{branch}`
2. Commits the change

**Debugging**: If this fails, the downstream kopia branch may not exist yet (must rebase kopia in Wave 1 first).

### go-mod-tidy-and-commit.sh
**Purpose**: Runs `go mod tidy` and optionally `go mod vendor` to resolve dependencies after replace directives are applied.

**Used by**: Almost all repos

**What it does**:
1. Runs `go mod tidy`
2. If a `vendor/` directory exists, runs `go mod vendor`
3. Commits all changes to go.mod, go.sum, and vendor/ as `UPSTREAM: <drop>: ...`

**Debugging**:
- If `go mod tidy` fails, check that all replace targets exist and are accessible
- Network issues can cause failures - retry
- Version incompatibilities may need manual resolution

### normalize-dockerfiles-and-commit.sh
**Purpose**: Normalizes Dockerfile formatting to ensure consistency.

**Used by**: Almost all repos

**What it does**:
1. Finds all Dockerfiles in the repo
2. Normalizes formatting (trailing newlines, etc.)
3. Commits changes as `UPSTREAM: <drop>: ...`

### restic-submodule-and-commit_{branch}.sh
**Purpose**: Updates the restic git submodule reference in the velero repo.

**Used by**: Velero only

**What it does**:
1. Updates the restic submodule to point to the downstream restic branch
2. Commits the submodule update

**Prerequisite**: Restic must be rebased first (Wave 1).

### fix-malformed-filenames-and-commit.sh
**Purpose**: Fixes filenames with special characters that can cause issues on some platforms.

**Used by**: Velero (some upstream test fixtures have special characters in filenames)

**What it does**:
1. Scans for files with problematic characters
2. Renames them to safe alternatives
3. Commits the changes

### oadp-operator-copy-crds-from-velero-and-commit_{branch}.sh
**Purpose**: Copies CRD YAML files from the downstream velero repo into the oadp-operator repo.

**Used by**: oadp-operator only

**What it does**:
1. Clones or fetches the downstream velero repo at the specified branch
2. Copies CRD files from velero's `config/crd/` to oadp-operator's expected location
3. Commits the updated CRDs

**Prerequisite**: Velero must be rebased and the velero PR must be merged first (Wave 2 before Wave 3).

### oadp-operator-run-make-bundle-and-commit.sh
**Purpose**: Runs `make bundle` in the oadp-operator repo to regenerate the OLM bundle.

**Used by**: oadp-operator only

**What it does**:
1. Runs `make bundle` which regenerates CSV, CRDs, and other OLM artifacts
2. Commits all generated changes

**Debugging**: Requires operator-sdk and other build tools. May fail if CRDs are stale (run CRD copy hook first).

## Hook Execution Order

The order of hooks matters. The typical order is:

1. `fix-malformed-filenames-and-commit.sh` (if needed - velero only)
2. `go-replace_kopia_{branch}.sh` (if needed - velero only)
3. `go-replace_velero_{branch}.sh` (if the repo imports velero)
4. `go-mod-tidy-and-commit.sh` (always after replace hooks)
5. `restic-submodule-and-commit_{branch}.sh` (velero only, after tidy)
6. `oadp-operator-copy-crds-from-velero-and-commit_{branch}.sh` (oadp-operator only)
7. `oadp-operator-run-make-bundle-and-commit.sh` (oadp-operator only, after CRD copy)
8. `normalize-dockerfiles-and-commit.sh` (always last)

## Running Hooks Manually

When you need to run hooks outside of rebasebot (e.g., for manual intervention):

```bash
# Set required environment variables
export REBASEBOT_SOURCE="https://github.com/{upstream-org}/{repo}:{tag}"
export GIT_USERNAME="oadp-team-rebase-bot"
export GIT_EMAIL="oadp-maintainers@redhat.com"
git config user.name "$GIT_USERNAME"
git config user.email "$GIT_EMAIL"

# Get hook scripts
# Option 1: Clone from remote
git clone -b oadp-dev https://github.com/oadp-rebasebot/oadp-rebase.git /tmp/hooks
HOOKS=/tmp/hooks/rebasebot-hook-scripts

# Option 2: Use local copy from oadp-rebase repo
HOOKS=/path/to/oadp-rebase/rebasebot-hook-scripts

# Run hooks in order
bash $HOOKS/{hook1}.sh
bash $HOOKS/{hook2}.sh
# etc.
```

## Which Repos Use Which Hooks

| Repository | go-replace velero | go-replace kopia | go-mod-tidy | restic-submodule | fix-filenames | CRD copy | make bundle | normalize-dockerfiles |
|-----------|:-:|:-:|:-:|:-:|:-:|:-:|:-:|:-:|
| kopia | | | Y | | | | | Y |
| velero | Y | Y | Y | Y | Y | | | Y |
| velero-plugin-for-aws | Y | | Y | | | | | Y |
| velero-plugin-for-gcp | Y | | Y | | | | | Y |
| velero-plugin-for-azure | Y | | Y | | | | | Y |
| velero-plugin-for-legacy-aws | Y | | Y | | | | | Y |
| kubevirt-velero-plugin | Y | | Y | | | | | Y |
| oadp-operator | Y | | Y | | | Y | Y | Y |
| oadp-non-admin | Y | | Y | | | | | Y |
| openshift-velero-plugin | Y | | Y | | | | | Y |
