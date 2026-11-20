---
type: Concept
title: Team Wiki Service
description: The team.ts Host service that lists a hub's project wikis, runs maintenance rounds, opens pull requests and bootstraps or syncs wikis from code.
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
  - resource: https://github.com/yishao1224-design/dsh-okf-plugin/blob/95b0615aefbbd4d1e16040efdd37d1e1884e5ed0/src/host/team.ts
    id: team
    title: src/host/team.ts
  - resource: https://github.com/yishao1224-design/dsh-okf-plugin/blob/95b0615aefbbd4d1e16040efdd37d1e1884e5ed0/src/host/registry.ts
    id: registry
    title: src/host/registry.ts
  - resource: https://github.com/yishao1224-design/dsh-okf-plugin/blob/95b0615aefbbd4d1e16040efdd37d1e1884e5ed0/src/host/github.ts
    id: github
    title: src/host/github.ts
  - resource: https://github.com/yishao1224-design/dsh-okf-plugin/blob/95b0615aefbbd4d1e16040efdd37d1e1884e5ed0/src/host/codesync.ts
    id: codesync
    title: src/host/codesync.ts
  - resource: https://github.com/yishao1224-design/dsh-okf-plugin/blob/95b0615aefbbd4d1e16040efdd37d1e1884e5ed0/src/host/summary.ts
    id: summary
    title: src/host/summary.ts
  - resource: https://github.com/yishao1224-design/dsh-okf-plugin/blob/95b0615aefbbd4d1e16040efdd37d1e1884e5ed0/src/host/run.ts
    id: run
    title: src/host/run.ts
  - resource: https://github.com/yishao1224-design/dsh-okf-plugin/blob/95b0615aefbbd4d1e16040efdd37d1e1884e5ed0/src/host/publish.ts
    id: publish
    title: src/host/publish.ts
generated:
  by: dsh/agent
  at: 2026-11-20
---

# Team Wiki Service

## Its job

`src/host/team.ts` is the Team Wiki: one hub repository whose project wikis are OKF bundles, listed in the hub's `AGENTS.md` registry (or found by their `SCHEMA.md` when there is no registry). The service answers what the page shows, and performs the work the page starts — Maintain, New project wiki, Open pull request, Initialize from code, Sync from code, Publish, Set up the hub [team][readme].

It is built by `openTeam(options)` in `src/index.ts` with the hub template paths (`templates/hub/`, `lib/check-pr.js`, and the two example workflows), the `python` executable, `team.okfScript`, the GitHub host/API, the auto-merge label, the agent actor and the commit author [readme].

## State on disk

`openTeam` keeps everything under `<libraryDir>/team` [team]:

```text
<libraryDir>/team/
  team.json        v:1, team, base, commit, empty, maintains[], code{}
  main/            a clone of the base branch, detached; read, never edited
  work/<slug>/     one maintenance clone per project wiki (reused per round)
```

`team.json` is written whole through the same temp-file-and-rename-with-retries pattern as `library.json`, and every mutating method runs through one `serial` queue [team]. `maintains[]` holds a `MaintainRecord` per round; `code{}` holds the last `HEAD` each project's code repository reported to Refresh, keyed by wiki folder [team].

## The methods

| Method | Behaviour |
|---|---|
| `status()` | Applies the `team.url` preset when nothing is connected; lists project wikis with review counts; returns connected config, base branch, commit, `empty`, active `maintains`, `hasToken` |
| `connect({url, ref, root})` | Parses the remote, clones into `main.<hex>.tmp` (blobless), moves it to `main/`, derives the base branch, records `team`, marks the old repository's open workspaces `closed`, clears caches |
| `refresh()` | Fetches the base branch into `main/`, runs `git ls-remote <repo> HEAD` for every registered code repository (no clone), sweeps old per-round folders, clears the PR cache |
| `wiki(folder)` | One project wiki: its `ProjectWiki` plus every page's `WikiPage` review state |
| `pulls(force?)` | Open GitHub PRs against the base branch, each fetched into `refs/okf-pr/<n>`, mapped to wikis, judged with `judge()`, with checks; 60 s memory cache |
| `create(request)` | New project wiki: slug the folder, refuse duplicates, clone on `wiki/new-<slug>-<hex>`, scaffold, add the registry row, open the setup PR |
| `maintain(folder)` | Reuse the open workspace, else a new round on `wiki/<slug>-<stamp>-<hex>`; registers/updates the library entry |
| `changes(id)` | The files a workspace holds against `origin/<base>`, reviewed/unreviewed, uncommitted count, a suggested PR title and summary |
| `openPr(id, {title, summary})` | Stamp, toolchain, commit, judge, push, open/update the PR, label |
| `attach(folder)` | Add the base-branch wiki to the library as a read-only local entry |
| `publish(folder)` | Render `host/publish.ts` output into `published/Projects/…` and open/update the PR |
| `setupHub()` | Push the hub skeleton as the first commit of an empty repository |
| `initialize(folder)` | Open (or reuse) the workspace, check the code out under `.code/<slug>/`, write the read-through task |
| `sync(folder)` | Answer `upToDate` when the code has not moved; else diff, find affected pages, set `code_commit`, write the sync task |

