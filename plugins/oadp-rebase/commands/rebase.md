---
description: Run a full OADP rebase for a single repository or an entire wave
argument-hint: <repo-branch|wave-number> [--branch <oadp-branch>] [--dry-run]
---

## Name
oadp-rebase:rebase

## Synopsis
```
/oadp-rebase:rebase <repo-branch|wave-number> [--branch <oadp-branch>] [--dry-run]
```

## Description
This command uses the `rebase` skill as its source of truth.

## Implementation
Follow the full implementation in:
`plugins/oadp-rebase/skills/rebase/SKILL.md`

## See Also
- `/oadp-rebase:verify-commits`
- `/oadp-rebase:manual-rebase`
- `/oadp-rebase:update-config`
- `/oadp-rebase:status`
