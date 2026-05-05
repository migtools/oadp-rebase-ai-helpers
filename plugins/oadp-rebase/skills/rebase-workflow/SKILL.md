---
name: OADP Rebase Workflow
description: Complete step-by-step guide to the OADP rebase process from start to finish
---

# OADP Rebase Workflow

This skill provides the complete, detailed knowledge needed to perform OADP rebases. It covers everything from understanding the architecture to executing rebases and handling failures.

## Background: What is OADP?

OADP (OpenShift API for Data Protection) is a Red Hat product that wraps the upstream Velero backup/restore project for use on OpenShift. The OADP team maintains downstream forks of approximately 20 repositories from the Velero ecosystem. Each fork contains:

- **Downstream patches**: OpenShift-specific modifications, security fixes, CI integration
- **Custom Dockerfiles**: Konflux/Tekton CI Dockerfiles (konflux.Dockerfile, Dockerfile.ubi)
- **Dependency replacements**: go.mod replace directives pointing upstream deps to downstream forks
- **CI integration**: .tekton/ directories for Konflux pipelines

## Repository Inventory

### Upstream Organizations
- `velero-io`: Velero core and plugins (aws, gcp, azure, csi) - formerly `vmware-tanzu`, see "Upstream Org Migration" section below
- `project-velero`: Kopia fork used by Velero
- `kubevirt`: kubevirt-velero-plugin
- `restic`: Restic backup tool

### Downstream Organizations
- `openshift`: Velero, plugins, oadp-operator, restic, must-gather, openshift-velero-plugin, hypershift-oadp-plugin
- `migtools`: Kopia, kubevirt-velero-plugin, oadp-non-admin, oadp-cli, udistribution, filebrowser, kubevirt-datamover-*, oadp-vmdp, oadp-vm-file-restore

### Working/PR Organization
- `oadp-rebasebot`: Contains rebase branches where PRs are created from

## Wave Ordering

Repos are organized into waves based on dependency ordering. **Always rebase in wave order.**

### Wave 1: Foundation (no velero dependency)
| Repo | Upstream | Downstream |
|------|----------|------------|
| kopia | project-velero/kopia | migtools/kopia |
| restic | restic/restic | openshift/restic |
| filebrowser | (varies) | migtools/filebrowser |
| oadp-vmdp | (varies) | migtools/oadp-vmdp |
| udistribution | (varies) | migtools/udistribution |

### Wave 2: Velero Core
| Repo | Upstream | Downstream |
|------|----------|------------|
| velero | velero-io/velero | openshift/velero |

### Wave 3: Velero Plugins + Operator
| Repo | Upstream | Downstream |
|------|----------|------------|
| velero-plugin-for-aws | velero-io/velero-plugin-for-aws | openshift/velero-plugin-for-aws |
| velero-plugin-for-gcp | velero-io/velero-plugin-for-gcp | openshift/velero-plugin-for-gcp |
| velero-plugin-for-microsoft-azure | velero-io/velero-plugin-for-microsoft-azure | openshift/velero-plugin-for-microsoft-azure |
| velero-plugin-for-legacy-aws | (downstream-only) | openshift/velero-plugin-for-legacy-aws |
| velero-plugin-for-csi | velero-io/velero-plugin-for-csi | openshift/velero-plugin-for-csi |
| kubevirt-velero-plugin | kubevirt/kubevirt-velero-plugin | migtools/kubevirt-velero-plugin |
| oadp-operator | (downstream-only) | openshift/oadp-operator |

### Wave 4: Higher-level Components
| Repo | Upstream | Downstream |
|------|----------|------------|
| oadp-non-admin | (varies) | migtools/oadp-non-admin |
| openshift-velero-plugin | (varies) | openshift/openshift-velero-plugin |
| kubevirt-datamover-controller | (varies) | migtools/kubevirt-datamover-controller |
| oadp-vm-file-restore | (varies) | migtools/oadp-vm-file-restore |

