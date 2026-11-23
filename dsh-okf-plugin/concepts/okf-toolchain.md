---
type: Concept
title: The OKF Toolchain (okf.py) and the Hub Template
description: The stdlib-only Python toolchain the hub carries — init, lint, index, links, log, stats, drift — and the template files Set up the hub writes.
status: draft
lifecycle: draft
confidence: high
created: 2026-11-20
updated: 2026-11-21
tags: []
sources:
  - resource: /sources/2026-11-20-okf-py-toolchain.md
    id: okf-py
    title: okf.py key regions (excerpt)
  - resource: /sources/2026-11-20-hub-agents-template.md
    id: hub-agents
    title: templates/hub/AGENTS.md capture
  - resource: /sources/2026-11-20-readme.md
    id: readme
    title: README at 95b0615
  - resource: https://github.com/yishao1224-design/dsh-okf-plugin/blob/95b0615aefbbd4d1e16040efdd37d1e1884e5ed0/templates/hub/scripts/okf.py
    id: okf-src
    title: templates/hub/scripts/okf.py
  - resource: https://github.com/yishao1224-design/dsh-okf-plugin/blob/ec5a774e593070c87dcca36c96a5012481f280c0/templates/hub/README.md
    id: hub-readme
    title: templates/hub/README.md
  - resource: https://github.com/yishao1224-design/dsh-okf-plugin/blob/ec5a774e593070c87dcca36c96a5012481f280c0/src/host/team.ts
    id: team
    title: src/host/team.ts
generated:
  by: dsh/agent
  at: 2026-11-21
---

# The OKF Toolchain (okf.py) and the Hub Template

## Why it lives in the plugin at all

The plugin does not implement bundle conventions in TypeScript; the hub repository carries its own `scripts/okf.py`, and the plugin runs **the repository's copy** so each hub can evolve its rules independently. `team.okfScript` defaults to `scripts/okf.py` and `team.python` to `python` on Windows / `python3` elsewhere [readme]. The same file ships as `templates/hub/scripts/okf.py` (1006 lines, stdlib only, PyYAML used when installed) so a new hub starts with it [okf-py].

## Subcommands

| Command | What it does |
|---|---|
| `init` | Create `concepts/ entities/ comparisons/ queries/ archive/ sources/`, seed `SCHEMA.md` and `log.md` when missing, sync `skills/` into `.claude/skills/` and `.github/prompts/`, regenerate `index.md`, run lint |
| `lint` | Conformance and lifecycle audit; exit 1 on errors |
| `lint --inbound PATH` | List the files linking to a bundle path (for safe deletes) |
| `index` | Regenerate `index.md` from the tree and each page's frontmatter |
| `links` | Expand `[[slug]]` / `[[slug|text]]` authoring shorthand into bundle-relative markdown links, in place |
| `log "MESSAGE"` | Prepend an entry under today's date |
| `stats` | Counts by type and lifecycle, plus the review queue and stale pinned sources |
| `drift` | Recheck pinned `sources[].blob` values against the current git blob |
| `drift --hash FILE` | Print the blob sha to pin for a file |

Every subcommand accepts `--bundle NAME`; without it the toolchain auto-detects a top-level folder holding a `SCHEMA.md`, falls back to `okf/`, and reports rather than guesses when several exist [okf-py].

## Frontmatter it understands

`split_frontmatter` splits a leading `---` block; `parse_frontmatter` uses `yaml.safe_load` when PyYAML is present, else a hand-written subset parser: scalars, inline `[a, b]` and `{k: v}`, block lists of scalars or mappings, and one level of nested mapping — enough for the shapes the schema template describes [okf-py]. `_scalar` strips a trailing ` #` comment, which is why the seeded templates can annotate values inline [okf-py].

## The lint rules

Lint is where the bundle's conventions are enforced [okf-py]:

