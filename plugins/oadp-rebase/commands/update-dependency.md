---
description: Update go.mod replace directives for repos that depend on downstream velero
argument-hint: <repo> <branch> [--velero-commit <commit>]
---

## Name
oadp-rebase:update-dependency

## Synopsis
```
/oadp-rebase:update-dependency <repo> <branch> [--velero-commit <commit>]
```

## Description
This command uses the `update-dependency` skill as its source of truth.

## Implementation
Follow the full implementation in:
`plugins/oadp-rebase/skills/update-dependency/SKILL.md`

## See Also
- `/oadp-rebase:rebase`
- `/oadp-rebase:status`
