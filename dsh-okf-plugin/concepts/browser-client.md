---
type: Concept
title: The Browser Client
description: The plugin's browser half — its slots, one store, the chip/dialog/team page components, the safe markdown renderer and the inline style table.
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
  - resource: /sources/2026-11-20-build-script.md
    id: build
    title: build.mjs capture
  - resource: https://github.com/yishao1224-design/dsh-okf-plugin/blob/95b0615aefbbd4d1e16040efdd37d1e1884e5ed0/src/client/index.ts
    id: client-entry
    title: src/client/index.ts
  - resource: https://github.com/yishao1224-design/dsh-okf-plugin/blob/95b0615aefbbd4d1e16040efdd37d1e1884e5ed0/src/client/store.ts
    id: store
    title: src/client/store.ts
  - resource: https://github.com/yishao1224-design/dsh-okf-plugin/blob/95b0615aefbbd4d1e16040efdd37d1e1884e5ed0/src/client/components.tsx
    id: components
    title: src/client/components.tsx
  - resource: https://github.com/yishao1224-design/dsh-okf-plugin/blob/95b0615aefbbd4d1e16040efdd37d1e1884e5ed0/src/client/team.tsx
    id: team-tsx
    title: src/client/team.tsx
  - resource: https://github.com/yishao1224-design/dsh-okf-plugin/blob/95b0615aefbbd4d1e16040efdd37d1e1884e5ed0/src/client/markdown.tsx
    id: markdown
    title: src/client/markdown.tsx
  - resource: https://github.com/yishao1224-design/dsh-okf-plugin/blob/95b0615aefbbd4d1e16040efdd37d1e1884e5ed0/src/client/styles.ts
    id: styles
    title: src/client/styles.ts
  - resource: https://github.com/yishao1224-design/dsh-okf-plugin/blob/95b0615aefbbd4d1e16040efdd37d1e1884e5ed0/src/client/locales.ts
    id: locales
    title: src/client/locales.ts
  - resource: https://github.com/yishao1224-design/dsh-okf-plugin/blob/95b0615aefbbd4d1e16040efdd37d1e1884e5ed0/src/types/contract.ts
    id: contract
    title: src/types/contract.ts
generated:
  by: dsh/agent
  at: 2026-11-20
---

# The Browser Client

## Shape

The browser half is built to CJS and wrapped by `build.mjs` in the registration the shell's lazy module table expects: one script that only registers a factory keyed by the package name, with React left as a `require` the shell answers [build]. `src/client/index.ts` exports `inject = ['slots', 'locale']`, registers the `okf` dictionaries and contributes to the slots below [client-entry].

| Slot | What it draws |
|---|---|
| `conversation.input.right` | The OKF chip in the composer's tool row; always present, including a blank session |
| `conversation.session.header.utilities` | The same chip in the session header (only once the session has messages), and the Open pull request button in a maintenance session |
| `shell.overlay` | The OKF dialog, and the Open pull request dialog while it is open |
| `main` (key `okf-team-wiki`) | The Team Wiki page |
| `sidebar.panellist` | The Team Wiki glyph and label that select that panel |

`PANEL_ID = 'okf-team-wiki'` is both the `main` key and the sidebar entry id [client-entry]. The `workspaces`, `uiWorkspace` and `layout` services are optional: they are picked up with `ctx.inject` into module-level variables, read at call time through `current()`, so the plugin loads on a shell without them and Edit as workspace degrades to a manual instruction [client-entry][store].

## One store

`src/client/store.ts` is a hand-written external store — no React inside — read by components through `useSyncExternalStore`. Its state is `{ library, selections, dialog, team }` [store].

- `call()` is the single transport: it fetches the **document-relative** route (leading slash stripped by `relative()`), with `credentials: 'same-origin'`, `cache: 'no-store'` and an `AbortSignal.timeout`, 15 s by default and 330 s for git-sized calls [store].
- A non-JSON answer is reported as `offline` — that is the symptom of the Host half not being loaded (a static fallback answers with HTML) [store].
- Requests for the library and for each session's selection are de-duplicated by an in-flight map [store].
- `openWorkspace(entry, path, prompt)` is the shared "open a session where the bundle can be edited" routine: `workspaces.create({path})`, wait for the new row to appear in the list snapshot (`waitForWorkspace`, 10 s), `uiWorkspace.connectWorkspace`, `select(sessionId, entryId)`, `layout.selectPanel(null)` to leave the Team Wiki page, then either `uiWorkspace.startSession(workspaceId, { prompt })` (prefilled composer, user presses send) or `openSession(sessionId)` [store].

