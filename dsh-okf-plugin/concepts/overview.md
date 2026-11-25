---
type: Concept
title: dsh-okf Plugin — Overview
description: What the dsh-okf DeepSeek Harness plugin is for, who uses it, and the main flows it supports.
status: draft
confidence: high
created: 2026-11-20
updated: 2026-11-25
tags: []
sources:
  - resource: https://github.com/yishao1224-design/dsh-okf-plugin/blob/da10648a182b38b410d22f952d8e1d973d8825f0/README.md
    id: readme
    title: README.md
  - resource: https://github.com/yishao1224-design/dsh-okf-plugin/blob/ec5a774e593070c87dcca36c96a5012481f280c0/package.json
    id: manifest
    title: package.json
  - resource: https://github.com/yishao1224-design/dsh-okf-plugin/blob/ec5a774e593070c87dcca36c96a5012481f280c0/cordis.patch.yml
    id: patch
    title: cordis.patch.yml
  - resource: https://github.com/yishao1224-design/dsh-okf-plugin/blob/b71e34fb1c76d88af680440531fda3c5a9c75c34/src/index.ts
    id: host-entry
    title: src/index.ts (Host entry)
  - resource: https://github.com/yishao1224-design/dsh-okf-plugin/blob/ec5a774e593070c87dcca36c96a5012481f280c0/src/client/index.ts
    id: client-entry
    title: src/client/index.ts (browser entry)
  - resource: https://github.com/yishao1224-design/dsh-okf-plugin/blob/ec5a774e593070c87dcca36c96a5012481f280c0/examples/wiki-auto-merge.yml
    id: ci-auto-merge
    title: examples/wiki-auto-merge.yml
  - resource: https://github.com/yishao1224-design/dsh-okf-plugin/blob/ec5a774e593070c87dcca36c96a5012481f280c0/examples/wiki-auto-complete-ado.yml
    id: ci-ado-auto-complete
    title: examples/wiki-auto-complete-ado.yml
generated:
  by: dsh/agent
  at: 2026-11-25
---

# dsh-okf Plugin — Overview

## What it is

