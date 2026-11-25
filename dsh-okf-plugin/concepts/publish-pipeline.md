---
type: Concept
title: Publishing to the Azure DevOps Wiki
description: How a project bundle is rendered into published/Projects pages for an Azure DevOps wiki, and how a merge mirrors them there.
status: draft
confidence: high
created: 2026-11-20
updated: 2026-11-25
tags: []
sources:
  - resource: /sources/2026-11-20-readme.md
    id: readme
    title: README at 95b0615
  - resource: /sources/2026-11-20-wiki-publish-ado-workflow.md
    id: ci-ado
    title: examples/wiki-publish-to-ado.yml capture
  - resource: https://github.com/yishao1224-design/dsh-okf-plugin/blob/da10648a182b38b410d22f952d8e1d973d8825f0/src/host/publish.ts
    id: publish
    title: src/host/publish.ts
  - resource: https://github.com/yishao1224-design/dsh-okf-plugin/blob/da10648a182b38b410d22f952d8e1d973d8825f0/src/host/team.ts
    id: team
    title: src/host/team.ts
  - resource: https://github.com/yishao1224-design/dsh-okf-plugin/blob/da10648a182b38b410d22f952d8e1d973d8825f0/src/host/review.ts
    id: review
    title: src/host/review.ts
  - resource: https://github.com/yishao1224-design/dsh-okf-plugin/blob/da10648a182b38b410d22f952d8e1d973d8825f0/src/host/stage.ts
    id: stage
    title: src/host/stage.ts
generated:
  by: dsh/agent
  at: 2026-11-25
---

# Publishing to the Azure DevOps Wiki

## The idea

Publish (called kb-publish in the README) renders a project's OKF bundle into the file layout an Azure DevOps wiki expects: a page `X.md` with its sub-pages in the folder `X/`. The output lives in the hub at `published/Projects/<Project>.md` (landing page) and `published/Projects/<Project>/…` (one page per bundle page plus a page per folder). After the pull request merges, a workflow mirrors `published/Projects/` into the ADO wiki's `Projects/` section [readme][publish].

Rendering is **pure and deterministic** — there is no clock, no randomness and no network in `renderBundle` — so publishing an unchanged bundle produces no file change at all [publish]. `publish()` relies on that: it asks git whether anything under the published folder changed and reports `unchanged` when nothing did [team].

## What goes in and what comes out