Mutation methods mirror the routes: `fetchBundle`, `addLocal`, `update`, `remove`, `editAsWorkspace`, `summary`, `page`, `loadTeam`, `connectTeam`, `refreshTeam`, `setupHub`, `loadWiki`, `loadPulls`, `createWiki`, `initializeWiki`, `syncWiki`, `maintainWiki`, `publishWiki`, `loadChanges`, `openPr`, `openPrDialog`, `previewWiki`, `setNotice` [store]. Each one refreshes the library or the team status it invalidates; `loadPulls` also clears the cached wiki details when the Host answers `statusChanged` [store].

## Components

`components.tsx` holds the shared pieces [components]:

- `SessionChip` — the chip; shows `OKF · <name>` or just `OKF`, and opens the dialog for its session.
- `OkfDialog` — the overlay dialog: header with the "This session uses" selector, the library column (entries, add form), and the preview column (header, stats, catalog, page reader). Escape and a backdrop click close it.
- `AddForm` — two tabs (git / local); a `many-bundles` error's `detail.candidates` become buttons that re-submit with a chosen path.
- `Preview` / `PreviewHeader` — entry name, origin, page and type/state badges, Update (with the force option offered when the failure mentions local edits), Remove (with a "delete the downloaded files" checkbox), and Edit as workspace.
- `Catalog` — a client-side filter over title, path, description, tags, type and lifecycle, grouped by top folder with root files first.
- `PageView` — frontmatter in a `<details>` table, then the rendered body.

`team.tsx` holds the Team Wiki page: header with Refresh / Change repository / New project wiki, the two tabs (Project wikis, Local library), the project list with review counts and a "Being maintained" / "Code updated" badge, the detail pane (stats, actions Maintain / Initialize / Sync from code / Publish / Preview, tabs for pull requests, pages and needs-review), the connect form, the empty-hub set-up panel, the New project wiki modal, the maintenance-session Open pull request button and its dialog [team-tsx]. The page also polls `loadPulls` every three minutes and on window focus while a workspace has an open pull request, so a merge made on GitHub is noticed without a manual refresh [team-tsx].

## Markdown rendering is a security boundary

Bundles come from remote repositories, so their text is untrusted. `markdown.tsx` builds React elements directly and **never** uses `innerHTML`: raw HTML in a page renders as text and cannot run script [markdown]. It supports ATX headings, paragraphs, emphasis, inline code, fenced code, block quotes, nested and task lists, GFM tables, rules, links and images (rendered as links — the preview loads no remote content) [markdown].

Links are limited to `http(s)` and `mailto`; anything else (`javascript:` and friends) becomes a `<span>` carrying the target as a title. External links open with `target="_blank"` and `rel="noopener noreferrer"` [markdown]. Bundle links (`/concepts/foo.md`, `../entities/bar.md`, with `#anchor` and `?query` stripped) resolve to a bundle-relative path and are handed to `onBundleLink`, which opens the page in place [markdown]. The smoke test asserts all of this against the built artifact [readme].

## Styles and copy

`styles.ts` is one inline style table; there is no stylesheet and no CSS toolchain, and `build.mjs` fails if one appears [styles][build]. Every colour is a shell `--dsw-*` custom property so both themes work; because inline styles cannot express `:hover`, pressable components merge explicit hover/pressed entries in their handlers [styles]. `st(name)` throws on an unknown name, so a typo is a build-time-class mistake rather than a silent no-op [styles].

`locales.ts` registers the `okf` namespace with the Simplified Chinese dictionary as the key-set source of truth and an English one alongside it; `OkfKey` is derived from the Chinese keys [locales][client-entry].

## Related pages

- [Architecture](/concepts/architecture.md) — how the client artifact is built and loaded.
- [Host routes and the bundle library](/concepts/host-routes-and-library.md) — the routes the store calls.
- [Team Wiki service](/concepts/team-wiki-service.md) — the Host behaviour behind the page's buttons.
