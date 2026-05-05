# Available Plugins

This document lists all available Claude Code plugins and their commands in the oadp-rebase-ai-helpers repository.

- [OADP Rebase](#oadp-rebase-plugin)

### OADP Rebase Plugin

OADP rebase workflows for rebasing upstream Velero ecosystem repos into OpenShift downstream forks using rebasebot

**Commands:**
- **`/oadp-rebase:rebase` `<repo-branch|wave-number> [--branch <oadp-branch>] [--dry-run]`** - Run a full OADP rebase for a single repository or an entire wave
- **`/oadp-rebase:verify-commits` `<repo> <branch> [--pr <pr-number>]`** - Verify that all downstream carry commits are preserved after a rebase
- **`/oadp-rebase:update-config` `<repo> <branch> <new-upstream-version>`** - Update a rebase config file to target a new upstream version
- **`/oadp-rebase:manual-rebase` `<repo> <branch> [--reason <reason>]`** - Handle repositories that need manual rebase intervention
- **`/oadp-rebase:status` `[--branch <oadp-branch>] [--wave <wave-number>]`** - Check current rebase status across all OADP repositories for a branch
- **`/oadp-rebase:update-dependency` `<repo> <branch> [--velero-commit <commit>]`** - Update go.mod replace directives for repos that depend on downstream velero
- **`/oadp-rebase:update-wiki` `[--branch <oadp-branch>]`** - Generate rebase status markdown and update the oadp-operator wiki page

See [plugins/oadp-rebase/README.md](plugins/oadp-rebase/README.md) for detailed documentation.
