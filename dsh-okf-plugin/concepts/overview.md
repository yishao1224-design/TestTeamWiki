---
type: Concept
title: dsh-okf Plugin — Overview
description: What the dsh-okf DeepSeek Harness plugin is for, who uses it, and the main flows it supports.
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
  - resource: https://github.com/yishao1224-design/dsh-okf-plugin/blob/95b0615aefbbd4d1e16040efdd37d1e1884e5ed0/package.json
    id: manifest
    title: package.json
  - resource: https://github.com/yishao1224-design/dsh-okf-plugin/blob/95b0615aefbbd4d1e16040efdd37d1e1884e5ed0/cordis.patch.yml
    id: patch
    title: cordis.patch.yml
  - resource: https://github.com/yishao1224-design/dsh-okf-plugin/blob/95b0615aefbbd4d1e16040efdd37d1e1884e5ed0/src/index.ts
    id: host-entry
    title: src/index.ts (Host entry)
  - resource: https://github.com/yishao1224-design/dsh-okf-plugin/blob/95b0615aefbbd4d1e16040efdd37d1e1884e5ed0/src/client/index.ts
    id: client-entry
    title: src/client/index.ts (browser entry)
generated:
  by: dsh/agent
  at: 2026-11-20
---

# dsh-okf Plugin — Overview

## What it is

`@local/dsh-okf` is a standalone **DeepSeek Harness (DSH) plugin bundle** that makes [OKF](https://github.com/GoogleCloudPlatform/open-knowledge-format) v0.2 knowledge bundles usable from inside the Harness. An OKF bundle is a folder of markdown pages with YAML frontmatter, a `SCHEMA.md` of conventions, an `index.md` catalog and a `log.md` change history; the plugin fetches such folders from git, previews them, hands their catalog to a chat session, and runs a team's project-wiki workflow on top of them [readme].

It is packaged the way other out-of-tree DSH bundles are: `package.json` declares `dsh.bundle.patch` pointing at a one-row `cordis.patch.yml`, and `dsh.client` declares a `web` platform client artifact with three injected client packages [manifest][patch]. `src/client/index.ts` is the browser entry the Harness mounts; `src/index.ts` is the Host entry the Loader imports [host-entry][client-entry].

## Who uses it

Three audiences share the same surfaces:

- **A DSH user with knowledge bundles.** They open the OKF chip in the composer or session header, fetch a bundle from GitHub / Azure DevOps / any git host (or register a local folder), preview it, and attach it to a session. The agent in that session then receives the bundle's catalog every turn and reads pages with the Harness's own file tools [readme].
- **A team running a wiki hub.** The **Team Wiki** sidebar page lists every project wiki in one hub repository with review counts and open pull requests, and starts the work: New project wiki, Maintain (a session in the wiki's own clone), Initialize / Sync from code, Publish, Open pull request [readme].
- **The agent in a maintenance session.** Its prompt context carries the team's editing rules — write unreviewed drafts, never fake a human review, never edit an existing capture, never commit — and it edits the wiki with `edit`/`write` in that workspace [readme].

## The jobs it does

1. **Fetch and update** a bundle into a local library, with sparse, blobless git clones so a large monorepo costs little more than the bundle [readme].
2. **Preview**: stats, a filterable catalog, a page reader, and in-place navigation between pages [readme].
3. **Attach a bundle to a session** through a system-prompt context named `okf:session-bundle`; the plugin registers **no tools** on purpose [readme][host-entry].
4. **Run the team wiki**: registry-driven project list, review counts, Maintain / New project wiki / Publish / Open pull request, with an auto-merge rule that CI re-checks before a merge [readme].
5. **Bootstrap a wiki from code** (Initialize) and keep it in step (Sync from code), by checking the code repository out read-only under `.code/<folder>/` and prefilling the session with a read-through or update task [readme].

## Main flows at a glance

| Flow | Starts at | Ends at |
|---|---|---|
| Fetch a bundle | `POST /api/okf/fetch` from the Add form | a `LibraryEntry` plus `<library>/repos/<id>/` |
| Attach to a session | the dialog's session selector → `POST /api/okf/session` | `library.json` selection; next turn's prompt context |
| Edit where it is | Edit as workspace → `POST /api/okf/edit-workspace` | the library checkout becomes a DSH workspace, opened with a session |
| Maintain a wiki | Team Wiki → Maintain → `POST /api/okf/team/maintain` | a clone in `<library>/team/work/<slug>` on a `wiki/…` branch, opened as a session |
| Open a pull request | the maintenance session's Open pull request button | stamps, `links`/`index`/`lint`/`log`, commit, push, GitHub PR, label |
| Auto-merge | CI workflow in the hub | `check-pr.js` + `okf.py lint` from the base branch, then GitHub auto-merge |
| Publish | Publish button → `POST /api/okf/team/publish` | `published/Projects/…` in the same PR, later mirrored to the Azure DevOps wiki |

The exact routes are listed in [Host routes and the bundle library](/concepts/host-routes-and-library.md); the Team Wiki flows in [Team Wiki service](/concepts/team-wiki-service.md).

## Shape of the system

The plugin is two halves in one package. The **Host half** runs inside the `dsh` process: it owns the library on disk, shells out to `git` and `python`, holds the Team Wiki state, and registers exact Fetch routes under `/api/okf` plus one prompt context [host-entry]. The **browser half** is a lazy CommonJS factory registered as `window.__ModuleLoader__.load({ id: '@local/dsh-okf', … })`; it renders the chip, the dialog, the Team Wiki page and the Open pull request dialog, and talks to the Host over document-relative fetches [client-entry].

Nothing is imported from, or written into, a DSH source checkout: the contract shapes the plugin needs are vendored in `src/types/contract.ts` [readme]. See [Architecture](/concepts/architecture.md) for the component map, [Data model](/concepts/data-model.md) for what is stored where, and [Build, run, test, deploy](/concepts/build-run-test-deploy.md) for the commands.

## What it deliberately is not

- **Not a tool server.** The prompt context tells the model to start from `index.md`, choose pages itself, and read/grep/glob with the Harness's tools; smoke tests assert that no `okf_*` tool reaches the tool registry [readme].
- **Not a second copy of the knowledge.** A bundle is edited where it already is; the library records a path or a checkout, never a duplicate [readme].
- **Not a merger of last resort.** The plugin only sets a `wiki-auto-merge` label; the hub's CI recomputes the verdict with the base branch's checker and linter before enabling auto-merge [readme][ci-auto-merge].

## Related pages

- [Architecture](/concepts/architecture.md) — components, how they talk, where data lives.
- [Review state and auto-merge](/concepts/review-and-auto-merge.md) — the rule that decides what a person must see.
- [Conventions, pitfalls and open questions](/concepts/conventions-and-pitfalls.md) — what the code expects and what is still unverified.
