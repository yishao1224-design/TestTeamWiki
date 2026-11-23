---
type: Concept
title: Data Model and Entities
description: The records the plugin stores — bundle frontmatter, library.json, team.json, work records and wire DTOs — and the invariants that hold them together.
status: draft
lifecycle: draft
confidence: high
created: 2026-11-20
updated: 2026-11-21
tags: []
sources:
  - resource: /sources/2026-11-20-readme.md
    id: readme
    title: README at 95b0615
  - resource: /sources/2026-11-20-okf-py-toolchain.md
    id: okf-py
    title: okf.py key regions
  - resource: /sources/2026-11-20-hub-agents-template.md
    id: hub-agents
    title: templates/hub/AGENTS.md capture
  - resource: https://github.com/yishao1224-design/dsh-okf-plugin/blob/95b0615aefbbd4d1e16040efdd37d1e1884e5ed0/src/types/okf.ts
    id: types
    title: src/types/okf.ts
  - resource: https://github.com/yishao1224-design/dsh-okf-plugin/blob/95b0615aefbbd4d1e16040efdd37d1e1884e5ed0/src/host/library.ts
    id: library
    title: src/host/library.ts
  - resource: https://github.com/yishao1224-design/dsh-okf-plugin/blob/ec5a774e593070c87dcca36c96a5012481f280c0/src/host/team.ts
    id: team
    title: src/host/team.ts
  - resource: https://github.com/yishao1224-design/dsh-okf-plugin/blob/95b0615aefbbd4d1e16040efdd37d1e1884e5ed0/src/host/bundle.ts
    id: bundle
    title: src/host/bundle.ts
  - resource: https://github.com/yishao1224-design/dsh-okf-plugin/blob/95b0615aefbbd4d1e16040efdd37d1e1884e5ed0/src/host/review.ts
    id: review
    title: src/host/review.ts
  - resource: https://github.com/yishao1224-design/dsh-okf-plugin/blob/95b0615aefbbd4d1e16040efdd37d1e1884e5ed0/src/host/codesync.ts
    id: codesync
    title: src/host/codesync.ts
generated:
  by: dsh/agent
  at: 2026-11-21
---

# Data Model and Entities

