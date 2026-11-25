---
type: Concept
title: Page Stages — OKF Status and the Derived Review States
description: How a page's lifecycle stage is read off OKF v0.2 status, the needs-review tag and the archive/ folder, what each stage means, and where the plugin and okf.py compute it.
status: draft
confidence: high
created: 2026-11-25
updated: 2026-11-25
tags: []
sources:
  - resource: https://github.com/yishao1224-design/dsh-okf-plugin/blob/da10648a182b38b410d22f952d8e1d973d8825f0/src/host/stage.ts
    id: stage
    title: src/host/stage.ts
  - resource: https://github.com/yishao1224-design/dsh-okf-plugin/blob/da10648a182b38b410d22f952d8e1d973d8825f0/src/host/review.ts
    id: review
    title: src/host/review.ts
  - resource: https://github.com/yishao1224-design/dsh-okf-plugin/blob/da10648a182b38b410d22f952d8e1d973d8825f0/src/host/bundle.ts
    id: bundle
    title: src/host/bundle.ts
  - resource: https://github.com/yishao1224-design/dsh-okf-plugin/blob/da10648a182b38b410d22f952d8e1d973d8825f0/src/host/team.ts
    id: team
    title: src/host/team.ts
  - resource: https://github.com/yishao1224-design/dsh-okf-plugin/blob/da10648a182b38b410d22f952d8e1d973d8825f0/src/types/okf.ts
    id: types
    title: src/types/okf.ts
  - resource: /sources/2026-11-25-okf-py-toolchain.md
    id: okf-py
    title: okf.py key regions at da10648
  - resource: https://github.com/yishao1224-design/dsh-okf-plugin/blob/da10648a182b38b410d22f952d8e1d973d8825f0/README.md
    id: readme
    title: README.md
generated:
  by: dsh/agent
  at: 2026-11-25
---

# Page Stages — OKF Status and the Derived Review States

## Definition

A page carries **one state field**, OKF v0.2's `status`, with exactly three
values: `draft`, `stable`, `deprecated` [okf-py][readme]. The five review stages
the plugin and the Team Wiki page show are **derived, never stored**
[stage][readme]:

| Stage | Frontmatter |
|---|---|
| `draft` | `status: draft` |
| `active` | `status: stable` |
| `needs_review` | `status: stable` **and** the `needs-review` tag |
| `deprecated` | `status: deprecated` |
| `archived` | `status: deprecated`, and the page lives under `archive/` |

`pageStage({ status, lifecycle, tags, path })` in `src/host/stage.ts` is the one
implementation of that table, with `REVIEW_TAG = 'needs-review'` and the
`PageStage` union type [stage]. It returns `undefined` when a page states no
state at all, which is what the wire records and the client fall back from: a
page with neither `status` nor a legacy `lifecycle` shows no badge [types].

`needs_review` is a **persistent flag**, not a date: a person or agent tags a
stable page when a contradiction turns up or a source changed. An overdue
`stale_after` is a *separate, computed* signal and deliberately not part of
`pageStage` — the callers that know today's date add it (the Team Wiki list and
`okf.py`'s `review_due`) [stage][review][okf-py].

## Why the key went away

The bundle used to carry a five-state `lifecycle` extension key whose value had
to agree with `status` (`active` and `needs_review` → `stable`, `archived` →
`deprecated`). That duplicated state and let the two fields disagree, so the key
was retired: OKF v0.2's `status` alone is stored, and everything finer is
derived from `status`, the tag and the folder [okf-py][readme].

**Bundles written before the change keep working.** `pageStage` still reads a
legacy `lifecycle` value first, and refines a legacy `active` to `needs_review`
when the tag is present, so an unmigrated wiki reads exactly as before
[stage]. Two moments drop the key on a page: an agent edit (`stampAgentEdit`
removes it as it writes `status: draft`) and Mark reviewed (`markReviewed`
removes it as it writes `status: stable` and strips `needs-review`)
[review]. A whole wiki is folded at once with `python scripts/okf.py migrate
--bundle <folder>` — or `migrate --dry-run` to list the pages first — and
`lint` warns on every page that still carries the key [okf-py].

## Where the stage is computed

| Place | Uses it for |
|---|---|
| `host/bundle.ts` `summarize()` | `PageSummary.stage`, and `BundleSummary.states`, the per-stage page count (falling back to `status` when no stage can be read) [bundle][types] |
| `host/review.ts` `reviewState()` | the `stage` in `ReviewState`; **reviewed** means stage `active` plus a current human `verified` entry [review] |
| `host/team.ts` `readWikiPages()` | `WikiPage.stage`, and `overdue` = stage `needs_review` or an expired `stale_after` [team] |
| `host/publish.ts` | which pages to skip (`deprecated` / `archived`) and whether to print the "Due for re-review" line [readme] |
| `lib/check-pr.js` | the CI verdict, through the same `review.ts` it bundles [readme] |
| `okf.py` | `lint`, `index` badges, `stats` counts and `review_due()` [okf-py] |

`src/client/components.tsx` and `team.tsx` only read the wire value: the
catalog filter and the page badge use `stage ?? status`, and the badge shows the
stage with its underscore replaced by a space ("needs review") [types].

One caller does not pass a path: `reviewState(source)` sees only the page text,
so a page under `archive/` reads as `deprecated` there and not `archived`. That
is harmless for the reviewed rule — neither stage is `active` — but it is why
the PR table and `check-pr.js` can say "deprecated" for an archived page, while
the catalog and Publish say "archived" [review][bundle].

## Invariants

- **A stage is never written.** No writer emits `needs_review`, `active` or
  `archived` into a file; an agent writes `status: draft` and a person's Mark
  reviewed writes `status: stable` [review].
- **Archived needs both signals.** Under `archive/` with any status other than
  `deprecated` is a lint warning ("archived pages are 'deprecated'"), and a
  `deprecated` page outside `archive/` needs `superseded_by` or a
  `## Deprecation` section [okf-py].
- **A tag on a draft means nothing.** `needs-review` on a `status: draft` page
  does not make it `needs_review`; the derivation checks `stable` first
  [stage][okf-py].
- **Two implementations, one rule.** The TypeScript `pageStage` and okf.py's
  status-plus-tag checks are kept in step by hand — there is no shared source
  between them [stage][okf-py].

## Related pages

- [Review state and auto-merge](/concepts/review-and-auto-merge.md) — how a stage plus `verified` becomes "reviewed", and what that does to a pull request.
- [The OKF toolchain](/concepts/okf-toolchain.md) — `migrate`, and the lint rules that enforce the stage model.
- [Data model](/concepts/data-model.md) — the frontmatter fields and the `PageSummary` / `WikiPage` records.
- [Conventions, pitfalls and open questions](/concepts/conventions-and-pitfalls.md) — what a legacy key or a missing status does.