### Wave 5: Tools + Gather
| Repo | Upstream | Downstream |
|------|----------|------------|
| oadp-must-gather | (varies) | openshift/oadp-must-gather |
| oadp-cli | (varies) | migtools/oadp-cli |
| kubevirt-datamover-plugin | (varies) | migtools/kubevirt-datamover-plugin |

### Special Cases (NOT in waves)
| Repo | Notes |
|------|-------|
| hypershift-oadp-plugin | Always manual. Update go.mod replace directive only. |

## Rebase Configuration Files

Located in `rebase-configs/` directory of the oadp-rebase repository.

### Naming Convention
```
{org}_{repo-with-underscores}_{branch}.env.sh
```
Examples:
- `openshift_velero_oadp-1.6.env.sh`
- `openshift_velero_plugin_for_aws_oadp-1.6.env.sh`
- `migtools_kubevirt_velero_plugin_oadp-1.6.env.sh`
- `openshift_oadp-operator_oadp-1.6.env.sh`

### Config Variables
```bash
# Full rebase (upstream -> downstream)
SOURCE_UPSTREAM_REPO="https://github.com/velero-io/velero:v1.18.1-rc.1"
DESTINATION_DOWNSTREAM_REPO="openshift/velero:oadp-1.6"
REBASE_REPO="oadp-rebasebot/velero:rebase-bot-oadp-1.6"

# Hooks-only rebase (downstream -> downstream, same repo)
SOURCE_UPSTREAM_REPO="https://github.com/openshift/oadp-operator:oadp-1.6"
DESTINATION_DOWNSTREAM_REPO="openshift/oadp-operator:oadp-1.6"
REBASE_REPO="oadp-rebasebot/oadp-operator:rebase-bot-oadp-1.6"

# Common settings
EXTRA_REBASEBOT_ARGS="--always-run-hooks"
HOOK_SCRIPTS_LOCATION="git:https://github.com/oadp-rebasebot/oadp-rebase/oadp-dev:rebasebot-hook-scripts"
HOOK_SCRIPTS="--post-rebase-hook \
  ${HOOK_SCRIPTS_LOCATION}/go-replace_velero_oadp-1.6.sh \
  ${HOOK_SCRIPTS_LOCATION}/go-mod-tidy-and-commit.sh \
  ${HOOK_SCRIPTS_LOCATION}/normalize-dockerfiles-and-commit.sh \
  "

# Skip this repo (used for repos not yet onboarded to a branch)
SKIP_REPO="true"
```

## Post-Rebase Hook Scripts

Hook scripts run after rebasebot finishes cherry-picking downstream commits. They are located in `rebasebot-hook-scripts/` directory of the oadp-rebase repository and referenced via `git:` URLs in configs.

### Common Hooks

| Hook Script | Purpose | Generates |
|------------|---------|-----------|
| `go-replace_velero_{branch}.sh` | Adds go.mod replace for the velero module (auto-detects `vmware-tanzu/velero` or `velero-io/velero` from go.mod) | `UPSTREAM: <drop>: update velero go.mod replace` |
| `go-replace_kopia_{branch}.sh` | Adds go.mod replace: `kopia/kopia => migtools/kopia` | `UPSTREAM: <drop>: update kopia go.mod replace` |
| `go-mod-tidy-and-commit.sh` | Runs `go mod tidy` and vendors | `UPSTREAM: <drop>: go mod tidy` |
| `normalize-dockerfiles-and-commit.sh` | Normalizes Dockerfile formatting | `UPSTREAM: <drop>: normalize Dockerfiles` |
| `restic-submodule-and-commit_{branch}.sh` | Updates restic git submodule in velero | `UPSTREAM: <drop>: update restic submodule` |
| `fix-malformed-filenames-and-commit.sh` | Fixes filenames with special characters | `UPSTREAM: <drop>: fix malformed filenames` |
| `oadp-operator-copy-crds-from-velero-and-commit_{branch}.sh` | Copies CRDs from velero repo into oadp-operator | `UPSTREAM: <drop>: copy velero CRDs` |
| `oadp-operator-run-make-bundle-and-commit.sh` | Runs `make bundle` in oadp-operator | `UPSTREAM: <drop>: make bundle` |

