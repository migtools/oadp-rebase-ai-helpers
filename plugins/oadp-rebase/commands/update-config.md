---
description: Update a rebase config file to target a new upstream version
argument-hint: <repo> <branch> <new-upstream-version>
---

## Name
oadp-rebase:update-config

## Synopsis
```
/oadp-rebase:update-config <repo> <branch> <new-upstream-version>
```

## Description
This command uses the `update-config` skill as its source of truth.

## Implementation
Follow the full implementation in:
`plugins/oadp-rebase/skills/update-config/SKILL.md`

## See Also
- `/oadp-rebase:rebase`
- `/oadp-rebase:status`
