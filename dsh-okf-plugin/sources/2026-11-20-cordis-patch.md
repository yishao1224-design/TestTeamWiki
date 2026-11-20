---
type: Source
title: "cordis.patch.yml at 95b0615 (verbatim)"
description: "The one-row patch that mounts the plugin, plus the commented list of optional config keys."
status: draft
lifecycle: draft
created: 2026-11-20
updated: 2026-11-20
source_url: https://github.com/yishao1224-design/dsh-okf-plugin/blob/95b0615aefbbd4d1e16040efdd37d1e1884e5ed0/cordis.patch.yml
tags: []
generated:
  by: dsh/agent
  at: 2026-11-20
---

# Capture: cordis.patch.yml

Verbatim copy of `cordis.patch.yml` at commit `95b0615aefbbd4d1e16040efdd37d1e1884e5ed0`.

```yaml
# OKF bundle plugin: one row, resolved by this package's own name.
#
# The Host half (lib/index.js) serves the /api/okf/* routes, adds the
# per-session prompt context (bundle folder + catalog). It registers no tools:
# the agent reads the bundle with the Harness's own file tools.
# The client graph reads `dsh.client` from this same manifest and serves
# exports["./client"] (the session-header chip, the dialog, and the Team Wiki
# sidebar page).
#
# Optional config (a patch replaces the whole config block):
#
#     config:
#       libraryDir: 'D:/knowledge/okf'   # default: <DSH home>/okf (DSH_HOME, else ~/.dsh)
#       gitPath: 'C:/Program Files/Git/cmd/git.exe'   # default: git on PATH
#       gitTimeoutMs: 300000
#       indexChars: 8000      # characters of index.md sent each turn; 0 = none
#       promptContext: true   # tell the model which bundle the session uses
#       team:                 # Team Wiki sidebar page (all optional)
#         url: 'https://github.com/acme/team-wiki'   # connect this repo when none is connected
#         root: ""            # where to look for bundles when AGENTS.md has no registry (blank = repo root)
#         python: python      # runs the repo's scripts/okf.py
#         autoMergeLabel: wiki-auto-merge
- insert:
    - id: okf
      name: '@local/dsh-okf'
```