- **Type.** A non-empty `type` is required (an error). A type missing from SCHEMA.md's *Type registry* table is a warning; a known type in the wrong top-level directory is an **error** (`archive/` is exempt). The registry is parsed from the `# Type registry` section: `` | `Type` | `dir` | … ``.
- **Status and lifecycle.** `status` must be one of OKF v0.2's `draft | stable | deprecated`, `lifecycle` one of `draft | active | needs_review | deprecated | archived`, and the two must agree through `LIFECYCLE_STATUS` (active and needs_review → stable; archived → deprecated). Disagreement is an error.
- **Required fields.** `title`, `description`, `status`, `lifecycle`, `updated` missing → warning. A legacy `review_after` is flagged in favour of `stale_after`.
- **Lifecycle consequences.** `active` without `stale_after` → warning; `active` past `stale_after` → warning; `deprecated` without `superseded_by` or a `## Deprecation` section → error; `archived` outside `archive/` → warning.
- **Sources.** Each `sources[]` entry must be a mapping with a `resource` (error otherwise), ids must be unique, and a legacy scalar entry is flagged. `generated` and `verified`, when present, must be mappings with `by` and `at`.
- **Citations.** A `# Citations` section is flagged: v0.2 replaced it with `sources`.
- **Tags.** A tag not in SCHEMA.md's `# Tag taxonomy` list is a warning (the seeded taxonomy has `unverified` and `contested`).
- **Links.** Broken bundle links to `.md` files are info; pages with `active`/`needs_review` must have at least two outbound bundle links; an unexpanded `[[wikilink]]` is a warning; a page with no inbound links and not in `index.md` is an orphan warning.
- **Reserved files.** An `index.md` with frontmatter below the bundle root is an error (OKF §6/§11).
- **Drift.** A pinned source whose file changed or vanished is a warning (skipped when git is unavailable).

`lint` prints `ERROR:`, `WARN:` and `INFO:` lines and a summary, and exits 1 when there are errors [okf-py].

## index, links, log

`index` groups pages by their top-level folder and writes `--- okf_version: "0.2" ---`, `# Bundle Index`, root entries first, then one `# Section` per folder in `SECTION_ORDER` (`concepts`, `entities`, `comparisons`, `queries`, `sources`, `archive`) followed by any others alphabetically. Each line is `* [title](path) - description`, with a ` *(lifecycle)*` badge for `needs_review`, `deprecated` and `archived` — which is exactly the shape of this bundle's `index.md` [okf-py].

`links` resolves `[[target]]` by explicit path (contains `/` or ends `.md`) or by lower-cased filename stem / title, rewrites the resolved ones to `[label](bundle/path)`, leaves code spans and fenced code untouched (so examples in `SCHEMA.md` survive), and reports ambiguous or missing targets instead of guessing [okf-py].

`log` inserts `* MESSAGE` under a `## YYYY-MM-DD` heading, creating the heading after the H1 when the date is new, and warns once the log passes 500 entries [okf-py].

## What the plugin runs

Open pull request runs, in the workspace, with `python <repo>/scripts/okf.py` [team]:

```sh
python scripts/okf.py links --bundle <folder>
python scripts/okf.py index --bundle <folder>
python scripts/okf.py lint  --bundle <folder>      # failure aborts, nothing committed
python scripts/okf.py log "<Verb>: [Title](/<rel>) — <PR title or wiki name>" --bundle <folder>   # once per changed page
```

New project wiki runs `init` then `index` for the new folder and then removes what `init` touched outside it, because `init` also re-syncs skill copies and a wiki PR may carry only its folder [team]. If `scripts/okf.py` is absent, the PR still gets pushed but is marked as needing a person (`lint did not run`) [team].

## The hub skeleton

`Set up the hub` copies `templates/hub/` into a fresh repository on the base branch and adds the checker and the three example workflows from the package [readme][team]:

| Path in the hub | Source |
|---|---|
| `AGENTS.md` | `templates/hub/AGENTS.md` (rules + empty registry table) |
| `README.md` | `templates/hub/README.md` |
| `scripts/okf.py` | `templates/hub/scripts/okf.py` |
| `published/Projects/` | `templates/hub/published/Projects/.gitkeep` |
| `.github/wiki/check-pr.js` | `lib/check-pr.js` |
| `.github/wiki/workflows/wiki-auto-merge.yml` | `examples/wiki-auto-merge.yml` |
| `.github/wiki/workflows/wiki-auto-complete-ado.yml` | `examples/wiki-auto-complete-ado.yml` |
| `.github/wiki/workflows/wiki-publish-to-ado.yml` | `examples/wiki-publish-to-ado.yml` |

The workflows deliberately sit in `.github/wiki/workflows/` rather than `.github/workflows/`, so the very first push needs no `workflow` token scope; the template README explains how to move them into place — or, for a hub in Azure Repos, how to add `wiki-auto-complete-ado.yml` as a pipeline — and what to enable ("Allow auto-merge", branch protection, a required Build validation with "Contribute to pull requests" for its Build Service account, `vars.ADO_WIKI_*`, `secrets.ADO_WIKI_PAT`) [hub-readme][readme].

## Related pages

- [Review state and auto-merge](/concepts/review-and-auto-merge.md) — how lint output and `judge()` cooperate.
- [Team Wiki service](/concepts/team-wiki-service.md) — the calls that run the toolchain.
- [Data model](/concepts/data-model.md) — the frontmatter fields lint enforces.
