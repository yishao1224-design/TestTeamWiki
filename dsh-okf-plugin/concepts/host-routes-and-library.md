---
type: Concept
title: Host Routes and the Bundle Library
description: The Host entry's twenty-four /api/okf routes, error mapping, the library.json store, and the git and bundle-reading modules behind it.
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
  - resource: /sources/2026-11-20-cordis-patch.md
    id: patch
    title: cordis.patch.yml capture
  - resource: https://github.com/yishao1224-design/dsh-okf-plugin/blob/b71e34fb1c76d88af680440531fda3c5a9c75c34/src/index.ts
    id: host-entry
    title: src/index.ts
  - resource: https://github.com/yishao1224-design/dsh-okf-plugin/blob/b71e34fb1c76d88af680440531fda3c5a9c75c34/src/host/tools.ts
    id: tools
    title: src/host/tools.ts
  - resource: https://github.com/yishao1224-design/dsh-okf-plugin/blob/ec5a774e593070c87dcca36c96a5012481f280c0/src/host/azure.ts
    id: azure
    title: src/host/azure.ts
  - resource: https://github.com/yishao1224-design/dsh-okf-plugin/blob/95b0615aefbbd4d1e16040efdd37d1e1884e5ed0/src/types/okf.ts
    id: types
    title: src/types/okf.ts
  - resource: https://github.com/yishao1224-design/dsh-okf-plugin/blob/95b0615aefbbd4d1e16040efdd37d1e1884e5ed0/src/host/library.ts
    id: library
    title: src/host/library.ts
  - resource: https://github.com/yishao1224-design/dsh-okf-plugin/blob/95b0615aefbbd4d1e16040efdd37d1e1884e5ed0/src/host/git.ts
    id: git
    title: src/host/git.ts
  - resource: https://github.com/yishao1224-design/dsh-okf-plugin/blob/95b0615aefbbd4d1e16040efdd37d1e1884e5ed0/src/host/bundle.ts
    id: bundle
    title: src/host/bundle.ts
  - resource: https://github.com/yishao1224-design/dsh-okf-plugin/blob/95b0615aefbbd4d1e16040efdd37d1e1884e5ed0/src/host/frontmatter.ts
    id: frontmatter
    title: src/host/frontmatter.ts
  - resource: https://github.com/yishao1224-design/dsh-okf-plugin/blob/95b0615aefbbd4d1e16040efdd37d1e1884e5ed0/src/host/knowledge.ts
    id: knowledge
    title: src/host/knowledge.ts
  - resource: https://github.com/yishao1224-design/dsh-okf-plugin/blob/95b0615aefbbd4d1e16040efdd37d1e1884e5ed0/src/host/run.ts
    id: run
    title: src/host/run.ts
generated:
  by: dsh/agent
  at: 2026-11-23
---

# Host Routes and the Bundle Library

## The entry point

`src/index.ts` is the Host module. It exports `inject = ['connection']` — the only hard dependency — plus `Config` (the row config), `handlers` (exported so the smoke test can call routes directly), and `apply(ctx, config)` [host-entry]. `connection` is what routes are registered on; `systemPrompt` and `tools` are reached optionally through `ctx.inject`, so the library and preview keep working on a Harness build that does not expose them [host-entry].

`Config` keys and their defaults, all optional [host-entry][patch]:

| Key | Default |
|---|---|
| `libraryDir` | `<DSH home>/okf` (`$DSH_HOME`, else `~/.dsh`) |
| `gitPath` | `git` |
| `gitTimeoutMs` | `300000` |
| `indexChars` | `8000` (0 sends no catalog) |
| `promptContext` | `true` |
| `tools` | `true` (registers `wiki_sync_from_code` and `wiki_open_pr`) |
| `team.*` | see [Team Wiki service](/concepts/team-wiki-service.md) |

A patch replaces the whole `config` block, so a partial config does not merge with a previous one [patch][readme].

## The routes

Every route is an **exact Fetch route** on `connection.fetch.register` with `requestBody: 'buffered'`, and everything sits under `/api/okf`. The plugin uses `connection.fetch` rather than `connection.rpc`, because under strict injection `rpc.handle` throws for an out-of-tree plugin [host-entry]. There are twenty-four routes [types]; the smoke test asserts that count and the prefix.