There are three data models: the **bundle** (markdown on disk, the team's knowledge), the **plugin state** (`library.json` and `team.json`), and the **wire records** the two halves exchange. They meet at bundle paths.

## 1 · The bundle

### A bundle root

A folder holding a `SCHEMA.md`, or an `index.md` whose frontmatter declares `okf_version` [bundle]. Markers nested inside another bundle belong to the outer one; `.git`, `node_modules`, `.obsidian`, `.github`, `.claude` and `.vscode` are never walked [bundle].

### Reserved files

| File | Role |
|---|---|
| `index.md` | Catalog; may carry `okf_version` at the bundle root only. Frontmatter below the root is a lint error [okf-py] |
| `log.md` | Change history, one dated line per change |
| `SCHEMA.md` | Conventions: domain, type registry, frontmatter template, tag taxonomy |

### Page frontmatter

The fields the plugin and toolchain read and write [readme][okf-py]:

| Field | Meaning |
|---|---|
| `type` | Required; must appear in SCHEMA.md's type registry, and the page must sit in that type's folder (`archive/` exempt) |
| `title` | Human title; the published wiki file name is derived from it |
| `description` | One sentence, used by `index.md` and the catalog |
| `status` | OKF v0.2 standard: `draft \| stable \| deprecated` |
| `lifecycle` | This bundle's five-state extension: `draft \| active \| needs_review \| deprecated \| archived`; must agree with `status` |
| `confidence` | `low \| medium \| high` (optional) |
| `created`, `updated` | Dates |
| `stale_after` | Required once `lifecycle: active`; a past date means overdue review |
| `superseded_by` | Required (or a `## Deprecation` section) for `deprecated` |
| `tags` | List from SCHEMA.md's tag taxonomy |
| `sources[]` | `{ resource, id, title, last_modified?, blob? }` — `resource` required, ids unique; a bundle-relative `/sources/…` capture or a document-relative live file; `blob` is a pinned git blob sha |
| `generated` | `{ by, at }` — what produced the page (a person's `human` actor kind plus id, `<tool>/<version>`, `process:<id>`) |
| `verified[]` | `{ by, at }` — appended only by a real re-check; an entry whose actor kind is `human` is what makes a page reviewed |

The status/lifecycle mapping is fixed: `draft → draft`, `active → stable`, `needs_review → stable`, `deprecated → deprecated`, `archived → deprecated` [okf-py].

### Derived review state

There is no stored "reviewed" field. `reviewState` derives it from `lifecycle`, the last `verified` entry and `generated.at` [review]. See [Review state and auto-merge](/concepts/review-and-auto-merge.md).

### Hub extension keys

Two keys are this team's additions, both read/written in the project's `SCHEMA.md` and only meaningful in a team wiki [codesync]:

| Key | Meaning |
|---|---|
| `code_commit` | The code commit the wiki reflects |
| `code_synced` | The date it was synced |
| `wiki_name` | The display name of the project wiki (falls back to the folder name) [team] |

### The hub registry

`AGENTS.md` in the hub holds the project registry as the first Markdown table with a `Project` and a `Bundle` column; `Repo`, `Summary` and `Owner` are optional. Columns are matched by header name, case-insensitively and in any order, with aliases (`Name`, `Repository`, `Code`, `Description`, `Owners`, `Directory`, `Folder`, `Path`, `Wiki`); cells may use backticks and `[text](url)` links, and the first table with both required columns wins [readme]. `addRegistryRow` appends a row in the table's own column order, creating a `## Project registry` section when the file has none [hub-agents].

### Published layout

| Path | Contents |
|---|---|
| `published/Projects/<adoName(Project)>.md` | The landing page: registry metadata plus `index.md` |
| `published/Projects/<adoName(Project)>/<Folders>/<Page>.md` | One page per bundle page, plus a page per folder |

`adoName` encodes titles into wiki-safe file names and `publishedProject(path)` reverses the mapping [readme]. See [Publishing to the Azure DevOps wiki](/concepts/publish-pipeline.md).

## 2 · Plugin state

### `library.json`

```ts
interface LibraryFile { v: 1; entries: LibraryEntry[]; sessions: Record<string, { bundleId: string; selectedAt: string }> }
```

A `LibraryEntry` [types][library]:

| Field | Meaning |
|---|---|
| `id` | Lowercase letters, digits and dashes; stable, derived from the repo/folder or given |
| `name` | Display name |
| `root` | Absolute path of the bundle root |
| `origin` | `{ kind: 'git', url, provider, ref?, path, commit? }` or `{ kind: 'local' }` |
| `addedAt`, `fetchedAt?` | ISO timestamps |
| `workspace?` | Present when editable: the absolute folder a DSH Workspace opens on |
| `team?` | `{ maintainId, wiki, project? }` on an entry the Team Wiki made for a maintenance workspace |

`provider` is `'github' | 'azure-devops' | 'git'` [types]. The file is written whole (temp + rename); every mutation is serialized through one queue, and `snapshot()` serves the last read/written copy synchronously for the prompt context [library].

### `team.json`

```ts
interface TeamFile {
  v: 1
  team?: TeamConfig        // url, provider, ref?, root, name, connectedAt
  base?: string            // base branch actually used
  commit?: string          // the commit main/ is at
  refreshedAt?: string
  empty?: boolean          // repository has no commits yet
  maintains: MaintainRecord[]
  code?: Record<string, { head?: string; checkedAt: string; error?: string }>
}
```

A `MaintainRecord` [types][team]:

| Field | Meaning |
|---|---|
| `id` | `<slug>-<YYYYMMDD-HHmm>-<hex>`; also the workspace id shown in the PR comment |
| `wiki`, `wikiName` | The bundle folder and its display name |
| `branch` | `wiki/<slug>-<stamp>-<hex>`, or `wiki/new-<slug>-<hex>` for a new project |
| `workspace` | The canonical folder of the clone (`<library>/team/work/<slug>`) |
| `entryId` | `team-<slug>` — one library entry per project wiki |
| `createdAt`, `isNew?` | Timestamps and origin |
| `pr?` | `{ number, url }` |
| `closed?` | `'merged' \| 'closed'` — the round is over; the next Maintain starts a new one in the same folder |

### Session selection

`{ bundleId, selectedAt }` keyed by session id, in `library.json` — deliberately **not** in the session log, so choosing a bundle writes no event an older DSH build would refuse to load [readme][library]. A subagent session with no selection of its own inherits its `parentSession`'s [readme].

## 3 · Wire records

The Host and browser share these in `src/types/okf.ts` [types]:

| Record | Direction | Holds |
|---|---|---|
| `FetchRequest`, `AddLocalRequest` | browser → Host | URL / ref / path / id / name, or a local path |
| `EditWorkspaceAnswer` | Host → browser | the now-editable entry, the workspace folder, `created` |
| `BundleSummary` | Host → browser | entry, `okfVersion`, `pages[]` (`PageSummary`), `types{}`, `states{}`, `untyped[]`, `truncated` |
| `PageContent` | Host → browser | path, parsed `meta`, body |
| `TeamStatus` | Host → browser | team config, base branch, commit, `empty`, `wikis[]` (`ProjectWiki`), `maintains[]`, `hasToken` |
| `ProjectWiki` | Host → browser | folder, name, registered, repo/summary/owner, missing, published, stats, maintainId, code state |
| `WikiDetail` | Host → browser | the `ProjectWiki` plus `WikiPage[]` with review state |
| `PullsAnswer` | Host → browser | `pulls[]` (`PullView`), `at`, `note`, `statusChanged` |
| `PullView` | Host → browser | number, title, url, author, draft, branch, labels, wikis, checks, eligible, reasons, files, maintainId |
| `ChangesAnswer` | Host → browser | the record, changed files with review state, uncommitted count, suggested PR text |
| `MaintainAnswer` | Host → browser | record, entry, workspace, `created`, `initialize?`, `initializeError?`, `sync?`, `pr?`, `prError?` |
| `PublishAnswer` | Host → browser | record, `dir`, page count, `skipped[]`, `unchanged?`, `pr?` |
| `OpenPrAnswer` | Host → browser | branch, commit, files, eligible, reasons, pushed, pr, stamped, notes |
| `ApiError` | Host → browser | `{ error: { code, message, detail? } }` |

## How the entities relate

```text
AGENTS.md registry row ──(Bundle column)──▶ project wiki folder ──▶ bundle root
                                                   │        (SCHEMA.md, index.md, pages, sources/)
                                                   │
library.json ──entry.id──▶ repos/<id>/ (git)      │
     │  └─entry.workspace──▶ DSH Workspace ────────┤
     │                                             │
     └─sessions[sessionId].bundleId ──▶ prompt context (catalog + rules)

team.json ──team──▶ hub repository
     ├─maintains[i].wiki ──▶ the same project wiki folder
     │     ├─workspace ──▶ work/<slug>/  ──▶ .code/<slug>/ (read-only code)
     │     └─pr ──▶ GitHub or Azure DevOps pull request ──▶ branch wiki/…
     └─code[folder].head ──▶ the code repository's HEAD as last seen
```

## Invariants, and where they hold

| Invariant | Enforced by |
|---|---|
| A local folder is never modified or deleted by the library | `remove()` deletes only `repos/<id>` checkouts [library] |
| An editable bundle is never force-updated | `update()` refuses `force` when `workspace` is set; `fastForward()` refuses dirty trees [library] |
| A page path never escapes the bundle | `resolvePage()` [bundle] |
| A wiki PR carries one project only | `judge()` prefix check and `check-pr.js`'s one-wiki rule [readme] |
| Captures are immutable | `judge()` rejects modified/deleted `sources/` files [readme] |
| Only a person makes a page reviewed | `reviewState` requires a `human` actor kind; the agent stamp never writes `verified` [review] |
| Published pages are generated | `renderBundle` is deterministic; the PR always needs a person [readme] |
| The base branch only changes through a PR | every write goes to a `wiki/…` branch, pushed and opened as a PR [team] |

## Related pages

- [Host routes and the bundle library](/concepts/host-routes-and-library.md) — where `library.json` is written.
- [Team Wiki service](/concepts/team-wiki-service.md) — where `team.json` and maintenance records are written.
- [Review state and auto-merge](/concepts/review-and-auto-merge.md) — the derivation rule.
