---
type: Concept
title: Azure DevOps Hubs and Pull Requests
description: How a hub in Azure Repos gets pull requests, tags, statuses and auto-complete — the shared PullHost shape, the AzureDevOps REST client and the pipeline that guards a merge.
status: draft
lifecycle: draft
confidence: medium
created: 2026-11-21
updated: 2026-11-21
tags: []
sources:
  - resource: https://github.com/yishao1224-design/dsh-okf-plugin/blob/ec5a774e593070c87dcca36c96a5012481f280c0/src/host/azure.ts
    id: azure
    title: src/host/azure.ts
  - resource: https://github.com/yishao1224-design/dsh-okf-plugin/blob/ec5a774e593070c87dcca36c96a5012481f280c0/src/host/team.ts
    id: team
    title: src/host/team.ts
  - resource: https://github.com/yishao1224-design/dsh-okf-plugin/blob/ec5a774e593070c87dcca36c96a5012481f280c0/src/host/github.ts
    id: github
    title: src/host/github.ts
  - resource: https://github.com/yishao1224-design/dsh-okf-plugin/blob/ec5a774e593070c87dcca36c96a5012481f280c0/src/host/git.ts
    id: git
    title: src/host/git.ts
  - resource: https://github.com/yishao1224-design/dsh-okf-plugin/blob/ec5a774e593070c87dcca36c96a5012481f280c0/src/check-pr.ts
    id: check-pr
    title: src/check-pr.ts
  - resource: https://github.com/yishao1224-design/dsh-okf-plugin/blob/ec5a774e593070c87dcca36c96a5012481f280c0/examples/wiki-auto-complete-ado.yml
    id: ci-ado
    title: examples/wiki-auto-complete-ado.yml
  - resource: https://github.com/yishao1224-design/dsh-okf-plugin/blob/ec5a774e593070c87dcca36c96a5012481f280c0/scripts/ado-smoke.mjs
    id: ado-smoke
    title: scripts/ado-smoke.mjs
  - resource: https://github.com/yishao1224-design/dsh-okf-plugin/blob/ec5a774e593070c87dcca36c96a5012481f280c0/README.md
    id: readme
    title: README.md
generated:
  by: dsh/agent
  at: 2026-11-21
---

# Azure DevOps Hubs and Pull Requests

## What changed

A hub repository does not have to be on GitHub for the pull-request features. `src/host/team.ts` reaches the hub through one small **`PullHost`** shape, implemented by `src/host/github.ts` for GitHub and by `src/host/azure.ts` for Azure Repos. Maintain, New project wiki and pushing already worked on Azure DevOps; listing pull requests, opening or updating them, tagging them, reading their checks and driving **auto-complete** now work there too [team][azure][readme].

The provider still comes from the remote URL, parsed by `parseRemote`, which recognizes `https://dev.azure.com/{org}/{project}/_git/{repo}`, `{org}.visualstudio.com/…` and `git@ssh.dev.azure.com:v3/{org}/{project}/{repo}` [git]. The Azure path exists only when the remote carries an organization and a project; otherwise the Host behaves as it does for any other git host.

## The PullHost shape

| Member | GitHub | Azure DevOps |
|---|---|---|
| `name`, `tokenEnv` | `GitHub`, `DSH_OKF_GITHUB_TOKEN` | `Azure DevOps`, `DSH_OKF_AZURE_TOKEN` |
| `login()` | the token's login | the token's account, usually the email |
| `openOrUpdate()` | create or update the PR for the branch | create or update the PR for the branch |
| `setLabel()` | a label | a tag (the UI name for the same thing) |
| `openPulls()` | one page of 100 open PRs | one page of 100 active PRs |
| `pullState()` | open / merged / closed | active → open, completed → merged, abandoned → closed |
| `checks()` | check runs on the head sha | the PR's statuses, latest per context |
| `headRef()` | `refs/pull/<n>/head` | `refs/heads/<source branch>` |
| `setAutoMerge()` | absent — the hub's workflow does it | **auto-complete** (squash, delete the source branch) |

Both clients return the same `PullSummary` and `PullRequest` records from `src/host/github.ts`, so `pulls()` and `openPr()` need no per-host branches beyond choosing the client [team][github].

## The AzureDevOps client

`AzureDevOps` is bound to one repository and token and builds every URL itself [azure]:

- **Base.** `team.azureApi`, default `https://dev.azure.com`, is the organization-less REST root; a call is `{api}/{org}/{project}/_apis/git/repositories/{repo}{path}?api-version=7.1`. Identity is the one exception: `/{org}/_apis/connectionData` sits above the repository.
- **Auth.** The PAT is sent as `Authorization: Basic <base64(":" + PAT)>`, the same scheme git uses, never in a URL or argv. It is read from `DSH_OKF_AZURE_TOKEN`, falling back to `DSH_OKF_GIT_TOKEN` [azure][team].
- **A sign-in redirect means a bad token.** The client sets `redirect: 'manual'`, so the sign-in page Azure DevOps returns for a missing or expired PAT becomes `GitHubError(401)` with "check DSH_OKF_AZURE_TOKEN", not a parse failure.
- **Errors are `GitHubError`s.** The Host's error wrapper therefore answers `github-<status>` for Azure DevOps failures too [azure].
- **Links.** A pull request's web URL is `{api}/{org}/{project}/_git/{repo}/pullrequest/{id}`, so it follows a configured `azureApi` root [team].