| Route | Method | Purpose |
|---|---|---|
| `/api/okf/library` | GET | Refresh and list the library, plus `libraryDir` |
| `/api/okf/fetch` | POST | Clone a remote into the library |
| `/api/okf/update` | POST | Pull the latest commit of a git entry (`force` discards local edits) |
| `/api/okf/add-local` | POST | Register a bundle already on disk |
| `/api/okf/edit-workspace` | POST | Make a bundle editable in place; answers the workspace folder |
| `/api/okf/remove` | POST | Forget an entry (`deleteFiles` also removes a git checkout) |
| `/api/okf/bundle` | GET | The bundle summary (catalog) |
| `/api/okf/page` | GET | One page's frontmatter and body |
| `/api/okf/session` | GET / POST | Read, set or clear a session's bundle |
| `/api/okf/team` | GET | Team Wiki status |
| `/api/okf/team/connect` | POST | Connect (or change) the hub repository |
| `/api/okf/team/refresh` | POST | Re-read the base branch and the code heads |
| `/api/okf/team/wiki` | GET | One project wiki, page by page, with review state |
| `/api/okf/team/pulls` | GET | Open pull requests and verdicts (`?refresh=1` bypasses the cache) |
| `/api/okf/team/create` | POST | New project wiki |
| `/api/okf/team/maintain` | POST | Open (or make) a maintenance workspace |
| `/api/okf/team/publish` | POST | Render the bundle into `published/Projects/` |
| `/api/okf/team/setup` | POST | Set up the hub in an empty repository |
| `/api/okf/team/initialize` | POST | Check out the code and write the read-through task |
| `/api/okf/team/sync` | POST | Diff the code since the recorded commit and write the update task |
| `/api/okf/team/review` | POST | Mark pages reviewed as the token's account (a `verified` entry) |
| `/api/okf/team/changes` | GET | What a maintenance workspace holds that the base does not |
| `/api/okf/team/open-pr` | POST | Stamp, lint, commit, push, open or update the pull request |
| `/api/okf/team/attach` | POST | Add the base-branch wiki to the library, read-only |

`src/types/okf.ts` holds the same table as `ROUTES`, the request and answer records for each route, and `relative()`, which strips the leading slash for the browser's document-relative fetch [types].

Not everything the Host entry registers is a route. Besides the prompt context, `apply()` registers the two Team Wiki model tools `wiki_sync_from_code` and `wiki_open_pr` through `ctx.inject(['tools'], …)` — globally, for every session, unless `config.tools` is `false`. They call the same `Team` service the `/api/okf/team` routes do, and are described in [Automation tasks and the Team Wiki tools](/concepts/automation-tasks.md) [host-entry][tools].

## Error mapping

One wrapper adds the method check (405 with an `allow` header) and turns failures into JSON [host-entry]:

| Thrown | Answer |
|---|---|
| `LibraryError` | its own status and `{ code, message, detail? }` (4xx; codes include `bad-url`, `bad-ref`, `bad-path`, `many-bundles`, `exists`, `unknown-bundle`, `unsaved-work`, `lint-failed`, `nothing-changed`) |
| `GitError` | 502, `git-<code>` (`auth`, `not-found`, `ref`, `timeout`, `missing-git`, `failed`) |
| `RunError` | 502, `command-failed`, with the last 6000 characters of stdout |
| `GitHubError` | 502, `github-<status>` — also what `host/azure.ts` throws, so Azure DevOps failures use the same code [azure] |
| `ENOENT` | 404, `not-found` |
| anything else | 500, `failed` |

Every answer carries `content-type: application/json` and `cache-control: no-store` [host-entry].

## The library on disk

`<libraryDir>/library.json` holds `{ v: 1, entries, sessions }`; `<libraryDir>/repos/<id>/` holds one checkout per fetched bundle [library]. Two invariants shape the implementation:

- **One queue per library.** Every mutating operation runs through a promise chain (`serial`), so two requests never interleave [library].
- **Whole-file writes.** `save()` writes a temp file and renames it over `library.json`, retrying up to five times on Windows `EPERM`/`EBUSY`/`EACCES`; after a successful rename it updates the in-memory snapshot and notifies listeners [library].

`Library` exposes `list`, `get`, `fetch`, `update`, `addLocal`, `remove`, `editWorkspace`, `put`, `getSession`, `setSession`, `refresh`, `snapshot`, `onChange` [library]. `put` is how the Team Wiki registers maintenance workspaces as ordinary library entries.

### Fetch

`fetch()` parses the remote, normalizes the subfolder with `cleanRepoPath` (rejecting `..` and `.git`), validates the ref against option injection and whitespace, derives a unique slug id, then [library]:

1. `cloneNoCheckout` into `repos/.tmp-<id>-<hex>` — `git clone --depth 1 --single-branch --no-tags --filter=blob:none --no-checkout` [git];
2. `git ls-tree -r --name-only -z HEAD` lists paths; only candidate `index.md` files are read with `git show` to test for `okf_version` [git][bundle];
3. `findBundleRoots` decides: no marker → `no-bundle`; several → `many-bundles` with the candidates in `detail`; one → that folder [bundle];
4. `checkoutPath` runs `git sparse-checkout set --no-cone /<folder>/` and `git checkout` [git];
5. the temp folder is renamed into place only after success. If `repos/<id>` already exists with unsaved work, the fetch is refused (`unsaved-work`) instead of overwriting it [library].

A failed clone removes its temp folder, so nothing is left behind [library][readme].

### Update, remove and edit

