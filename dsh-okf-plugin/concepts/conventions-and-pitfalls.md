---
type: Concept
title: Conventions, Pitfalls and Open Questions
description: The coding and workflow conventions this repository follows, the traps a maintainer should know, and what the authors themselves flag as unverified.
status: draft
lifecycle: draft
confidence: medium
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
  - resource: /sources/2026-11-20-tsconfig.md
    id: tsconfig
    title: tsconfig.json capture
  - resource: /sources/2026-11-20-okf-py-toolchain.md
    id: okf-py
    title: okf.py key regions
  - resource: https://github.com/yishao1224-design/dsh-okf-plugin/blob/95b0615aefbbd4d1e16040efdd37d1e1884e5ed0/src/host/library.ts
    id: library
    title: src/host/library.ts
  - resource: https://github.com/yishao1224-design/dsh-okf-plugin/blob/95b0615aefbbd4d1e16040efdd37d1e1884e5ed0/src/host/review.ts
    id: review
    title: src/host/review.ts
  - resource: https://github.com/yishao1224-design/dsh-okf-plugin/blob/95b0615aefbbd4d1e16040efdd37d1e1884e5ed0/src/host/team.ts
    id: team
    title: src/host/team.ts
  - resource: https://github.com/yishao1224-design/dsh-okf-plugin/blob/95b0615aefbbd4d1e16040efdd37d1e1884e5ed0/src/host/frontmatter.ts
    id: frontmatter
    title: src/host/frontmatter.ts
  - resource: https://github.com/yishao1224-design/dsh-okf-plugin/blob/95b0615aefbbd4d1e16040efdd37d1e1884e5ed0/src/host/git.ts
    id: git
    title: src/host/git.ts
  - resource: https://github.com/yishao1224-design/dsh-okf-plugin/blob/95b0615aefbbd4d1e16040efdd37d1e1884e5ed0/src/client/team.tsx
    id: team-tsx
    title: src/client/team.tsx
  - resource: https://github.com/yishao1224-design/dsh-okf-plugin/blob/95b0615aefbbd4d1e16040efdd37d1e1884e5ed0/src/host/knowledge.ts
    id: knowledge
    title: src/host/knowledge.ts
generated:
  by: dsh/agent
  at: 2026-11-20
---

# Conventions, Pitfalls and Open Questions

## Conventions the code follows

**Standalone bundle.** Nothing is imported from or written into a DSH checkout. The Harness contract shapes the plugin uses are vendored in `src/types/contract.ts`, with a header listing the DSH files they were read from; a Harness upgrade that moves a shape is supposed to touch only that file [readme].

**The Host has no dependencies.** `lib/index.js` runs from the installed folder with no `node_modules`, so it may import only `node:` built-ins — `build.mjs` fails the build otherwise [build]. That constraint explains several hand-rolled pieces: the frontmatter reader, the GitHub client, the frontmatter text surgery in `review.ts`, and the `execFile` wrapper.

**One file per concern, exported interfaces, JSDoc everywhere.** Every exported function and interface carries a doc comment with `@param` / `@returns` / `@throws`; the TypeScript config is strict and enables `exactOptionalPropertyTypes` [tsconfig]. That last flag is why optional fields are almost always written as `...(value === undefined ? {} : { value })` rather than `{ value }` — an explicit `undefined` would not type-check.

**Immutable records.** Wire records and domain records use `readonly` fields; mutations build a new object. Functions that produce data (`judge`, `suggestPr`, `renderBundle`, `parseRegistry`, `reviewState`, `stampAgentEdit`) are pure and shared between the Host, the CI checker and the tests.

**One queue per store, whole-file writes.** `library.json` and `team.json` each have a promise chain that serializes mutations, and a `save()` that writes a temp file and renames it over the target, retrying `EPERM`/`EBUSY`/`EACCES` up to five times because Windows refuses a rename over an open file [library][team].

**Secrets never travel in argv or files.** A token is read from the environment and passed as `http.extraHeader` through `GIT_CONFIG_COUNT`/`GIT_CONFIG_KEY_n`/`GIT_CONFIG_VALUE_n`; output is redacted before it reaches the browser, and a URL that embeds a password is refused outright [readme]. Git never prompts (`GIT_TERMINAL_PROMPT=0`, null askpass, `GCM_INTERACTIVE=never`).

**Path safety.** `cleanRepoPath` refuses `..` and `.git` in a bundle subfolder; `resolvePage` refuses anything that leaves the bundle or is not markdown; `contains()` compares canonical paths (case-insensitively on Windows) before a workspace is claimed to hold a bundle [library].

**The browser never uses `innerHTML`.** Markdown becomes React elements; raw HTML renders as text; links are limited to `http(s)`/`mailto`; external links get `rel="noopener noreferrer"`; images are shown as links because the preview loads no remote content [readme].

**Inline styles only.** One style table, every colour a `--dsw-*` token, no stylesheet toolchain — the build refuses CSS artifacts [build].

**Chinese is the copy of record.** The `okf` locale namespace takes the Simplified Chinese dictionary as the key-set source of truth, with English kept alongside [readme].