`renderBundle(files, project)` takes every file of the bundle (path plus source text; non-markdown files may carry `''`) and a `PublishProject` (`name`, `folder`, hub web URL, base branch, and the registry's `summary`, `owner`, `repo`), and returns `{ dir, landing, pages, skipped }` [publish].

Skipped, with a reason recorded in `skipped` [publish]:

| Skipped | Reason |
|---|---|
| `index.md`, `log.md`, `SCHEMA.md`, any nested `index.md` | bundle bookkeeping |
| anything under `sources/` | raw source capture |
| anything under `archive/` | archived |
| a page whose derived stage is `deprecated` or `archived` — that is `status: deprecated`, on or off `archive/` | that stage [publish][stage] |

A non-markdown file is skipped silently (its reason `not markdown` is not reported), and the root `index.md` becomes the landing page rather than a skipped page [publish].

## File names

Names follow the wiki's title rules, computed by `adoName` from each page's **title** — not from its path [publish]:

- `\`, `/`, `#`, tabs and newlines become spaces; runs of whitespace collapse; leading dots are dropped; an empty title becomes `Untitled`;
- `%` becomes `%25`;
- `:`, `<`, `>`, `*`, `?`, `|`, `"` and `-` are percent-encoded (`-` → `%2D`);
- spaces become `-`.

Folder segments are title-cased first (`concepts` → `Concepts`, `how-to` → `How To`). A name that collides case-insensitively inside the same folder gets ` (2)`, ` (3)`, … appended [publish]. `publishedDir(name)` returns `published/Projects/<adoName(name)>`, and `publishedProject(path)` reverses it for a hub path [publish].

## What a published page says

Each page drops its frontmatter and gets a review line under its first heading [publish]:

- `> Reviewed by <reviewer> on <date>.` when `reviewState` says reviewed;
- `> **Due for re-review.** …` when the page's stage is `needs_review` — a stable page carrying the `needs-review` tag;
- `> **Unreviewed.** Written by <generated.by> on <date>; a person has not checked it yet.` otherwise [publish][stage].

The page ends with a footer linking back to the hub file (`<hubWeb>/blob/<branch>/<folder>/<path>`) and telling the reader to edit the bundle there, not the page [publish].

## Links

`relink` rewrites markdown links outside code fences [publish]:

| Link target | Becomes |
|---|---|
| a published page | `/Projects/<Project>/<folder path>` (with `#anchor` kept) |
| the bundle's `index.md` | `/Projects/<Project>` |
| another file of the bundle (`SCHEMA.md`, a capture) | the file's URL in the hub |
| an external URL, `#anchor`, `//host` | unchanged |
| a target that escapes the bundle (`..` past the root) | the label text, or the original link for an image |
| a broken link | plain text (or the original link for an image) |

Images are never loaded remotely: a rendered image link keeps its `![]()` form only when it points at a real bundle file; otherwise it stays as written [publish].

## Folder pages and the landing page

Every folder that holds published pages gets a page listing its members, sorted by title, with descriptions where the bundle had them; sub-folders are listed first. If a page already occupies the folder's name, no folder page is written [publish].

The landing page is built from the project metadata (summary, owner, and either a link to or the text of the code repository) plus the bundle's `index.md` body, with its headings demoted one level so they sit under the project title. When the bundle has no `index.md`, the landing page lists the top folders and loose pages instead [publish].

## How the Team Wiki service runs it

`publish(folder)` [team]:

1. finds the project (registry or scan) and reuses its open maintenance round, creating one when there is none;
2. walks the bundle folder (dot files skipped), rendering `host/publish.ts`;
3. deletes the project's published folder and landing page, then writes the rendered pages — so a page removed from the bundle leaves the wiki too, while other projects' published pages are untouched;
4. registers the workspace entry in the library and asks git whether anything under the published paths changed; if not, returns `unchanged`;
5. otherwise calls `openPr()` with the publish title and summary, which runs the toolchain, commits the wiki folder **and** the published pages, pushes, and opens or updates the pull request.

Because `judge()` treats any change under `published/` as needing a person, a publish PR is **never auto-merged** [readme][publish]. The change reaches people only after someone reviews and merges it.

## The mirror to the Azure DevOps wiki

`examples/wiki-publish-to-ado.yml` runs on a push to `main` that changes `published/Projects/**` (and on `workflow_dispatch`) [ci-ado]. It:

1. clones the existing ADO wiki repository with an `Authorization: Basic :<PAT>` header (`vars.ADO_WIKI_REPO`, `vars.ADO_WIKI_BRANCH` default `wikiMaster`, `secrets.ADO_WIKI_PAT`);
2. `rsync -a --delete published/Projects/ → <wiki>/Projects/`, creating `Projects.md` once when the wiki has none;
3. stages only `Projects` and `Projects.md`, refuses to continue if anything else is staged, and pushes.

The refusal step is the point: hand-written wiki pages outside `Projects/` are never touched, and a broken mirror fails loudly instead of rewriting the wiki [ci-ado][readme]. A `concurrency` group serializes syncs, and an empty diff exits quietly [ci-ado].

## Known gap

The README's own "not verified yet" list says published pages have **not** been opened in a real Azure DevOps wiki: the file names follow Microsoft's documented title rules, but how the wiki resolves links to names holding an encoded character (for example `%2D` for a hyphen) is untested [readme]. See [Conventions, pitfalls and open questions](/concepts/conventions-and-pitfalls.md).

## Related pages

- [Team Wiki service](/concepts/team-wiki-service.md) — the `publish()` wrapper and `openPr()`.
- [Page stages](/concepts/page-stages.md) — the stage the skip rule and the re-review line read.
- [Review state and auto-merge](/concepts/review-and-auto-merge.md) — why `published/` always needs a person.
- [Data model](/concepts/data-model.md) — the published path layout.