## Listing project wikis

`projectsIn(repo, team)` reads `AGENTS.md` with `parseRegistry`; when the table has rows, those rows are the project list, in file order, with `Repo`, `Summary` and `Owner` when present [team][registry]. Without a registry it falls back to `discoverWikis(repo, team.root)`: the wiki folder itself when it holds a `SCHEMA.md`, plus each folder directly under it that does — with `team.root = ''` that means top-level folders, skipping `NON_BUNDLE_DIRS` and bundle-internal names (`concepts`, `sources`, …) [team].

For each project the list computes `missing` (registered but no folder on the base branch), `published` (`published/Projects/<Project>/` or `<Project>.md` exists, plus the last commit date that touched it), `stats` (from `readWikiPages`: pages, reviewed, unreviewed, overdue) and `code.recorded` (the SCHEMA.md `code_commit`, else the commit its pages cite most) [team][codesync]. Review counts are cached per base commit (`statsCache`), so Refresh is the way to see a moved base [team].

`readWikiPages` walks the wiki, skipping `sources/` and `archive/`, skipping a nested folder that holds its own `SCHEMA.md` (a nested wiki is listed separately), and computes each page's `review`, `overdue` (`lifecycle: needs_review` or `stale_after` before today), reviewer and generated stamps [team].

## A maintenance round

`newWorkspace(data, folder, name, isNew)` builds the round [team]:

- folder: `work/<wikiSlug(folder with / → -)>` — one folder per project, reused round after round;
- branch: `wiki/new-<slug>-<hex>` for a new project, else `wiki/<slug>-<YYYYMMDD-HHmm>-<hex>`;
- if the folder already holds a clone of **this** hub, it fetches the base branch, stashes anything uncommitted (`git stash push --include-untracked`), switches to the new branch from `origin/<base>` and cleans; a clone of another hub is renamed aside, never deleted;
- otherwise it clones `--filter=blob:none --branch <base>` and switches to the new branch;
- the record gets `entryId: team-<slug>`, so a project has one folder, one DSH workspace and one library entry;
- `sweepOld` removes the per-round folders older layouts left behind, but only when they hold nothing uncommitted or unpushed [team][readme].

`maintain()` first calls `openRecordOf`, which treats a missing `.git` or a merged/closed pull request as the end of the round; otherwise it returns the open record, so pressing Maintain again reopens the same workspace instead of starting a second one [team].

`entryFor(record)` is the library entry a maintenance session holds: `root = <workspace>/<wiki>`, `origin: { kind: 'local' }`, `workspace` set, and `team: { maintainId, wiki, project }`. That `team` field is what `knowledge.ts` uses to apply the team editing rules and what the client uses to show the Open pull request button [team][readme].

## Open pull request

`openPr(id, {title, summary})` is the most consequential method. In order [team]:

0. **Reset stale stamps.** For every modified page inside the wiki whose text changed, if it was reviewed before *and* still claims the same review after, it is rewritten with `stampAgentEdit` — `status: draft`, `lifecycle: draft`, `updated`, a fresh `generated`, keeping `verified` as history. The reset list comes back in `stamped`.
1. **Run the repository's toolchain.** `links`, `index`, `lint`, then one `log` line per changed page (`Creation` / `Update` / `Removal` / `Ingestion`). A lint failure raises `lint-failed` with the linter's output and **nothing is committed or pushed**.
2. **Commit only what belongs to the wiki PR.** The scope is the wiki folder, the project's `published/Projects/<Project>/` pages and landing page, and `AGENTS.md` only for a new project. Everything else is left out and listed in `notes`. Commits use `user.name`/`user.email` from git when configured, else `team.commitAuthor`, and the message carries a `Wiki-Workspace: <id>` trailer.
3. **Judge the branch against the base** with `judge(record.wiki, files)`; a missing toolchain script or a missing merge base adds a reason, so the PR needs a person.
4. **Push and open.** The branch is pushed with the token; on GitHub the PR is opened or updated and labeled (`wiki-auto-merge` added when eligible, removed when not). Any other host leaves the branch pushed and says to open the PR by hand; without a token the same happens with a note.

