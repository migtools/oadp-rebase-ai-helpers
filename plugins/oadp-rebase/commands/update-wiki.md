---
description: Generate rebase status markdown and update the oadp-operator wiki page
argument-hint: [--branch <oadp-branch>]
---

## Name
oadp-rebase:update-wiki

## Synopsis
```
/oadp-rebase:update-wiki [--branch <oadp-branch>]
```

## Description
This command uses the `update-wiki` skill as its source of truth.

## Implementation
Follow the full implementation in:
`plugins/oadp-rebase/skills/update-wiki/SKILL.md`

## See Also
- `/oadp-rebase:status`
- `/oadp-rebase:rebase`