### Hook Environment Variables
Hook scripts expect these environment variables:
- `REBASEBOT_SOURCE`: The source repo URL (e.g., `https://github.com/velero-io/velero:v1.18.1-rc.1`)
- `GIT_USERNAME`: Git committer name (default: `oadp-team-rebase-bot`)
- `GIT_EMAIL`: Git committer email (default: `oadp-maintainers@redhat.com`)

## Commit Conventions

### UPSTREAM: <carry>
- Downstream patches that MUST be preserved across every rebase
- Manually authored by developers
- Examples: security fixes, OpenShift integration, CI configuration
- **NEVER drop these** - losing a carry commit is a critical failure

### UPSTREAM: <drop>
- Auto-generated by post-rebase hook scripts
- Regenerated fresh each rebase cycle
- Safe to lose - they will be recreated by hooks
- Examples: go.mod replace directives, go mod tidy results, Dockerfile normalization

### No prefix
- Upstream commits from the source tag/branch
- These come from the upstream project

## The run-oadp-rebase.sh Script

Main entry point for running rebases. Located at the root of the oadp-rebase repository.

### Key Flags
```
./run-oadp-rebase.sh [OPTIONS] <target>

Options:
  -d, --dry-run              Dry-run mode (no push, no PR)
  -l, --local                Use local rebasebot CLI instead of container
  -b, --branch BRANCH        OADP branch (default: oadp-dev)
  -w, --wave                 Execute entire wave
  -t, --test                 Test configuration only
  --working-dir DIR          Working directory for cloned repos
  --local-hooks              Use local hook scripts from ./rebasebot-hook-scripts
  -r, --remote               Load config from remote GitHub (instead of local files)
```

### Working Directory (Important)
Always use `--working-dir` with a unique temp directory per rebase session. This prevents stale git state from previous runs from interfering, and avoids collisions when running multiple rebases. The script automatically creates `{working-dir}/{repo-name}/` subdirectories.

```bash
# Create a unique working directory for this session
WORK_DIR="/tmp/oadp-rebase-$(date +%Y%m%d-%H%M%S)"
```

### Usage Examples
```bash
# Rebase a single repo with its own isolated working directory
WORK_DIR="/tmp/oadp-rebase-velero-$(date +%Y%m%d-%H%M%S)"
./run-oadp-rebase.sh --local --working-dir "$WORK_DIR" --branch oadp-1.6 velero

# Dry-run first
WORK_DIR="/tmp/oadp-rebase-velero-$(date +%Y%m%d-%H%M%S)"
./run-oadp-rebase.sh --local --dry-run --working-dir "$WORK_DIR" --branch oadp-1.6 velero

# Test config loading (no working dir needed)
./run-oadp-rebase.sh --test --branch oadp-1.6 velero
```

## Complete Rebase Procedure (Start to Finish)

### Step 1: Determine Target Versions
- Identify the new upstream versions for each repo
- Common pattern: Velero `v1.18.x` uses plugins `v1.14.x`, Kopia `v0.22.x-velero-patch`
- Check upstream release notes for compatible versions

### Step 2: Update Config Files
- Edit `rebase-configs/` files to point to new upstream versions
- Commit the config changes to the oadp-rebase repository

### Step 3: Execute Wave 1 (repos individually)
Repos within the same wave are independent - rebase each one separately for better control.

```bash
# Wave 1: kopia, restic, filebrowser, oadp-vmdp (check config for which are active)
# For each repo: create isolated working dir, rebase, verify, trigger CI

WORK_DIR="/tmp/oadp-rebase-kopia-$(date +%Y%m%d-%H%M%S)"
./run-oadp-rebase.sh --local --working-dir "$WORK_DIR" --branch oadp-1.6 kopia
# Verify carry commits, then:
gh pr comment {pr-number} --repo migtools/kopia --body "/ok-to-test"

# Repeat for each Wave 1 repo (restic, filebrowser, oadp-vmdp)
# Each gets its own WORK_DIR
```
- For each repo: verify carry commits, comment `/ok-to-test`, wait for CI
- All Wave 1 PRs must be merged before starting Wave 2

