---
type: Concept
title: Automation Tasks and the Team Wiki Tools
description: The two model tools wiki_sync_from_code and wiki_open_pr that let a DSH automation task keep a project wiki in step with its code, how they pick a wiki, and what they answer.
status: draft
confidence: high
created: 2026-11-23
updated: 2026-11-25
tags: []
sources:
  - resource: https://github.com/yishao1224-design/dsh-okf-plugin/blob/da10648a182b38b410d22f952d8e1d973d8825f0/README.md
    id: readme
    title: README.md
  - resource: https://github.com/yishao1224-design/dsh-okf-plugin/blob/b71e34fb1c76d88af680440531fda3c5a9c75c34/src/host/tools.ts
    id: tools
    title: src/host/tools.ts
  - resource: https://github.com/yishao1224-design/dsh-okf-plugin/blob/b71e34fb1c76d88af680440531fda3c5a9c75c34/src/index.ts
    id: host-entry
    title: src/index.ts
  - resource: https://github.com/yishao1224-design/dsh-okf-plugin/blob/b71e34fb1c76d88af680440531fda3c5a9c75c34/src/types/contract.ts
    id: contract
    title: src/types/contract.ts
  - resource: https://github.com/yishao1224-design/dsh-okf-plugin/blob/da10648a182b38b410d22f952d8e1d973d8825f0/src/host/team.ts
    id: team
    title: src/host/team.ts
  - resource: https://github.com/yishao1224-design/dsh-okf-plugin/blob/da10648a182b38b410d22f952d8e1d973d8825f0/scripts/hub-smoke.mjs
    id: hub-smoke
    title: scripts/hub-smoke.mjs
  - resource: https://github.com/yishao1224-design/dsh-okf-plugin/blob/b71e34fb1c76d88af680440531fda3c5a9c75c34/scripts/harness-check.mjs
    id: harness-check
    title: scripts/harness-check.mjs
generated:
  by: dsh/agent
  at: 2026-11-25
---

# Automation Tasks and the Team Wiki Tools

## Why the tools exist

The Team Wiki page runs **Sync from code** and **Open pull request** when a person presses a button. `src/host/tools.ts` exposes the same two steps to the model, so a DSH automation task — a Schedule reminder delivered into a session that runs in the project's maintenance workspace — can keep a wiki in step with its code without anyone at the page [tools][readme]. Both tools call the `Team` service; they duplicate no git or pull-request work, and a run where the code has not moved costs one `git ls-remote` [tools][team].

## The two tools

| Tool | What it does | What it answers |
|---|---|---|
| `wiki_sync_from_code` | Calls `team.sync(folder)`: compares the code repository's head with the commit the wiki records, and when the code moved opens the maintenance round and prepares the update task | `up-to-date` (`folder`, `commit`); `task` (`folder`, `wiki`, `workspace`, `branch`, `readThrough`, `from?`, `to`, `files`, `pages`, `task`); `nothing-to-do` (`folder`, `workspace`, `to`) |
| `wiki_open_pr` | Calls `team.openPr(record.id, { title, summary })` for the wiki's open maintenance round | `opened` (`number`, `url`, `reused`, `eligible`, `reasons`, `files`, `branch`); `pushed` (files plus a note, when no pull request was opened); `nothing-changed`; `no-round` |

`wiki_sync_from_code` answers `up-to-date` from the early check, when the recorded commit equals the code repository's head and no round is opened. Otherwise it opens (or reuses) the round: a `task` carries the same prompt text the page prefills, with `readThrough: true` when the wiki records no usable code commit and the task is Initialize's full read-through instead of a diff. `nothing-to-do` means the round came up to the recorded commit and no page needs changing. `wiki_open_pr` is what finishes the job: it stamps agent edits as unreviewed, runs `links`, `index`, `lint` and `log`, commits, pushes the wiki branch and opens or updates the round's pull request. If lint fails the call fails and nothing is pushed; the model fixes what it reports and calls again. It **never merges** — a person reviews and merges the pull request [tools][readme].

`wiki_open_pr` answers `no-round` when that wiki has no open maintenance round (call `wiki_sync_from_code` first), and `pushed` when the branch was pushed but the hub host did not open a pull request (another git host, no token, or a reported error — the note says which). A `LibraryError` with code `nothing-changed` becomes `nothing-changed` instead of a failed call [tools][team].

## Choosing the project wiki

Both tools take an optional `folder`, the wiki folder in the hub (for example `payments`). `wikiOf()` resolves it in this order [tools]:

1. **the named folder**, with leading and trailing slashes trimmed; it must be one of the connected hub's non-missing project wikis, else the call fails listing the wikis there are;
2. **the session's workspace**: when `exec.agent.session.header.cwd` is inside a project wiki's maintenance folder, `team.wikiAt(cwd)` names that wiki. `wikiAt` first checks the **open rounds** — a round can sit in a fallback folder (`work/<folder>-<date>-<hex>`) when the usual one was locked at open time — and only then compares canonical paths against `<libraryDir>/team/work/<slug>` and its subfolders, case-folded on Windows, over the project wikis on the base branch plus the folders with a maintenance record [team];
3. **the only project wiki**, when the hub has exactly one;
4. otherwise the call fails and asks for `folder`, listing the choices.

