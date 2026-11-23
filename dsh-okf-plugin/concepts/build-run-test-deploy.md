---
type: Concept
title: Build, Run, Test and Deploy
description: The exact commands to install, type-check, build, smoke-test, harness-check and preview the plugin, and how it is installed and updated in DSH.
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
  - resource: https://github.com/yishao1224-design/dsh-okf-plugin/blob/ec5a774e593070c87dcca36c96a5012481f280c0/package.json
    id: manifest
    title: package.json
  - resource: /sources/2026-11-20-build-script.md
    id: build
    title: build.mjs capture
  - resource: /sources/2026-11-20-tsconfig.md
    id: tsconfig
    title: tsconfig.json capture
  - resource: https://github.com/yishao1224-design/dsh-okf-plugin/blob/95b0615aefbbd4d1e16040efdd37d1e1884e5ed0/scripts/smoke.mjs
    id: smoke
    title: scripts/smoke.mjs
  - resource: https://github.com/yishao1224-design/dsh-okf-plugin/blob/95b0615aefbbd4d1e16040efdd37d1e1884e5ed0/scripts/team-smoke.mjs
    id: team-smoke
    title: scripts/team-smoke.mjs
  - resource: https://github.com/yishao1224-design/dsh-okf-plugin/blob/ec5a774e593070c87dcca36c96a5012481f280c0/scripts/hub-smoke.mjs
    id: hub-smoke
    title: scripts/hub-smoke.mjs
  - resource: https://github.com/yishao1224-design/dsh-okf-plugin/blob/ec5a774e593070c87dcca36c96a5012481f280c0/scripts/ado-smoke.mjs
    id: ado-smoke
    title: scripts/ado-smoke.mjs
  - resource: https://github.com/yishao1224-design/dsh-okf-plugin/blob/95b0615aefbbd4d1e16040efdd37d1e1884e5ed0/scripts/harness-check.mjs
    id: harness-check
    title: scripts/harness-check.mjs
  - resource: https://github.com/yishao1224-design/dsh-okf-plugin/blob/95b0615aefbbd4d1e16040efdd37d1e1884e5ed0/scripts/preview-server.mjs
    id: preview
    title: scripts/preview-server.mjs
generated:
  by: dsh/agent
  at: 2026-11-21
---

# Build, Run, Test and Deploy

## Prerequisites

| Tool | Needed for |
|---|---|
| Node.js | building and running the scripts; the CI workflows pin Node 22, and `package.json` declares no `engines` field |
| npm | `npm install` / the three scripts in `package.json` |
| `git` on `PATH` (or `gitPath` configured) | every fetch, update and Team Wiki operation |
| Python 3 (or `team.python`) | the hub's `scripts/okf.py`; `python` on Windows, `python3` elsewhere |
| a DSH source checkout | only `scripts/harness-check.mjs`; the package otherwise needs none |

## Build

```sh
npm install
npm run typecheck     # tsc --noEmit, src/ only
npm run build         # node build.mjs -> lib/index.js, lib/check-pr.js, lib/client.js
```

`package.json` defines exactly those three scripts [manifest]. `npm run typecheck` uses the strict `tsconfig.json` (`strict`, `noUncheckedIndexedAccess`, `exactOptionalPropertyTypes`, `noUnusedLocals/Parameters`, `noEmit`) over `src` only [tsconfig].

`build.mjs` runs three `tsdown` builds and then asserts the boundaries [build]:

- Host: `src/index.ts` → `lib/index.js`, ESM/Node/es2022, and **no non-`node:` import** may survive;
- checker: `src/check-pr.ts` → `lib/check-pr.js`;
- browser: `src/client/index.ts` → the `exports["./client"]` path, CJS/browser, React and React DOM never bundled, wrapped in `window.__ModuleLoader__.load({ id: '<package name>', factory })`, and forbidden from carrying `.module.css` or `lightningcss`.

`lib/` is committed on purpose: a DSH profile installs the package as it is, with no build toolchain, so a source change must be rebuilt before it is installed or tested [readme].

## Test

```sh
npm run smoke
```

That is `node scripts/smoke.mjs && node scripts/team-smoke.mjs && node scripts/hub-smoke.mjs && node scripts/ado-smoke.mjs` [manifest].

| Script | Needs | Covers |
|---|---|---|
| `scripts/smoke.mjs` | `git` only — no Harness, no browser, no network | Two throwaway `file://` remotes; the built Host routes (fetch, auto-detect, many-bundles, duplicate id, refused URL/ref/path, preview, traversal refusal, update clean/dirty/forced, add-local, remove); the prompt context incl. subagent inheritance; that no tools are registered; the built browser artifact rendered with `react-dom/server` and the markdown safety rules [smoke][readme] |
| `scripts/team-smoke.mjs` | a knowledgeAgent checkout (`WIKI_FIXTURE`, else `../knowledgeAgent`) and Python | A bare repo behind a GitHub URL via `url.<file>.insteadOf` plus a stand-in GitHub API: connect, review counts, Preview, Maintain and reuse, team rules in the prompt, Open pull request (eligible, then a reviewed page reset, then a lint failure that commits nothing), the PR list, `lib/check-pr.js`, New project wiki, merge-closes-workspace [team-smoke][readme] |
| `scripts/hub-smoke.mjs` | the same fixture | The hub layout on GitHub Enterprise (`github.acme.com`, API under `/api/v3`): registry-driven list, eligible page, `published/` needing a person, New project wiki adding a registry row, the first Sync from code reading the code through, a setup pull request opened later from its pending card, Publish, and `check-pr.js` mapping each case [hub-smoke][readme] |
| `scripts/ado-smoke.mjs` | the same fixture plus a fake Azure DevOps API | The pull request side on Azure Repos (`https://dev.azure.com/acme/Wiki/_git/team-wiki`): connect, Maintain, an eligible pull request tagged and auto-complete on (squash), pull requests listed from the source branch with the latest status per context, a reviewed page edited (tag and auto-complete off, title kept), Mark reviewed with the PAT's account, and a completed pull request closing the workspace and moving the base clone; every call is checked for the PAT and `api-version=7.1` [ado-smoke][readme] |

