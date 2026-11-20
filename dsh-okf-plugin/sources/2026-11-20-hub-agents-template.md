---
type: Source
title: "templates/hub/AGENTS.md at 95b0615 (verbatim)"
description: "The hub skeleton's AGENTS.md: rules for agents and the empty project registry table that Set up the hub writes."
status: draft
lifecycle: draft
created: 2026-11-20
updated: 2026-11-20
source_url: https://github.com/yishao1224-design/dsh-okf-plugin/blob/95b0615aefbbd4d1e16040efdd37d1e1884e5ed0/templates/hub/AGENTS.md
tags: []
generated:
  by: dsh/agent
  at: 2026-11-20
---

# Capture: templates/hub/AGENTS.md

Verbatim copy of `templates/hub/AGENTS.md` at commit `95b0615aefbbd4d1e16040efdd37d1e1884e5ed0`. This file is copied into a newly set-up hub by `Set up the hub`.

```markdown
# Team wiki hub

This repository holds the team's knowledge: one OKF bundle per project,
registered in the table below. Agents working in a project's code repository
are pointed here by that repository's own AGENTS.md.

## How to use it

1. Find the project in the registry, then read `<Bundle>/index.md` and the pages it lists.
2. Follow `<Bundle>/SCHEMA.md` when writing pages. Every page an agent writes
   is an unreviewed draft (`status: draft`, `lifecycle: draft`, `generated: { by, at }`).
   Never write `verified` entries or `human:` reviewers; a person adds those.
3. Never edit anything under a bundle's `sources/`; capture a new dated file instead.
4. Run `python scripts/okf.py lint --bundle <Bundle>` before proposing a change.
5. Propose changes as a pull request that touches one project only.
   A pull request that only adds or edits unreviewed pages may merge automatically;
   anything else waits for a person.
6. `published/Projects/` is generated (the Publish button in DSH's Team Wiki page)
   and synced to the Azure DevOps wiki after merge. Do not edit it by hand.

## Project registry

One row per project. Agents read this table to find a project's OKF bundle in this repository.

| Project | Repo | Summary | Owner | Bundle |
|---|---|---|---|---|
```