So a session opened with **Open workspace** on the Team Wiki page needs no folder argument even when the hub holds several wikis, while a call from anywhere else must name one [tools][readme].

## Arguments and registration

Both tools declare only optional string arguments — `folder` for both, `title` and `summary` for `wiki_open_pr`. Arguments are validated by the tool itself rather than by `defineTool`: an unknown key, a non-string value or a non-object argument is refused with an error that names what is allowed, and blank strings are dropped while values are trimmed [tools][contract]. An unknown `folder` is refused with the list of project wikis, which is also what the hub smoke test checks [hub-smoke].

`src/index.ts` builds the definitions with `wikiTools(team)` and registers each one through `ctx.inject(['tools'], …)` unless the row config sets `tools: false` [host-entry][readme]. The tools are registered **globally for every session**, not per attached bundle, and they only ever act on the connected team wiki. The contract shapes they need — `TextBlock`, `ToolRunContext`, `ToolDefinition`, `ToolsService` — are vendored in `src/types/contract.ts`; the tools are raw `ToolDefinition`s with a JSON-schema argument object, an `output.schema` and a pure `render` that turns the value into one text block [contract].

## Running one as an automation task

The README's setup, which is a DSH flow rather than a plugin one [readme]:

1. On the **Team Wiki** page pick the project and press **Open workspace**. That opens a session in `<libraryDir>/team/work/<folder>`, the folder every round reuses, so the session stays valid.
2. Give that session a permission preset that lets it edit files in the workspace and run `python scripts/okf.py` without asking; otherwise every run waits for an approval.
3. Ask it, for example: *"Every weekday at 09:00 Asia/Shanghai: call wiki_sync_from_code. If it gives a task, do it, then call wiki_open_pr with a one-sentence summary. If it is up to date, stop."* The model creates the reminder with `schedule_create`, and it appears on DSH's **Automation tasks** page, where its time and text can be edited.

Each time the reminder fires, DSH posts it into that session as a follow-up message. On a quiet day the run ends after one `git ls-remote`. When the code moved, the agent updates the pages and `wiki_open_pr` opens the pull request, which then appears in the Team Wiki list for a person; if the previous round's pull request is still open, the new changes go on the same branch and update it [readme].

## How the sync commit is recorded

Sync records the code commit in the wiki's `log.md` **as a dated Sync entry**, not in `SCHEMA.md` — `wiki_sync_from_code` gets this for free from `team.sync()`, and `wiki_open_pr` commits it with the pages [tools][team]. Because `log.md` is an allowed file for the auto-merge rule while `SCHEMA.md` always needs a person, a sync whose changed pages are all unreviewed stays eligible to merge on its own. A page someone had reviewed still makes it need a person. See [Sync from code](/concepts/team-wiki-service.md#sync-from-code) and [Review state and auto-merge](/concepts/review-and-auto-merge.md).

## What DSH's scheduler does and does not do

DSH 0.2.1's scheduler, as the README describes it [readme]:

- reminders run only while DSH is running, and after a restart only the latest missed run fires;
- every run goes to the same session, so its history grows (compaction keeps it in bounds);
- a delivered reminder means the message was queued, not that the agent finished — check the pull request list rather than the task's run log.

The tools themselves are registered for every session but only act on the connected team wiki and only ever open pull requests; `tools: false` in the row config leaves them out [readme][host-entry].

## What is still unverified

The tools have **not yet run from a real automation task**: a Schedule reminder firing in a live maintenance session, with its permission preset, has not been tried [readme]. What exists is two stand-ins:

- `scripts/harness-check.mjs` registered them in DSH 0.2.1-rc.2's real `ToolRuntime` and called them through its pipeline: a call with no team wiki connected fails with a message the model can act on, bad arguments are refused, and disposal removes them [harness-check];
- `scripts/hub-smoke.mjs` ran them against a stand-in GitHub Enterprise hub, as an automation task would: an unknown folder is refused with the choices, `wiki_sync_from_code` answers up to date and then a task after a code commit, Initialize and each sync record `code_commit` in `log.md`, `wiki_open_pr` updates the round's pull request (a second call updates the same one), and with two project wikis a call outside any workspace asks for the folder while one from the payments workspace finds it [hub-smoke].

## Related pages

- [Team Wiki Service](/concepts/team-wiki-service.md) — the `sync()`, `openPr()` and `wikiAt()` methods the tools call.
- [Review state and auto-merge](/concepts/review-and-auto-merge.md) — why recording the commit in `log.md` keeps a sync eligible.
- [Data model](/concepts/data-model.md) — the records the tools answer with.
- [DeepSeek Harness (DSH)](/entities/deepseek-harness.md) — the `tools` Host service and its registry.