- `update(id, force)` refuses Team Wiki maintenance workspaces (`team-workspace`) and local entries (`not-git`). An entry with `workspace` set (being edited) only ever fast-forwards and cannot be forced; otherwise a shallow `pull` (fetch `--depth 1`, then `reset --hard FETCH_HEAD`) runs, and `force` discards local edits [library][git].
- `remove(id, deleteFiles)` detaches the entry from every session. With `deleteFiles` it refuses while the checkout has uncommitted changes or unpushed commits; a **local** folder is never deleted, only forgotten [library].
- `editWorkspace(id)` is idempotent. For a git entry it runs `git fetch --unshallow` (keeping the blob filter) and makes `repos/<id>` the workspace; for a local entry the workspace is the git top level when the bundle root is inside one, else the bundle folder [library][git].

### Tokens

`TOKEN_ENV` maps provider → environment variables, most specific first: `azure-devops` → `DSH_OKF_AZURE_TOKEN`, then `DSH_OKF_GIT_TOKEN`; `github` → `DSH_OKF_GITHUB_TOKEN`, then `DSH_OKF_GIT_TOKEN`; any other host → `DSH_OKF_GIT_TOKEN` [library]. The token is read at call time from an injectable `env` (a test seam) and handed to git only as a config header [library][git].

## The git runner

`src/host/git.ts` is the only module that starts `git` for the library. `parseRemote` recognizes GitHub, Azure DevOps (all three URL forms) and generic hosts; it refuses whitespace, a leading `-`, unsupported protocols, and a URL carrying a password [git]. `gitEnv` sets `GIT_TERMINAL_PROMPT=0`, blank `GIT_ASKPASS`/`SSH_ASKPASS`, `GCM_INTERACTIVE=never` and `ssh -o BatchMode=yes`, and appends the token as `http.extraHeader` (GitHub: `Authorization: Bearer`; everything else: Basic with an empty user) [git].

`runGit` uses `execFile` with a deadline (default five minutes) and a 16 MB buffer; a missing `git` becomes `GitError('missing-git')`, a killed process `GitError('timeout')`, and stderr is redacted and classified (`auth`, `ref`, `not-found`, else `failed`) [git]. The library-facing helpers are `cloneNoCheckout`, `listFiles`, `showFile`, `checkoutPath`, `unshallow`, `unsavedWork`, `fastForward` and `pull` [git].

`src/host/run.ts` is the equivalent for `git`/`python` in the Team Wiki service: same no-prompt environment, plus UTF-8 Python settings, a shorter 120 s default timeout, and a Basic `x-access-token` header for pushes [run].

## Reading bundles

`src/host/bundle.ts` defines the reader [bundle]:

- A folder is a **bundle root** when it holds a `SCHEMA.md`, or an `index.md` whose frontmatter declares `okf_version`.
- `index.md` and `log.md` are reserved (no `type` needed); `.git`, `node_modules`, `.obsidian`, `.github`, `.claude` and `.vscode` are never walked.
- `summarize()` lists up to `PAGE_CAP = 5000` markdown pages, counts pages per `type` and per `lifecycle`/`status`, records pages without a `type`, and reads `okf_version` from the root `index.md`.
- `resolvePage()` refuses anything that is not a `.md` inside the bundle or that contains `..`/`.git`; `readPage()` refuses files over `PAGE_BYTES_CAP = 2 MB`.

`src/host/frontmatter.ts` is a deliberately small reader: scalar keys, inline lists, block lists of scalars, and the odd inline map. Nested mappings (`sources`, `generated`, `verified`) are recorded as `undefined` rather than guessed, and a page whose frontmatter it cannot read still appears without metadata [frontmatter]. That is why the preview shows flat fields only; review state is computed by a separate reader in `host/review.ts` (see [Review state and auto-merge](/concepts/review-and-auto-merge.md)).

## The prompt context

`sessionContext(library, cache, options)` returns a `PromptContext` named `okf:session-bundle` at order 130, behind the in-tree contexts at 110–120 [knowledge]. Its `text` callback:

- finds the bundle by the agent's session id, falling back to the session header's `parentSession`, so a subagent inherits its parent's bundle [knowledge];
- describes the origin (a local mirror of a URL at a commit / a clone being edited / the project wiki in a maintenance workspace / a plain folder) and whether this session may edit it — `editableHere` requires `entry.workspace` and a session `cwd` that contains the bundle root [knowledge];
- appends the edit rules (generic OKF rules, or the Team Wiki's stricter team rules), and the `index.md` **body** (frontmatter stripped) truncated to `indexChars` [knowledge].

The context text callback must be synchronous, so `IndexCache` keeps each bundle's `index.md` in memory and re-reads it only when its modification time moves — which is how the catalog follows an agent's own edits on the next turn [knowledge].

## Related pages

- [Architecture](/concepts/architecture.md) — where these modules sit.
- [Team Wiki service](/concepts/team-wiki-service.md) — the routes from `/api/okf/team` onward.
- [Data model](/concepts/data-model.md) — `LibraryEntry`, `BundleSummary`, `PageSummary` and the library file.
