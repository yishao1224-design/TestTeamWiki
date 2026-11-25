---
type: Concept
title: The OKF Toolchain (okf.py) and the Hub Template
description: The stdlib-only Python toolchain the hub carries — init, lint, index, links, log, stats, migrate, drift — its TOOLCHAIN_VERSION, and the template files Set up the hub writes.
status: draft
confidence: high
created: 2026-11-20
updated: 2026-11-25
tags: []
sources:
  - resource: /sources/2026-11-25-okf-py-toolchain.md
    id: okf-py
    title: okf.py key regions at da10648
  - resource: https://github.com/yishao1224-design/dsh-okf-plugin/blob/da10648a182b38b410d22f952d8e1d973d8825f0/templates/hub/scripts/okf.py
    id: okf-src
    title: templates/hub/scripts/okf.py
  - resource: https://github.com/yishao1224-design/dsh-okf-plugin/blob/da10648a182b38b410d22f952d8e1d973d8825f0/templates/hub/AGENTS.md
    id: hub-agents-src
    title: templates/hub/AGENTS.md
  - resource: https://github.com/yishao1224-design/dsh-okf-plugin/blob/ec5a774e593070c87dcca36c96a5012481f280c0/templates/hub/README.md
    id: hub-readme
    title: templates/hub/README.md
  - resource: https://github.com/yishao1224-design/dsh-okf-plugin/blob/da10648a182b38b410d22f952d8e1d973d8825f0/src/host/team.ts
    id: team
    title: src/host/team.ts
  - resource: https://github.com/yishao1224-design/dsh-okf-plugin/blob/da10648a182b38b410d22f952d8e1d973d8825f0/README.md
    id: readme
    title: README.md
generated:
  by: dsh/agent
  at: 2026-11-25
---

# The OKF Toolchain (okf.py) and the Hub Template

## Why it lives in the plugin at all

The plugin does not implement bundle conventions in TypeScript; the hub repository carries its own `scripts/okf.py`, and the plugin runs **the repository's copy** so each hub can evolve its rules independently. `team.okfScript` defaults to `scripts/okf.py` and `team.python` to `python` on Windows / `python3` elsewhere [readme]. The same file ships as `templates/hub/scripts/okf.py` (1154 lines at `da10648`, stdlib only, PyYAML used when installed) so a new hub starts with it [okf-py].

The script declares its own version — `TOOLCHAIN_VERSION = "2026.11.23"`, a date, bumped on every change — and `okf.py --version` prints it with the OKF version [okf-py]. That single constant is what lets the plugin tell an out-of-date hub copy from a current one: when a maintenance round opens, and again at Open pull request, `ensureToolchain` copies the plugin's `templates/hub/scripts/okf.py` over a hub copy whose version is **older or missing**, and never touches one that is the same or newer [team]. Updating the plugin is therefore enough to bring hubs forward; see [Team Wiki service](/concepts/team-wiki-service.md).

## Subcommands

| Command | What it does |
|---|---|
| `init` | Create `concepts/ entities/ comparisons/ queries/ archive/ sources/`, seed `SCHEMA.md` and `log.md` when missing, sync `skills/` into `.claude/skills/` and `.github/prompts/`, regenerate `index.md`, run lint |
| `lint` | Conformance and review-state audit; exit 1 on errors |
| `lint --inbound PATH` | List the files linking to a bundle path (for safe deletes) |
| `index` | Regenerate `index.md` from the tree and each page's frontmatter |
| `links` | Expand `[[slug]]` / `[[slug|text]]` authoring shorthand into bundle-relative markdown links, in place |
| `log "MESSAGE"` | Prepend an entry under today's date |
| `stats` | Counts by type and `status`, the review queue with a reason per page, and stale pinned sources |
| `migrate` | Fold the retired `lifecycle` key into `status` + the `needs-review` tag, page by page; `--dry-run` lists the changes and writes nothing |
| `drift` | Recheck pinned `sources[].blob` values against the current git blob |
| `drift --hash FILE` | Print the blob sha to pin for a file |
| `--version` | Print `okf.py <TOOLCHAIN_VERSION> (OKF <version>)` and exit |