What each call does:

| Call | Detail |
|---|---|
| `openOrUpdate` | finds an active PR by source and target ref, patches its description (and its title only when one is given), else creates one; a body over **4000 characters** is cut with a note, because that is Azure DevOps' description limit |
| `setLabel` | `POST …/labels` to add, `DELETE …/labels/<name>` to remove; a 404 on removal is ignored |
| `openPulls` | `searchCriteria.status=active`, `$top=100`; author is `createdBy.uniqueName` (else display name), `headRef` and `baseRef` are stripped of `refs/heads/`, `headSha` is `lastMergeSourceCommit`, `updatedAt` is the creation date, and inactive labels are dropped |
| `pullState` | `completed` → merged, `abandoned` → closed, anything else open |
| `checks` | `GET …/statuses`; the last status per `genre/name` context counts, `notApplicable` is dropped, `failed`/`error` → failure, any other non-`succeeded` state → pending, and none → `none` |
| `login` | `connectionData.authenticatedUser`: its `id` is what auto-complete is set by, its account (`properties.Account.$value`, else the display name) is the `human:<account>` that Mark reviewed writes |

## Auto-complete

Azure DevOps has no "label turns on auto-merge" mechanism. Instead, after the pull request is opened or updated and the `wiki-auto-merge` tag is set, `openPr()` calls the client's `setAutoMerge` with the verdict [team][azure]:

- **eligible:** `autoCompleteSetBy` is the token user's id, with `completionOptions` `{ mergeStrategy: 'squash', deleteSourceBranch: true, transitionWorkItems: false }`. The pull request completes once its branch policies pass.
- **needs a person:** `autoCompleteSetBy` is cleared with the all-zero GUID, so the pull request waits for someone to complete it.

A failure in either call is appended to the Open pull request answer's `notes`, not raised — the branch is already pushed and the pull request exists. Because auto-complete obeys branch policies, an eligible pull request can still wait for a required reviewer or a running build validation; see [Review state and auto-merge](/concepts/review-and-auto-merge.md).

## The CI guard

`examples/wiki-auto-complete-ado.yml` is the Azure Pipelines counterpart of the GitHub auto-merge workflow [ci-ado]. `Set up the hub` writes it to `.github/wiki/workflows/wiki-auto-complete-ado.yml`; for a hub in Azure Repos you add it as a pipeline and make it a **required Build validation policy** on the base branch. It never trusts the tag:

1. it fetches the base branch and runs that branch's `.github/wiki/check-pr.js` against the pull request, publishing an `eligible` output;
2. when eligible, it lints `$(check.wiki)` with the base branch's `scripts/okf.py` — a lint failure fails the run and blocks completion;
3. when the change needs a person, it PATCHes `autoCompleteSetBy` to the all-zero id with `System.AccessToken`, turning auto-complete off while the run succeeds, so a reviewer completes it by hand.

The `wiki` output variable comes from `src/check-pr.ts`: when `TF_BUILD=True` it prints `##vso[task.setvariable variable=wiki;isOutput=true]<folder>`, in addition to the `GITHUB_OUTPUT` line it writes for GitHub Actions [check-pr]. The pipeline needs the project's Build Service account allowed **Contribute to pull requests** to turn auto-complete off, and a **Require a minimum number of reviewers** policy also holds auto-complete back until an approval lands [ci-ado][readme].

## What is still unverified

`scripts/ado-smoke.mjs` runs the whole pull request side against a stand-in Azure DevOps API written from Microsoft's REST reference — connect, Maintain, an eligible pull request tagged with auto-complete on (squash), the PR list fetched from the source branch with the latest status per context, a reviewed page edited (tag and auto-complete off, title kept), Mark reviewed recording the PAT's account, and a completed pull request closing the workspace and moving the base clone; every call is checked for the PAT and `api-version=7.1` [ado-smoke][readme]. No real Azure DevOps organization, pull request or pipeline has been exercised, so the pipeline's permissions model and the live status shapes are the parts to watch first [readme].

## Related pages

- [Team Wiki service](/concepts/team-wiki-service.md) — where `pullHost()` is chosen and `openPr()` runs.
- [Review state and auto-merge](/concepts/review-and-auto-merge.md) — the verdict auto-complete follows, and the GitHub guard.
- [Host routes and the bundle library](/concepts/host-routes-and-library.md) — how `GitHubError` becomes an API answer.
- [Build, run, test and deploy](/concepts/build-run-test-deploy.md) — the commands to run `ado-smoke.mjs`.
- [Conventions, pitfalls and open questions](/concepts/conventions-and-pitfalls.md) — the traps this host adds.