The PR body (`prBody`) is a table of pages with their review state before and after, the auto-merge verdict or the reasons it needs a person, the review instructions, the lint output in a `<details>` block, the notes, and an HTML comment recording the workspace id and verdict [team].

## Initialize and Sync from code

`prepareCode(record)` puts the project's code at `.code/<slug>/` inside the maintenance workspace [team]:

- the URL comes from the registry's `Repo` column for that bundle, via `codeRemote` (which refuses a missing or non-URL repo and picks the GitHub provider when the host matches `team.githubHost`);
- `/.code/` is appended to the workspace clone's `.git/info/exclude`, so it can never enter a wiki PR;
- a reusable clone is fetched and detached at `FETCH_HEAD`; otherwise the code is cloned `--filter=blob:none --single-branch`.

`initialize(folder)` opens the round (falling back to the project's open setup workspace while the new wiki is still in its setup PR), calls `checkoutCode`, records `code_commit`/`code_synced` into the workspace's `SCHEMA.md`, and returns the read-through task: survey, read the main modules, write overview / architecture / per-module / data-model / build-run-test / conventions pages as unreviewed drafts citing `<repo>/blob/<commit>/<path>`, then links, index and lint [team][readme].

`sync(folder)` first answers without starting a round when `git ls-remote` shows the same commit the wiki records. Otherwise it opens a round, diffs `recorded..HEAD`, lists commits, computes `affectedPages` (pages whose frontmatter cites a changed path, with a path-boundary check), records the new commit, and builds the sync task prompt [team][codesync]. Without a commit the code clone actually has, it falls back to the full read-through [team].

## Set up the hub

`setupHub()` refuses when `git ls-remote --heads origin` is non-empty; otherwise it creates a temporary repository on the base branch, copies `templates/hub/` into it, copies the single files (`check-pr.js` → `.github/wiki/check-pr.js`, the two workflows → `.github/wiki/workflows/`), commits with the configured identity and pushes `HEAD:refs/heads/<base>` [team][readme]. The workflows deliberately sit outside `.github/workflows/` so the first push needs no `workflow` token scope [readme][hub-agents].

## Pull request listing

`pulls()` lists open PRs through `GitHub.openPulls` (one page of 100, newest first), fetches their heads into `refs/okf-pr/<n>`, and for each computes the merge base, the changed files, the wikis it touches (`wikiOfPath` maps `published/Projects/<Project>/…` and bundle folders to a project), and the verdict [team][github]. A PR that touches no wiki and was not opened here is not listed. After listing, any maintenance record whose PR is no longer open is marked `merged` or `closed`; when something merged, `main/` is re-synced so the page shows the new base [team].

The list is cached for `PULLS_TTL_MS = 60_000`; a 404 on a label removal is ignored, and the `statusChanged` flag tells the client to drop its cached wiki details [team].

## Dependencies and failure modes

| Dependency | Used for |
|---|---|
| `git` (`team.git` / `gitPath`) | clone, fetch, switch, stash, status, diff, show, log, ls-remote, merge-base, push |
| `python` (`team.python`) + the repo's `okfScript` | `init`, `index`, `links`, `lint`, `log` |
| GitHub REST (`team.githubHost` / `githubApi`) | pull requests, labels, check runs |
| A token in `DSH_OKF_GITHUB_TOKEN` / `DSH_OKF_GIT_TOKEN` / `DSH_OKF_AZURE_TOKEN` | push and PR calls |

Recurring failure classes surface as `LibraryError` with a code the client shows: `no-team`, `bad-url`, `bad-ref`, `unknown-wiki`, `exists`, `busy`, `reset-failed`, `no-repo`, `closed`, `nothing-changed`, `lint-failed`, `not-empty`. Azure DevOps hubs are supported for Maintain, New project wiki and pushing, but their pull requests are neither listed nor opened — `pulls()` returns a `note` of `azure` [team][readme].

## Related pages

- [Review state and auto-merge](/concepts/review-and-auto-merge.md) — the `judge()` rule this service calls at step 3.
- [Publishing to the Azure DevOps wiki](/concepts/publish-pipeline.md) — what `publish()` writes.
- [The OKF toolchain](/concepts/okf-toolchain.md) — the `okf.py` subcommands step 1 runs.
- [Data model](/concepts/data-model.md) — `MaintainRecord`, `ProjectWiki`, `PullView`.
