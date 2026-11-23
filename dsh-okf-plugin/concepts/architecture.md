---
type: Concept
title: Architecture of the dsh-okf Plugin
description: The two halves of the plugin, the components inside each, how they talk to each other, and where state lives on disk.
status: draft
lifecycle: draft
confidence: high
created: 2026-11-20
updated: 2026-11-23
tags: []
sources:
  - resource: /sources/2026-11-20-readme.md
    id: readme
    title: README at 95b0615
  - resource: /sources/2026-11-20-build-script.md
    id: build
    title: build.mjs capture
  - resource: /sources/2026-11-20-package-manifest.md
    id: manifest
    title: package.json capture
  - resource: https://github.com/yishao1224-design/dsh-okf-plugin/blob/b71e34fb1c76d88af680440531fda3c5a9c75c34/src/index.ts
    id: host-entry
    title: src/index.ts
  - resource: https://github.com/yishao1224-design/dsh-okf-plugin/blob/b71e34fb1c76d88af680440531fda3c5a9c75c34/src/types/contract.ts
    id: contract
    title: src/types/contract.ts
  - resource: https://github.com/yishao1224-design/dsh-okf-plugin/blob/95b0615aefbbd4d1e16040efdd37d1e1884e5ed0/src/types/okf.ts
    id: types
    title: src/types/okf.ts
  - resource: https://github.com/yishao1224-design/dsh-okf-plugin/blob/95b0615aefbbd4d1e16040efdd37d1e1884e5ed0/src/host/library.ts
    id: library
    title: src/host/library.ts
  - resource: https://github.com/yishao1224-design/dsh-okf-plugin/blob/b71e34fb1c76d88af680440531fda3c5a9c75c34/src/host/team.ts
    id: team
    title: src/host/team.ts
  - resource: https://github.com/yishao1224-design/dsh-okf-plugin/blob/b71e34fb1c76d88af680440531fda3c5a9c75c34/src/host/tools.ts
    id: tools
    title: src/host/tools.ts
  - resource: https://github.com/yishao1224-design/dsh-okf-plugin/blob/95b0615aefbbd4d1e16040efdd37d1e1884e5ed0/src/client/index.ts
    id: client-entry
    title: src/client/index.ts
  - resource: https://github.com/yishao1224-design/dsh-okf-plugin/blob/95b0615aefbbd4d1e16040efdd37d1e1884e5ed0/src/client/store.ts
    id: store
    title: src/client/store.ts
generated:
  by: dsh/agent
  at: 2026-11-23
---

# Architecture of the dsh-okf Plugin

## Two halves, three artifacts

The package ships three build artifacts, all produced by `build.mjs` and committed under `lib/` [build][manifest]:

| Artifact | Format | Role |
|---|---|---|
| `lib/index.js` | ESM, Node | The Host half; the Loader imports it for the `okf` row. May import only `node:` built-ins. |
| `lib/check-pr.js` | ESM, Node | The CI eligibility checker copied into a wiki hub as `.github/wiki/check-pr.js`. |
| `lib/client.js` | CJS factory, browser | The browser half, wrapped in the `window.__ModuleLoader__.load({ id, factory })` registration the shell's lazy module table expects. |

`build.mjs` enforces the boundaries rather than trusting them: it fails if the Host artifact imports anything not starting with `node:`, if the browser artifact does not open with the module-loader registration, if the package name is missing, if `dsh.client.platform` is not `web`, or if a stylesheet toolchain (`.module.css`, `lightningcss`) leaked into the browser artifact [build]. React and React DOM are `neverBundle` so the shell's single React instance is the one the components run against [build].

## The Host half

`src/index.ts` exports `inject = ['connection']` and an `apply(ctx, config)` that:

1. builds a `Library` over the configured `libraryDir` (default `<DSH home>/okf`),
2. builds a `Team` service over `<libraryDir>/team`,
3. registers every handler in its exported `handlers` table as an exact Fetch route under `/api/okf` with a buffered body [host-entry],
4. registers the prompt context `okf:session-bundle` through `ctx.inject(['systemPrompt'], …)`, so the library and preview still work if a Harness build does not expose that service [host-entry],
5. registers the two Team Wiki tools through `ctx.inject(['tools'], …)` unless `config.tools` is `false`: `wiki_sync_from_code` and `wiki_open_pr`, built by `wikiTools(team)` [host-entry][tools].

The route table is shared by both halves in `src/types/okf.ts` (`ROUTES`, `relative()`); the browser fetches the document-relative form so the session cookie rides along and the Connection Host/Origin fence applies [types][host-entry]. The tools do not go through a route: they are global entries in the Harness's tool registry, registered once for every session, and they reach the same `Team` service the routes do [host-entry][tools].

Host modules and their jobs:

| Module | Job |
|---|---|
| `host/git.ts` | Remote parsing (GitHub, Azure DevOps, any git), the `git` runner, clone / sparse checkout / pull / fast-forward / unshallow, credential redaction |
| `host/library.ts` | `library.json`, `repos/<id>/`, fetch / update / add-local / remove / edit-workspace, per-session selections |
| `host/bundle.ts` | Bundle-root detection, catalog summary, safe page reads (path-traversal guards, page and byte caps) |
| `host/frontmatter.ts` | A dependency-free YAML-frontmatter reader for flat keys and scalar lists |
| `host/knowledge.ts` | The per-session prompt context, the `index.md` cache, editing rules |
| `host/team.ts` | The Team Wiki service: project list, pull requests, Maintain, New project wiki, Open pull request, Sync from code (first round = the Initialize read-through), Set up the hub, Publish |
| `host/tools.ts` | The model tools `wiki_sync_from_code` and `wiki_open_pr`, so an automation task can run Sync from code and Open pull request without the page; they call `team.sync()`, `team.openPr()` and `team.wikiAt()` |
| `host/review.ts` | Reviewed / unreviewed from frontmatter; the agent-edit stamp; frontmatter text surgery |
| `host/eligibility.ts` | The auto-merge rule, shared by the Host and `lib/check-pr.js` |
| `host/registry.ts` | The hub's `AGENTS.md` project-registry table (parse and append) |
| `host/github.ts` | The few GitHub REST calls (open/update PR, label, list PRs, checks, state) |
| `host/azure.ts` | The same calls for Azure DevOps (pull requests, tags, statuses, auto-complete) |
| `host/publish.ts` | Rendering a bundle into Azure DevOps wiki pages |
| `host/codesync.ts` | The recorded code commit, affected pages, and the sync task prompt |
| `host/summary.ts` | A suggested PR title and summary from the changed files |
| `host/run.ts` | Child processes (`git`, `python`) with the token header and redaction |