Every subcommand accepts `--bundle NAME`; without it the toolchain auto-detects a top-level folder holding a `SCHEMA.md`, falls back to `okf/`, and reports rather than guesses when several exist [okf-py].

## Frontmatter it understands

`split_frontmatter` splits a leading `---` block; `parse_frontmatter` uses `yaml.safe_load` when PyYAML is present, else a hand-written subset parser: scalars, inline `[a, b]` and `{k: v}`, block lists of scalars or mappings, and one level of nested mapping — enough for the shapes the schema template describes [okf-py]. `_scalar` strips a trailing ` #` comment, which is why the seeded templates can annotate values inline [okf-py].

Pages carry OKF v0.2's `status` alone (`draft | stable | deprecated`); the review stages are derived from it, the `needs-review` tag and `archive/`, as [Page stages](/concepts/page-stages.md) describes. The seeded `SCHEMA.md` template has no `lifecycle` line, requires `stale_after` "once status: stable", and its tag taxonomy gains a third entry, `needs-review` [okf-py].

## The lint rules

Lint is where the bundle's conventions are enforced [okf-py]:

- **Type.** A non-empty `type` is required (an error). A type missing from SCHEMA.md's *Type registry* table is a warning; a known type in the wrong top-level directory is an **error** (`archive/` is exempt). The registry is parsed from the `# Type registry` section: `` | `Type` | `dir` | … ``.
- **Status.** A value other than `draft | stable | deprecated` is an error, and the message says to flag a page for review with the `needs-review` tag instead of inventing a state [okf-py].
- **The retired `lifecycle` key.** Still present → warning ("run 'okf.py migrate'"). If its value is one of the five old ones and disagrees with `status` (through `LEGACY_LIFECYCLE_STATUS`: active and needs_review → stable, archived → deprecated), that is an **error** [okf-py].
- **Required fields.** `title`, `description`, `status`, `updated` missing → warning. A legacy `review_after` is flagged in favour of `stale_after`.
- **Review consequences.** `stable` without `stale_after` → warning; `stable` past `stale_after` and not tagged `needs-review` → warning; `deprecated` outside `archive/` without `superseded_by` or a `## Deprecation` section → error; anything under `archive/` whose status is not `deprecated` → warning.
- **Sources.** Each `sources[]` entry must be a mapping with a `resource` (error otherwise), ids must be unique, and a legacy scalar entry is flagged. `generated` and `verified`, when present, must be mappings with `by` and `at`.
- **Citations.** A `# Citations` section is flagged: v0.2 replaced it with `sources`.
- **Tags.** A tag not in SCHEMA.md's `# Tag taxonomy` list is a warning (the seeded taxonomy has `unverified`, `contested` and `needs-review`).
- **Links.** Broken bundle links to `.md` files are info; pages with `status: stable` must have at least two outbound bundle links; an unexpanded `[[wikilink]]` is a warning; a page with no inbound links and not in `index.md` is an orphan warning.
- **Reserved files.** An `index.md` with frontmatter below the bundle root is an error (OKF §6/§11).
- **Drift.** A pinned source whose file changed or vanished is a warning (skipped when git is unavailable).

`lint` prints `ERROR:`, `WARN:` and `INFO:` lines and a summary, and exits 1 when there are errors [okf-py].

## migrate

`migrate` is the one-time (per wiki) conversion off the retired key [okf-py]. For every page outside `sources/` it edits the frontmatter **as text**, so key order, comments and every other field survive: it drops the `lifecycle:` line, writes the mapped `status` when it differs, and for `needs_review` adds `needs-review` to `tags:` — inline list, block list or scalar, whichever the page uses. `--dry-run` prints `would migrate <page>: <notes>` and writes nothing. It ends by telling the operator to add `needs-review` to `SCHEMA.md`'s taxonomy, drop `lifecycle` from its templates, and run `index`, `log` and `lint`. Captures under `sources/` are never touched, so after a full migration the only legacy-key warnings lint still prints are the historical capture pages — expected, not a failure [okf-py].

