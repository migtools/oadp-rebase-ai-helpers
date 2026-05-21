---
description: Handle repositories that need manual rebase intervention
argument-hint: <repo> <branch> [--reason <reason>]
---

## Name
oadp-rebase:manual-rebase

## Synopsis
```
/oadp-rebase:manual-rebase <repo> <branch> [--reason <reason>]
```

## Description
This command uses the `manual-rebase` skill as its source of truth.

## Implementation
Follow the full implementation in:
`plugins/oadp-rebase/skills/manual-rebase/SKILL.md`

## See Also
- `/oadp-rebase:rebase`
- `/oadp-rebase:verify-commits`
