# OADP Rebase AI Helpers

Claude Code plugins for automating OADP (OpenShift API for Data Protection) rebase workflows — rebasing downstream OpenShift forks of the Velero ecosystem to newer upstream versions.

## Overview

This project provides [Claude Code](https://docs.anthropic.com/en/docs/claude-code) plugins for automating the OADP rebase process. It encapsulates all knowledge needed to safely rebase ~20 repositories across 5 dependency waves, including automated conflict resolution, downstream commit verification, and manual intervention procedures.

### What is OADP?

OADP is a Red Hat product that wraps the upstream [Velero](https://velero.io/) project for OpenShift. The OADP team maintains downstream forks of repositories across the Velero ecosystem, each containing OpenShift-specific patches, CI integration, and dependency replacements.

### What is a Rebase?

A rebase updates a downstream fork to a newer upstream version while preserving downstream patches. The [rebasebot](https://github.com/oadp-rebasebot) tool automates this by cherry-picking downstream commits (marked with `UPSTREAM: <carry>`) onto the new upstream base, then running post-rebase hook scripts.

## Installation

### Prerequisites

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) CLI installed and configured
- [GitHub CLI](https://cli.github.com/) (`gh`) authenticated with access to the OADP repositories
- Docker or Podman (for container-mode rebases via rebasebot)
- Go toolchain (for building the `rebase-status` tool used by `/oadp-rebase:update-wiki`)

### Required Access (for rebase operations)

- GitHub App private keys in `~/.rebasebot/secrets/`:
  - `oadp-rebasebot-app-key` (App ID: 1810299)
  - `oadp-rebasebot-cloner-key` (App ID: 1810272)
- Write access to `oadp-rebasebot` organization repositories

### Install the Plugin

From within a Claude Code session, first add the marketplace, then install the plugin:

```
/plugin marketplace add migtools/oadp-rebase-ai-helpers
/plugin install oadp-rebase@oadp-rebase-ai-helpers
/reload-plugins
```

Verify the installation by checking available commands:

```
/skills
```

You should see the `oadp-rebase:*` commands listed.

### Uninstall

```
/plugin uninstall oadp-rebase
/reload-plugins
```

## Usage

All commands are invoked as Claude Code slash commands within a conversation.

### Check rebase status

```
/oadp-rebase:status --branch oadp-1.6
```

### Update configs to target a new upstream version

```
/oadp-rebase:update-config velero oadp-1.6 v1.18.1-rc.1
/oadp-rebase:update-config velero-plugin-for-aws oadp-1.6 v1.14.1-rc.1
```

### Rebase repositories (in wave order)

```
/oadp-rebase:rebase velero --branch oadp-1.6
/oadp-rebase:rebase velero-plugin-for-aws --branch oadp-1.6
```

### Verify downstream commits after rebase

```
/oadp-rebase:verify-commits velero oadp-1.6 --pr 503
```

### Handle repos that need manual intervention

```
/oadp-rebase:manual-rebase kubevirt-velero-plugin oadp-1.6
```

### Update velero dependency in downstream repos

```
/oadp-rebase:update-dependency hypershift-oadp-plugin oadp-1.6
```

### Generate and push rebase status to the wiki

```
/oadp-rebase:update-wiki --branch oadp-1.6
```

## Quick Start: Full Rebase Cycle

### 1. Update configs

```
/oadp-rebase:update-config velero oadp-1.6 v1.18.1-rc.1
/oadp-rebase:update-config velero-plugin-for-aws oadp-1.6 v1.14.1-rc.1
# ... repeat for all plugins
```

### 2. Rebase in wave order

```
/oadp-rebase:rebase 1 --branch oadp-1.6        # Wave 1: kopia, restic
# Wait for PRs to merge
/oadp-rebase:rebase velero --branch oadp-1.6    # Wave 2: velero
# Wait for PR to merge
/oadp-rebase:rebase 3 --branch oadp-1.6        # Wave 3: plugins, operator
# Handle any failures with /oadp-rebase:manual-rebase
/oadp-rebase:rebase 4 --branch oadp-1.6        # Wave 4
/oadp-rebase:rebase 5 --branch oadp-1.6        # Wave 5
```

### 3. Verify each wave

```
/oadp-rebase:verify-commits velero oadp-1.6 --pr 503
```

### 4. Handle special cases

```
/oadp-rebase:update-dependency hypershift-oadp-plugin oadp-1.6
```

### 5. Update the wiki

```
/oadp-rebase:update-wiki --branch oadp-1.6
```

## Available Commands

| Command | Description |
|---------|-------------|
| `/oadp-rebase:rebase` | Run a full rebase for a single repository or an entire wave |
| `/oadp-rebase:verify-commits` | Verify that all downstream carry commits are preserved after a rebase |
| `/oadp-rebase:update-config` | Update a rebase config file to target a new upstream version |
| `/oadp-rebase:manual-rebase` | Handle repositories that need manual rebase intervention |
| `/oadp-rebase:status` | Check current rebase status across all OADP repositories |
| `/oadp-rebase:update-dependency` | Update go.mod replace directives for repos that depend on downstream velero |
| `/oadp-rebase:update-wiki` | Generate rebase status markdown and push to the oadp-operator wiki |

See **[PLUGINS.md](PLUGINS.md)** for argument details on each command.

## Wave Ordering

Repositories must be rebased in this order due to dependency chains:

| Wave | Repositories | Depends On |
|------|-------------|------------|
| 1 | kopia, restic, filebrowser, oadp-vmdp | Nothing |
| 2 | velero | Wave 1 (kopia) |
| 3 | velero plugins (aws, gcp, azure, legacy-aws, csi), kubevirt-velero-plugin, oadp-operator | Wave 2 (velero) |
| 4 | oadp-non-admin, openshift-velero-plugin, kubevirt-datamover-controller, oadp-vm-file-restore | Wave 3 |
| 5 | oadp-must-gather, oadp-cli, kubevirt-datamover-plugin | Wave 4 |
| Special | hypershift-oadp-plugin (always manual) | Wave 2 (velero) |

## Critical Safety Rules

1. **ALWAYS verify carry commits** after every rebase using `/oadp-rebase:verify-commits`
2. **NEVER leave `--conflict-policy warn`** in config files permanently
3. **ALWAYS rebase in wave order** — later waves depend on earlier ones being merged
4. **ALWAYS dry-run first** for unfamiliar repos
5. **Check for stale rebase branches** if a PR shows unexpected file deletions

## Additional Documentation

- **[PLUGINS.md](PLUGINS.md)** — Complete list of all available commands with arguments
- **[AGENTS.md](AGENTS.md)** — Guide for AI agents working with this repository
- **[plugins/oadp-rebase/skills/](plugins/oadp-rebase/skills/)** — Detailed implementation guides

## Related Repositories

- [oadp-rebasebot/oadp-rebase](https://github.com/oadp-rebasebot/oadp-rebase) — Rebase configs, hook scripts, and `run-oadp-rebase.sh`
- [openshift/oadp-operator](https://github.com/openshift/oadp-operator) — OADP operator (wiki hosts rebase status pages)
- [oadp-rebasebot/rebasebot](https://github.com/oadp-rebasebot/rebasebot) — The rebasebot tool itself