**The team rules are prompt-level.** A maintenance session's context tells the agent to write unreviewed drafts, never write `verified` entries or claim a person as the reviewer, never edit an existing capture, never change `SCHEMA.md` unless asked, never touch `published/` or `AGENTS.md`, treat `.code/` as read-only, and not to commit [readme][knowledge].

## Pitfalls

- **A source change without `npm run build` changes nothing.** `lib/` is committed and the smoke tests import the built artifacts, not `src/` [readme].
- **Restart `dsh web` after installing or rebuilding the Host half.** The browser half hot-loads; the dialog showing "the Host does not answer" is the expected symptom until the restart [readme].
- **A config patch replaces the block.** Setting one key drops every other key that was there [readme].
- **Two different frontmatter readers.** The preview's reader handles flat keys and scalar lists and drops nested mappings; `review.ts` parses `generated`/`verified` from text in flow or block style [frontmatter][review]. Anything relying on nested `sources` data must go through the review/codesync readers, not `BundleSummary`.
- **Review dates are compared at day granularity.** `notBefore` compares the first ten characters, so a page rewritten on the same day as the human verification still counts as reviewed [review].
- **Git failure classes come from English stderr patterns.** `classify()` matches text such as "authentication failed" or "repository not found"; a localized git build would fall back to `failed` [git].
- **The auto-merge label is not the decision.** CI re-runs the checker and linter from the base branch; an eligible verdict without a label change simply never reaches the workflow [readme].
- **A new page's first pull request can be ineligible for surprising reasons.** A page that already claims a human review, a deleted page, or any edit under `published/` all force a person [readme].
- **`publish` skips non-markdown files silently.** Only the listed reasons appear in `skipped`; `not markdown` is filtered out before reporting [readme].
- **A deleted workspace silently ends a round.** `openRecordOf` marks a record `closed` when its `.git` is gone, and the next Maintain starts fresh [team].
- **Changing the hub closes workspaces.** `connect()` marks the previous repository's open workspaces `closed` in the list; their folders stay on disk [team].
- **The PR list is cached for 60 s**, and `pulls(force)` is the way to bypass it [team].
- **Azure DevOps hubs are second-class.** Maintain, New project wiki and pushing work; pull requests are neither listed nor opened, and the page says so [readme].
- **The GitHub Enterprise API path is derived from the host** (`https://<host>/api/v3`), so setting `team.githubHost` without `team.githubApi` is usually enough — but a proxy or a non-standard install needs both [readme].
- **`scripts/okf.py` must exist in the hub** for a PR to be linted and logged; without it the branch is still pushed but the PR is marked as needing a person [team].
- **`okf.py init` writes outside the new folder** (it re-syncs skill copies), so New project wiki reverts everything outside the wiki folder — and swallows a revert failure [team].
- **The `.code/` checkout is excluded through `.git/info/exclude`,** not `.gitignore`, so it never appears as a change; a reused clone is only reused when its `remote.origin.url` still matches [team].
- **New project wiki's client fallback is `root ?? 'okf'`** while the Host's default `team.root` is `''` (the repository root) [team-tsx]. With the current types `root` is always a string, so this is latent, but a future `undefined` would silently put new wikis under `okf/`.

## Open questions

These are the authors' own "not verified yet" items plus what this survey found [readme]:

1. **Azure DevOps wiki link resolution.** Published file names follow Microsoft's title rules, but links to names containing an encoded character (for example `%2D`) have never been opened in a real wiki.
2. **The Team Wiki page in the live Harness.** Only a stand-in GitHub API has been used; opening a pull request, labels, check runs and `refs/pull/<n>/head` fetches are untested live, and `layout.selectPanel(null)` before `openSession` is untested against the real shell.
3. **Editing in place in the live Harness.** The workspace services have worked from an out-of-tree bundle, but pushing from an edited bundle has not been tried.
4. **Reads under the library folder.** The prompt tells the agent to read pages under `~/.dsh/okf/`; whether a session's sandbox or permission profile allows that is unconfirmed.
5. **The chip's placement.** DSH does not render the session-header utilities in a blank session, and the composer row needs a session id — so in a brand-new "New session" view neither button can appear.
6. **Azure DevOps against a live organization.** URL parsing and the Basic-PAT header are covered, but no real ADO clone was run.
7. **No CI in this repository.** There is no `.github/workflows` here: type-check, build and the three smoke scripts are manual, and nothing checks that the committed `lib/` matches `src/`. The two workflow files under `examples/` are for *hub* repositories, not for this one.
8. **The vendored contract** was read from DSH 0.2.1-alpha.1; any Harness change to those shapes must be reflected in `src/types/contract.ts`.
9. **What counts as a "substantial" change** — the rule that makes an edited page a draft again — is left to the agent's instructions; only the review reset and the eligibility judge are mechanical.
10. **Cost of the code checkout.** Initialize/Sync clone the code repository with `--filter=blob:none`, which keeps the full history; for a very large repository that is still a sizeable download per maintenance folder.

## Related pages

- [Review state and auto-merge](/concepts/review-and-auto-merge.md) — the mechanical rules behind several pitfalls.
- [Build, run, test, deploy](/concepts/build-run-test-deploy.md) — the commands to reproduce or check any of this.
- [Team Wiki service](/concepts/team-wiki-service.md) — the code most of the runtime pitfalls come from.
