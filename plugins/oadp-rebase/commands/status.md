---
description: Check current rebase status across all OADP repositories for a branch
argument-hint: [--branch <oadp-branch>] [--wave <wave-number>]
---

## Name
oadp-rebase:status

## Synopsis
```
/oadp-rebase:status [--branch <oadp-branch>] [--wave <wave-number>]
```

## Description
This command uses the `status` skill as its source of truth.

## Implementation
Follow the full implementation in:
`plugins/oadp-rebase/skills/status/SKILL.md`

## See Also
- `/oadp-rebase:rebase`
- `/oadp-rebase:update-config`
