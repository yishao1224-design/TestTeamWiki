---
type: Concept
title: Review State and Auto-Merge
description: How a page counts as reviewed, how an agent edit resets it, and the eligibility rule that decides whether a wiki pull request may merge without a person.
status: draft
lifecycle: draft
confidence: high
created: 2026-11-20
updated: 2026-11-20
tags: []
sources:
  - resource: /sources/2026-11-20-readme.md
    id: readme
    title: README at 95b0615
  - resource: /sources/2026-11-20-hub-agents-template.md
    id: hub-agents
    title: templates/hub/AGENTS.md capture
  - resource: /sources/2026-11-20-wiki-auto-merge-workflow.md
    id: ci-auto-merge
    title: examples/wiki-auto-merge.yml capture
  - resource: /sources/2026-11-20-okf-py-toolchain.md
    id: okf-py
    title: okf.py key regions
  - resource: https://github.com/yishao1224-design/dsh-okf-plugin/blob/95b0615aefbbd4d1e16040efdd37d1e1884e5ed0/src/host/review.ts
    id: review
    title: src/host/review.ts
  - resource: https://github.com/yishao1224-design/dsh-okf-plugin/blob/95b0615aefbbd4d1e16040efdd37d1e1884e5ed0/src/host/eligibility.ts
    id: eligibility
    title: src/host/eligibility.ts
  - resource: https://github.com/yishao1224-design/dsh-okf-plugin/blob/95b0615aefbbd4d1e16040efdd37d1e1884e5ed0/src/check-pr.ts
    id: check-pr
    title: src/check-pr.ts
  - resource: https://github.com/yishao1224-design/dsh-okf-plugin/blob/95b0615aefbbd4d1e16040efdd37d1e1884e5ed0/src/host/summary.ts
    id: summary
    title: src/host/summary.ts
  - resource: https://github.com/yishao1224-design/dsh-okf-plugin/blob/95b0615aefbbd4d1e16040efdd37d1e1884e5ed0/src/host/registry.ts
    id: registry
    title: src/host/registry.ts
generated:
  by: dsh/agent
  at: 2026-11-20
---

# Review State and Auto-Merge

## No new field

The bundle carries no "reviewed" flag. A page is **reviewed** when all three hold [review][readme]:

1. `lifecycle: active`;
2. the last entry of `verified:` names a person — its `by` value starts with the `human` actor kind — with an `at` date;
3. that verification is not older than `generated.at` — i.e. nobody rewrote the page after the person confirmed it.

Everything else is **unreviewed**. "Overdue" is a separate idea used by the Team Wiki list: `lifecycle: needs_review`, or a `stale_after` date before today [readme].

`reviewState(source)` reads this out of the frontmatter text: it splits the page at its `---` fences, groups lines by top-level key so nested blocks stay with their key, parses `generated` and `verified` in either flow style (`{by: x, at: y}`) or block style, and compares dates as text after truncating to 10 characters [review]. It returns `{ state, lifecycle, reviewer, reviewedAt, generatedBy, generatedAt }`, and `describeReview` renders it for the PR table (`reviewed by alice on 2026-08-27`, or `unreviewed, written by dsh/agent on …`) [review].

## The agent-edit stamp

An agent cannot make a page reviewed; it can only make it unreviewed. `stampAgentEdit(source, agent, date)` rewrites exactly four top-level keys and leaves every other key alone [review]:

```yaml
status: draft
lifecycle: draft
updated: <date>
generated:
  by: <agent>
  at: <date>
```

It adds `created: <date>` when the page has none, keeps `verified` (it is history) and keeps comments and blank lines that trailed a replaced value [review]. `today()` returns the local `YYYY-MM-DD` [review].

The Team Wiki service calls this in two places: Open pull request resets any reviewed page that was edited without resetting its own stamp, and New project wiki / Maintain rely on the session's rules, which instruct the agent to write `generated` itself [readme][team].

## The eligibility rule

`judge(bundleDir, files)` is pure: it takes the changed files (path, status, before text, after text) and returns `{ eligible, reasons, pages }` [eligibility]. A change is eligible only when **every** changed file falls into one of these buckets:

