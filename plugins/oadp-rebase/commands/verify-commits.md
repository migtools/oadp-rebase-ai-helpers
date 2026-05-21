---
description: Verify that all downstream carry commits are preserved after a rebase
argument-hint: <repo> <branch> [--pr <pr-number>]
---

## Name
oadp-rebase:verify-commits

## Synopsis
```
/oadp-rebase:verify-commits <repo> <branch> [--pr <pr-number>]
```

## Description
This command uses the `verify-commits` skill as its source of truth.

## Implementation
Follow the full implementation in:
`plugins/oadp-rebase/skills/verify-commits/SKILL.md`

## See Also
- `/oadp-rebase:rebase`
- `/oadp-rebase:manual-rebase`