Two checks are outside `npm run smoke`:

```sh
# real DSH SystemPrompt + ToolRuntime, provided by sibling plugins (needs the checkout's deps)
DSH_ROOT=D:/project/dsh node --import file:///D:/project/dsh/node_modules/tsx/dist/esm/index.mjs scripts/harness-check.mjs
```

It asserts the module applies under strict injection, that `assemble({ agent })` carries the bundle for the right session only, that no `okf_*` tool reaches the real registry, and that disposal removes the context [harness-check].

```sh
node scripts/preview-server.mjs --port 4317 [--library <dir>] [--github-api <url>]
```

A development page at `http://127.0.0.1:4317/` that applies the built Host half to a stub context, serves its routes for real (real git, real library folder), and executes the built browser artifact against a stand-in shell. `?panel=okf-team-wiki` opens the Team Wiki page, `?dark` switches theme, `?lang=zh` switches language; with `DSH_ROOT` set it loads DSH's own `design-platform.css`. It has no authentication and binds to `127.0.0.1` [preview]. `.claude/launch.json` is a ready-made launch configuration for it (`okf-preview`, port 4317, `DSH_ROOT=D:/project/dsh`).

## Run (inside DSH)

There is no standalone server. Install the bundle into the running Harness's profile:

```text
plugin_manager action=install_bundle target=file:F:\project\dsh_plugins\dsh-okf-plugin
```

This needs Full access or approval because Host code runs in the Harness process. Then **restart `dsh web`** — Host code loads only at startup. The browser half hot-loads, which is why a freshly installed plugin shows the OKF chip but the dialog reports that the Host does not answer until the restart [readme].

`cordis.patch.yml` inserts the single row `id: okf`, `name: '@local/dsh-okf'`; optional settings go under `config:` in the same row, and a patch replaces the whole config block [readme].

## Deploy

"Deploy" for this package means: rebuild, reinstall, restart. The artifacts are the committed `lib/*.js` files plus `templates/hub`, `examples/*.yml`, `cordis.patch.yml` and `README.md` — the `files` list in `package.json` [manifest]. Nothing else is published.

For the **hub repository** (a different deployment), three example workflows have to be enabled by hand in the hub, because `Set up the hub` writes them under `.github/wiki/` so the first push needs no `workflow` scope [readme]:

1. on **GitHub**, move `.github/wiki/workflows/wiki-auto-merge.yml` to `.github/workflows/`, enable "Allow auto-merge" in the repository settings, and require the `wiki-check` job in branch protection;
2. on **Azure Repos**, add `.github/wiki/workflows/wiki-auto-complete-ado.yml` as a pipeline and make it a required Build validation policy on the base branch — the Host turns Azure DevOps auto-complete on itself, and this pipeline re-checks the change and turns it off again when the change needs a person; give the project's Build Service account "Contribute to pull requests" so it can;
3. move `.github/wiki/workflows/wiki-publish-to-ado.yml` to `.github/workflows/`, after setting `vars.ADO_WIKI_REPO`, optionally `vars.ADO_WIKI_BRANCH` (default `wikiMaster`) and `secrets.ADO_WIKI_PAT` (Code read & write on the wiki repository).

Both guards take the checker and the linter **from the base branch**, so a pull request cannot weaken the rules that judge it. They also expect `.github/wiki/check-pr.js`, which `Set up the hub` copies from `lib/check-pr.js` [readme].

## Running the toolchain by hand

Inside a hub checkout, the commands the plugin runs (and any person can run) are [okf-py][readme]:

```sh
python scripts/okf.py links --bundle <folder>
python scripts/okf.py index --bundle <folder>
python scripts/okf.py lint  --bundle <folder>
python scripts/okf.py log "<message>" --bundle <folder>
python scripts/okf.py stats --bundle <folder>
python scripts/okf.py drift --bundle <folder>
node .github/wiki/check-pr.js <base-ref> [head-ref] [wiki-root]   # the CI verdict, locally
```

## Related pages

- [The OKF toolchain](/concepts/okf-toolchain.md) — what each subcommand enforces.
- [Conventions, pitfalls and open questions](/concepts/conventions-and-pitfalls.md) — failure modes to expect after install.
- [Architecture](/concepts/architecture.md) — what the three artifacts are for.