| Bucket | Verdict |
|---|---|
| `index.md` or `log.md` inside the wiki | allowed |
| a `sources/…` file that is **added** | allowed (captures are append-only) |
| another markdown page inside the wiki, unreviewed before and after | allowed |
| anything under `published/` | needs a person — pages published for people |
| `AGENTS.md` | needs a person — the project registry |
| a file outside the wiki folder | needs a person — outside the wiki |
| `SCHEMA.md` | needs a person — the conventions |
| `archive/…` | needs a person |
| a non-markdown file | needs a person |
| a deleted page | needs a person |
| a page reviewed before the change | needs a person — it resets to unreviewed |
| a page that claims a human review after the change | needs a person — only a person may stamp `verified` |
| an existing capture that is modified or deleted | needs a person — captures are immutable |

An empty change is also a reason. The same function backs the Host (`pulls()` and `openPr()`) and `lib/check-pr.js`, so the label the plugin sets and the verdict CI computes come from one implementation [eligibility][readme].

## The CI checker

`src/check-pr.ts` builds into `lib/check-pr.js`, a Node-built-ins-only script copied into the hub as `.github/wiki/check-pr.js` [readme]. Usage:

```text
node check-pr.js <base-ref> [head-ref=HEAD] [wiki-root]
```

It diffs `merge-base(base, head)..head` with `git diff --name-status --no-renames -z`, reads each changed file before and after with `git show`, and maps every path to a project [check-pr]:

- through the hub's `AGENTS.md` registry, read on **both** sides of the change (`merge-base` and `head`), so a PR cannot hide a project by editing the registry;
- `published/Projects/<Project>/…` maps to the registry entry whose project encodes to the same name;
- a PR adding a `…/SCHEMA.md` maps that new folder to the PR;
- without a registry, the folder directly under the wiki root that holds a `SCHEMA.md`, or the root itself when it is a bundle.

Exit codes: `0` eligible, `1` needs a person (reasons printed), `2` usage or git error. It prints `project wiki: <folder>` and one line per page with its review state before and after, and when `GITHUB_OUTPUT` is set it writes `wiki=<folder>` there so the workflow can lint exactly that bundle [check-pr].

The workflow takes the checker and the linter **from the base branch**, so a PR cannot weaken the rules that judge it, and only then enables GitHub auto-merge [ci-auto-merge]:

```yaml
git show "origin/${{ github.base_ref }}:.github/wiki/check-pr.js" > "$RUNNER_TEMP/check-pr.mjs"
node "$RUNNER_TEMP/check-pr.mjs" "origin/${{ github.base_ref }}" HEAD
git show "origin/${{ github.base_ref }}:scripts/okf.py" > scripts/okf.py
python scripts/okf.py lint --bundle "${{ steps.check.outputs.wiki }}"
gh pr merge --auto --squash …
```

The workflow only runs when the `wiki-auto-merge` label is present, so an ineligible PR is never considered; the label is a hint, not the decision [ci-auto-merge].

## Companion: lint

Eligibility says what a person must see; `okf.py lint` says whether the bundle is well formed. Open pull request runs the repository's own `links`, `index` and `lint` before committing, and a lint error aborts the whole operation (`lint-failed`, nothing committed or pushed) [team]. The linter's frontmatter, lifecycle, tag, link and orphan rules are described in [The OKF toolchain](/concepts/okf-toolchain.md).

## Suggested PR titles and summaries

`src/host/summary.ts` turns a change list into the title and body the dialog prefills: it counts added / modified / removed pages, new source captures and republished wiki pages, lists titles and descriptions, caps each list at 25 lines and the title at 120 characters, and has dedicated titles for a new wiki (`Add the <Name> project wiki`), a single page, and a publish-only change [summary]. The Team Wiki service calls it from `changes()` and again from `openPr()` when the dialog was left empty [team].

## Related pages

- [Team Wiki service](/concepts/team-wiki-service.md) — where `judge()` and the stamp are applied.
- [Data model](/concepts/data-model.md) — the `verified` / `generated` field shapes.
- [Conventions, pitfalls and open questions](/concepts/conventions-and-pitfalls.md) — date-granularity and other edge cases.