### Step 4: Execute Wave 2 (velero)
```bash
WORK_DIR="/tmp/oadp-rebase-velero-$(date +%Y%m%d-%H%M%S)"
./run-oadp-rebase.sh --local --working-dir "$WORK_DIR" --branch oadp-1.6 velero
```
- Verify carry commits preserved
- Add follow-up version bump commit if needed:
  ```bash
  # In the working directory ($WORK_DIR/velero/), update konflux.Dockerfile version
  git commit -m "UPSTREAM: <carry>: update velero version to v1.18.1-rc.1"
  git push
  ```
- Comment `/ok-to-test`, wait for CI to pass, merge

### Step 5: Execute Wave 3 (repos individually)
```bash
# Rebase each Wave 3 repo separately
for repo in velero-plugin-for-aws velero-plugin-for-gcp velero-plugin-for-microsoft-azure \
            velero-plugin-for-legacy-aws kubevirt-velero-plugin oadp-operator; do
  WORK_DIR="/tmp/oadp-rebase-${repo}-$(date +%Y%m%d-%H%M%S)"
  ./run-oadp-rebase.sh --local --working-dir "$WORK_DIR" --branch oadp-1.6 "$repo"
  # Verify carry commits for this repo
  # Comment /ok-to-test on the PR
done
```
- Some repos may need `--conflict-policy warn` temporarily
- kubevirt-velero-plugin may need manual intervention (see manual-intervention skill)
- Verify carry commits for each repo individually

### Step 6: Handle Special Cases
- **hypershift-oadp-plugin**: Manual go.mod update (see update-dependency command)
- Any repos that failed: use manual-rebase command

### Step 7: Execute Waves 4 and 5 (repos individually)
Same pattern - rebase each repo individually with its own working directory:
```bash
# Wave 4 repos
for repo in oadp-non-admin openshift-velero-plugin kubevirt-datamover-controller oadp-vm-file-restore; do
  WORK_DIR="/tmp/oadp-rebase-${repo}-$(date +%Y%m%d-%H%M%S)"
  ./run-oadp-rebase.sh --local --working-dir "$WORK_DIR" --branch oadp-1.6 "$repo"
done

# Wave 5 repos (after all Wave 4 PRs merged)
for repo in oadp-must-gather oadp-cli kubevirt-datamover-plugin; do
  WORK_DIR="/tmp/oadp-rebase-${repo}-$(date +%Y%m%d-%H%M%S)"
  ./run-oadp-rebase.sh --local --working-dir "$WORK_DIR" --branch oadp-1.6 "$repo"
done
```

### Step 8: Final Verification
- Verify all PRs are created and downstream commits preserved
- Review each PR for unexpected changes
- Ensure no files were deleted that shouldn't be

## Triggering CI on Rebase PRs

Rebase PRs are created by the `oadp-rebasebot` bot account. Most OADP repos use Prow or Konflux CI that requires an authorized user (org member) to comment `/ok-to-test` on PRs from external/bot accounts before CI pipelines will run.

After each rebase PR is created:
```bash
gh pr comment {pr-number} --repo {org}/{repo} --body "/ok-to-test"
```

Wait for CI to pass before merging. If CI fails, investigate the failure before proceeding to the next wave - later waves depend on earlier ones being merged.

## GitHub App Authentication

Rebasebot uses two GitHub Apps:
- **oadp-rebasebot-app** (ID: 1810299): Creates PRs and pushes to rebase branches
- **oadp-rebasebot-cloner** (ID: 1810272): Clones private repositories

Private keys must be stored in `~/.rebasebot/secrets/`:
- `oadp-rebasebot-app-key`
- `oadp-rebasebot-cloner-key`