The GitHub and Azure DevOps clients sit behind one `PullHost` shape in `team.ts`: who is behind the token, open or update a pull request, label it, list open ones with their checks, read whether one merged or closed, and the ref to fetch a pull request's head from. Only Azure DevOps gets a `setAutoMerge` (auto-complete); on GitHub the hub's workflow drives the merge instead.

## The browser half

`src/client/index.ts` exports `inject = ['slots', 'locale']` and an `apply(ctx)` that registers the `okf` locale dictionaries and contributes to five slots [client-entry]:

- `conversation.input.right` — the OKF chip in the composer's tool row (always reachable, including blank sessions);
- `conversation.session.header.utilities` — the same chip in the session header, plus the Open pull request button in maintenance sessions;
- `shell.overlay` — the OKF dialog and the Open pull request dialog;
- `main` (key `okf-team-wiki`) and `sidebar.panellist` — the Team Wiki page and its sidebar glyph.

`workspaces`, `uiWorkspace` and `layout` are reached with `ctx.inject`, not the hard inject list, so the dialog still loads on a shell that lacks them; Edit as workspace then just reports the folder for the user to add by hand [client-entry].

One store (`src/client/store.ts`) holds the library list, each session's selection, the dialog state and the Team Wiki state; components read it through `useSyncExternalStore` [store]. See [The browser client](/concepts/browser-client.md).

## How the halves talk

```text
browser (client.js)                      dsh Host process (index.js)
  store.call(relative(route))  ──fetch──▶  connection.fetch handler
       │                                        │
       │                                   library.ts ── git ──▶ remote
       │                                   team.ts ───── git/python ──▶ hub,
       │                                        │                      code repo,
       │                                        │                      GitHub / Azure DevOps REST
       │◀──────── JSON { entry | status | … } ──┘
       │
  system prompt ◀── ctx.inject systemPrompt.context('okf:session-bundle')
  tool registry ◀── ctx.inject tools.register(wiki_sync_from_code, wiki_open_pr)
```

Three properties matter for the boundary:

- **The Host is the only writer.** The browser never touches the filesystem; it calls routes. Everything that needs `git`, `python`, a token or the disk lives on the Host [host-entry][store].
- **The Host never prompts.** Git runs with `GIT_TERMINAL_PROMPT=0`, empty askpass and `GCM_INTERACTIVE=never`; a missing credential becomes an error answer instead of a hung request [readme].
- **Tokens stay out of argv and files.** Both `host/git.ts` and `host/run.ts` pass a token as `http.extraHeader` through `GIT_CONFIG_COUNT` / `GIT_CONFIG_KEY_n` / `GIT_CONFIG_VALUE_n`; error text is redacted on the way out [readme].

## Where state lives

| Location | What it holds |
|---|---|
| `<libraryDir>/library.json` | `v: 1`, `entries[]` (`LibraryEntry`), `sessions{}` (session id → `{ bundleId, selectedAt }`) [library] |
| `<libraryDir>/repos/<id>/` | One shallow, sparse, blobless git checkout per fetched bundle; after Edit as workspace it is unshallowed and becomes the workspace [library] |
| `<libraryDir>/team/team.json` | The connected hub, base branch, last commit, `maintains[]`, last-seen code heads (`code`) [team] |
| `<libraryDir>/team/main/` | A read-only clone of the hub's base branch that the page reads (never edited) [team] |
| `<libraryDir>/team/work/<slug>/` | One maintenance clone per project wiki, reused round after round on a new `wiki/…` branch [team] |
| `<maintenance workspace>/.code/<slug>/` | The project's code, cloned `--filter=blob:none`, excluded from git via `.git/info/exclude` [team] |
| The hub repository | `AGENTS.md` registry, one OKF bundle per project, `scripts/okf.py`, `published/Projects/`, `.github/wiki/…` [readme] |

The default `libraryDir` is `$DSH_HOME/okf` when `DSH_HOME` is set, else `~/.dsh/okf`; `apply()` computes it in `dshHome()` and passes it to `openLibrary` [host-entry].

## Deployment shape

There is no server to run: the bundle is installed into a DSH profile and loaded in-process. `npm run build` emits the artifacts, `plugin_manager action=install_bundle` installs the folder, and `dsh web` must be **restarted** because Host code loads only at startup (the browser half hot-loads) [readme]. See [Build, run, test, deploy](/concepts/build-run-test-deploy.md).

## Related pages

- [Host routes and the bundle library](/concepts/host-routes-and-library.md) — the route table and the library's guarantees.
- [Team Wiki service](/concepts/team-wiki-service.md) — the largest Host module.
- [Automation tasks and the Team Wiki tools](/concepts/automation-tasks.md) — what step 5 registers and why.
- [The browser client](/concepts/browser-client.md) — slots, store and rendering.
- [Data model](/concepts/data-model.md) — every record and file shape.