`@local/dsh-okf` is a standalone **DeepSeek Harness (DSH) plugin bundle** that makes [OKF](https://github.com/GoogleCloudPlatform/open-knowledge-format) v0.2 knowledge bundles usable from inside the Harness. An OKF bundle is a folder of markdown pages with YAML frontmatter, a `SCHEMA.md` of conventions, an `index.md` catalog and a `log.md` change history; the plugin fetches such folders from git, previews them, hands their catalog to a chat session, and runs a team's project-wiki workflow on top of them [readme].

It is packaged the way other out-of-tree DSH bundles are: `package.json` declares `dsh.bundle.patch` pointing at a one-row `cordis.patch.yml`, and `dsh.client` declares a `web` platform client artifact with three injected client packages [manifest][patch]. `src/client/index.ts` is the browser entry the Harness mounts; `src/index.ts` is the Host entry the Loader imports [host-entry][client-entry].

## Who uses it

Three audiences share the same surfaces:

- **A DSH user with knowledge bundles.** They open the OKF chip in the composer or session header, fetch a bundle from GitHub / Azure DevOps / any git host (or register a local folder), preview it, and attach it to a session. The agent in that session then receives the bundle's catalog every turn and reads pages with the Harness's own file tools [readme].
- **A team running a wiki hub.** The **Team Wiki** sidebar page lists every project wiki in one hub repository with review counts and open pull requests, and starts the work: New project wiki, Maintain (a session in the wiki's own clone), Sync from code (the one code button — the first round initializes the wiki, later rounds update it), Publish, Open pull request [readme].
- **The agent in a maintenance session.** Its prompt context carries the team's editing rules — write unreviewed drafts, never fake a human review, never edit an existing capture, never commit — and it edits the wiki with `edit`/`write` in that workspace [readme].

## The jobs it does

1. **Fetch and update** a bundle into a local library, with sparse, blobless git clones so a large monorepo costs little more than the bundle [readme].
2. **Preview**: stats, a filterable catalog, a page reader, and in-place navigation between pages [readme].
3. **Attach a bundle to a session** through a system-prompt context named `okf:session-bundle`; the plugin registers no bundle-reading tools on purpose — its only tools are the two Team Wiki ones below [readme][host-entry].
4. **Run the team wiki**: registry-driven project list, review counts and a Needs-review queue read off each page's OKF `status` (the five stages are derived, never stored — see [Page stages](/concepts/page-stages.md)), Maintain / New project wiki / Publish / Open pull request, with an auto-merge rule (Azure DevOps auto-complete for a hub there) that CI re-checks before a merge [readme].
5. **Bootstrap a wiki from code and keep it in step** with Sync from code: its first usable round checks the code repository out read-only under `.code/<folder>/` and prefills a read-through task, later rounds prefill the update task for the pages the changed files affect; the recorded code commit lives in the wiki's `log.md` [readme]. Each round also makes sure the hub has a current `scripts/okf.py`, adding or refreshing the plugin's copy when the hub's is missing or older [readme].
6. **Let a scheduled automation task do the upkeep.** The model gets `wiki_sync_from_code` and `wiki_open_pr`, so a DSH automation task in the project's maintenance workspace can check for new code, do the update task, and open or update the pull request without anyone at the page; a page someone reviewed still waits for a person [readme][host-entry].

## Main flows at a glance

| Flow | Starts at | Ends at |
|---|---|---|
| Fetch a bundle | `POST /api/okf/fetch` from the Add form | a `LibraryEntry` plus `<library>/repos/<id>/` |
| Attach to a session | the dialog's session selector → `POST /api/okf/session` | `library.json` selection; next turn's prompt context |
| Edit where it is | Edit as workspace → `POST /api/okf/edit-workspace` | the library checkout becomes a DSH workspace, opened with a session |
| Maintain a wiki | Team Wiki → Maintain → `POST /api/okf/team/maintain` | a clone in `<library>/team/work/<slug>` on a `wiki/…` branch, opened as a session |
| Open a pull request | the maintenance session's Open pull request button | stamps, `links`/`index`/`lint`/`log`, commit, push, a GitHub or Azure DevOps pull request, a label (a tag on Azure DevOps, which also gets auto-complete) |
| Auto-merge | the hub's CI — a GitHub workflow, or an Azure Pipelines build validation | `check-pr.js` + `okf.py lint` from the base branch, then GitHub auto-merge or Azure DevOps auto-complete |
| Publish | Publish button → `POST /api/okf/team/publish` | `published/Projects/…` in the same PR, later mirrored to the Azure DevOps wiki |
| Scheduled sync | an automation task's reminder → `wiki_sync_from_code` | the update task run in the maintenance session, then `wiki_open_pr` opens or updates the pull request |

The exact routes are listed in [Host routes and the bundle library](/concepts/host-routes-and-library.md); the Team Wiki flows in [Team Wiki service](/concepts/team-wiki-service.md).

## Shape of the system

The plugin is two halves in one package. The **Host half** runs inside the `dsh` process: it owns the library on disk, shells out to `git` and `python`, holds the Team Wiki state, and registers exact Fetch routes under `/api/okf` plus one prompt context and the two Team Wiki tools [host-entry]. The **browser half** is a lazy CommonJS factory registered as `window.__ModuleLoader__.load({ id: '@local/dsh-okf', … })`; it renders the chip, the dialog, the Team Wiki page and the Open pull request dialog, and talks to the Host over document-relative fetches [client-entry].

Nothing is imported from, or written into, a DSH source checkout: the contract shapes the plugin needs are vendored in `src/types/contract.ts` [readme]. See [Architecture](/concepts/architecture.md) for the component map, [Automation tasks and the Team Wiki tools](/concepts/automation-tasks.md) for the model tools, [Data model](/concepts/data-model.md) for what is stored where, and [Build, run, test, deploy](/concepts/build-run-test-deploy.md) for the commands.

## What it deliberately is not

- **Not a bundle search engine.** The prompt context tells the model to start from `index.md`, choose pages itself, and read/grep/glob with the Harness's tools; smoke tests assert that no `okf_*` tool reaches the tool registry, and the only tools the plugin adds are the two Team Wiki ones [readme].
- **Not an unattended merger.** `wiki_open_pr` opens or updates a pull request and never merges; a person — or, for a change whose pages are all unreviewed, the hub's own CI — decides [readme].
- **Not a second copy of the knowledge.** A bundle is edited where it already is; the library records a path or a checkout, never a duplicate [readme].
- **Not a merger of last resort.** On GitHub the plugin only sets a `wiki-auto-merge` label; on Azure DevOps it also turns auto-complete on or off. Either way the hub's CI recomputes the verdict with the base branch's checker and linter before a merge can complete [readme][ci-auto-merge][ci-ado-auto-complete].

## Related pages

- [Architecture](/concepts/architecture.md) — components, how they talk, where data lives.
- [Page stages](/concepts/page-stages.md) — the status/tag/folder model the review counts come from.
- [Review state and auto-merge](/concepts/review-and-auto-merge.md) — the rule that decides what a person must see.
- [Conventions, pitfalls and open questions](/concepts/conventions-and-pitfalls.md) — what the code expects and what is still unverified.