## index, links, log, stats

`index` groups pages by their top-level folder and writes `--- okf_version: "0.2" ---`, `# Bundle Index`, root entries first, then one `# Section` per folder in `SECTION_ORDER` (`concepts`, `entities`, `comparisons`, `queries`, `sources`, `archive`) followed by any others alphabetically. Each line is `* [title](path) - description`, with a badge derived from the stage: ` *(archived)*` for anything under `archive/`, ` *(deprecated)*` for `status: deprecated`, ` *(needs review)*` for a stable page carrying the tag — which is exactly the shape of this bundle's `index.md` [okf-py].

`links` resolves `[[target]]` by explicit path (contains `/` or ends `.md`) or by lower-cased filename stem / title, rewrites the resolved ones to `[label](bundle/path)`, leaves code spans and fenced code untouched (so examples in `SCHEMA.md` survive), and reports ambiguous or missing targets instead of guessing [okf-py].

`log` inserts `* MESSAGE` under a `## YYYY-MM-DD` heading, creating the heading after the H1 when the date is new, and warns once the log passes 500 entries [okf-py].

`stats` counts pages by type and by `status` (a deprecated page under `archive/` counts as `deprecated (archived)`), then lists the review queue — every stable page that is tagged `needs-review` or whose `stale_after` has passed — with the reason and the due date, followed by the pinned-source drift summary [okf-py]. That queue is the same rule the Team Wiki's Needs review tab shows, computed independently.

## What the plugin runs

Open pull request runs, in the workspace, with `python <repo>/scripts/okf.py` [team]:

```sh
python scripts/okf.py links --bundle <folder>
python scripts/okf.py index --bundle <folder>
python scripts/okf.py lint  --bundle <folder>      # failure aborts, nothing committed
python scripts/okf.py log "<Verb>: [Title](/<rel>) — <PR title or wiki name>" --bundle <folder>   # once per changed page
```

Before that it makes sure the workspace **has** the toolchain: `ensureToolchain` writes the plugin's copy when the hub's base branch has none or an older one, and the pull request then carries `scripts/okf.py` (one file, needing a person once) [team]. A hub that already has a current copy is left alone.

New project wiki runs `init` then `index` for the new folder and then removes what `init` touched outside it, because `init` also re-syncs skill copies and a wiki PR may carry only its folder [team]. Because the toolchain is in place before `init` runs, a hub made by hand gets the full starter `SCHEMA.md` rather than a two-line stub; if the hub template is unavailable the PR is still pushed but marked as needing a person (`lint did not run`) [team][readme].

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

The skeleton `AGENTS.md` now states the draft rule without the retired key: an agent's page is `status: draft`, `generated: { by, at }`, "there is no `lifecycle` key" [hub-agents-src].

The workflows deliberately sit in `.github/wiki/workflows/` rather than `.github/workflows/`, so the very first push needs no `workflow` token scope; the template README explains how to move them into place — or, for a hub in Azure Repos, how to add `wiki-auto-complete-ado.yml` as a pipeline — and what to enable ("Allow auto-merge", branch protection, a required Build validation with "Contribute to pull requests" for its Build Service account, `vars.ADO_WIKI_*`, `secrets.ADO_WIKI_PAT`) [hub-readme][readme].

## Related pages

- [Page stages](/concepts/page-stages.md) — the status/tag/folder model lint and `migrate` enforce.
- [Review state and auto-merge](/concepts/review-and-auto-merge.md) — how lint output and `judge()` cooperate.
- [Team Wiki service](/concepts/team-wiki-service.md) — the calls that run the toolchain, and `ensureToolchain`.
- [Data model](/concepts/data-model.md) — the frontmatter fields lint enforces.