## Troubleshooting

### "Dest already contains source"
This message from rebasebot means the upstream tag hasn't changed since the last rebase. If `--always-run-hooks` is set, hooks will still run. However, if a stale rebase branch exists, rebasebot may use it as the base instead of the current dest, causing downstream files to be dropped. Solution: delete the remote rebase branch and re-run, or use manual intervention.

### "Manual intervention is needed" (strict conflict policy)
The strict conflict policy detected content that would be lost. Usually this is go.mod/go.sum content from `UPSTREAM: <drop>` commits that will be regenerated by hooks. Verify the content is indeed from drop commits, then temporarily use `--conflict-policy warn`.

### Hook script fails with "REBASEBOT_SOURCE not set"
When running hooks manually outside of rebasebot, you must export the `REBASEBOT_SOURCE` environment variable. Example:
```bash
export REBASEBOT_SOURCE="https://github.com/velero-io/velero:v1.18.1-rc.1"
```

### go mod tidy fails
Usually means the replace directives point to a branch/commit that doesn't exist yet. Ensure the dependency repos (e.g., downstream velero) have been rebased first (wave ordering).

### Rebase PR shows too many changes
If the PR diff is unexpectedly large, check:
1. Is the rebase branch based on the correct dest branch? (stale branch problem)
2. Are downstream-only files being deleted? (stale branch problem)
3. Is the upstream version jump very large? (expected for major version bumps)

## Upstream Org Migration: vmware-tanzu -> velero-io

The upstream Velero project moved its GitHub repositories from the `vmware-tanzu` org to the `velero-io` org. This affects two distinct identifiers that must NOT be confused:

### 1. GitHub URL (where the git repo lives) - CHANGED
- Old: `https://github.com/vmware-tanzu/velero`
- New: `https://github.com/velero-io/velero`

GitHub may redirect old URLs temporarily, but this is unreliable for API calls and raw file fetches. All config files (`rebase-configs/*.env.sh`) use the new `velero-io` org for `SOURCE_UPSTREAM_REPO` and `UPSTREAM_PLUGIN_REPO`.

### 2. Go module path (declared in go.mod) - NOT YET CHANGED
Upstream velero's `go.mod` still declares:
```
module github.com/vmware-tanzu/velero
```

This is the identifier Go uses internally. The `replace` directive in downstream go.mod files MUST match the module path used in the `require` directive. If the require says `github.com/vmware-tanzu/velero`, the replace MUST use `github.com/vmware-tanzu/velero` on the left side. Using `github.com/velero-io/velero` when the require still says `vmware-tanzu` will cause Go to silently ignore the replace.

### Transition Timeline
- **Phase 1 (current)**: GitHub org = `velero-io`, Go module path = `github.com/vmware-tanzu/velero`
- **Phase 2 (future)**: GitHub org = `velero-io`, Go module path = `github.com/velero-io/velero`

### How Hook Scripts Handle This
The `go-replace_velero_{branch}.sh` hook scripts use **dynamic detection** - they read the project's own `go.mod` to determine which module path (`vmware-tanzu` or `velero-io`) is actually in use, rather than hardcoding either one. They also clean up stale replace lines from the other module path.

**NEVER hardcode the Go module path for velero in hook scripts.** Always detect it from the project's `go.mod` at runtime. The GitHub org (git URL) and the Go module path are two independent things that may change at different times. Individual plugins may also transition independently.

### What To Update When (for future reference)
- **Config files** (`SOURCE_UPSTREAM_REPO`, `UPSTREAM_PLUGIN_REPO`): Already updated to `velero-io`. Only change if upstream moves again.
- **Hook scripts** (`go-replace_velero_*.sh`): Already use dynamic detection. No change needed when upstream switches Go module path.
- **Manual go.mod edits** (e.g., hypershift-oadp-plugin): Must match whatever module path the project's `go.mod` currently uses in its `require` block. Check before editing.
