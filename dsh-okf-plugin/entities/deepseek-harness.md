---
type: Entity
title: DeepSeek Harness (DSH)
description: The host application the plugin runs inside — its plugin bundles, Host services, client slots and workspace services.
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
  - resource: https://github.com/yishao1224-design/dsh-okf-plugin/blob/b71e34fb1c76d88af680440531fda3c5a9c75c34/README.md
    id: readme-b71e
    title: README.md at b71e34f
  - resource: https://github.com/yishao1224-design/dsh-okf-plugin/blob/b71e34fb1c76d88af680440531fda3c5a9c75c34/src/types/contract.ts
    id: contract
    title: src/types/contract.ts
  - resource: https://github.com/yishao1224-design/dsh-okf-plugin/blob/b71e34fb1c76d88af680440531fda3c5a9c75c34/src/index.ts
    id: host-entry
    title: src/index.ts
  - resource: https://github.com/yishao1224-design/dsh-okf-plugin/blob/b71e34fb1c76d88af680440531fda3c5a9c75c34/src/host/tools.ts
    id: tools
    title: src/host/tools.ts
  - resource: https://github.com/yishao1224-design/dsh-okf-plugin/blob/95b0615aefbbd4d1e16040efdd37d1e1884e5ed0/src/client/index.ts
    id: client-entry
    title: src/client/index.ts
generated:
  by: dsh/agent
  at: 2026-11-23
---

# DeepSeek Harness (DSH)

## What it is

DeepSeek Harness is the application this plugin extends: a chat harness with a web shell, a Host process that owns services and tools, and a plugin bundle system. The plugin treats it as an external platform — nothing is imported from or written into a DSH checkout, and the surfaces it uses were re-declared in `src/types/contract.ts` from DSH 0.2.1-alpha.1, with the tool shapes read from 0.2.1-rc.2 [readme][contract].

## Surfaces this plugin depends on

**Bundling.** A package declares `dsh.bundle.patch` (a `cordis.patch.yml` that inserts one row) and `dsh.client` (`platform: web`, plus the client packages to inject). A profile installs the folder as it is; `lib/` must therefore be committed [readme].

**Host services.**

| Service | Use | Required? |
|---|---|---|
| `connection` | `connection.fetch.register` for exact Fetch routes under `/api` | required (`inject = ['connection']`) |
| `systemPrompt` | `systemPrompt.context(PromptContext)` for the per-turn bundle context | optional, through `ctx.inject` |
| `tools` | `tools.register(ToolDefinition)` for `wiki_sync_from_code` and `wiki_open_pr` | optional, through `ctx.inject` |

`connection.fetch` is used instead of `connection.rpc` because under strict injection `rpc.handle` throws for an out-of-tree plugin; Connection's Host/Origin fence and browser-cookie check run before the handler [host-entry].

The tool definitions are raw `ToolDefinition`s rather than `defineTool` results: a name, a description, an object-rooted JSON-schema parameter map, an `output.schema` with a pure `render`, and an `execute(args, exec)` that throws for a failed call. The tools validate their own arguments and read the calling agent's `session.header.cwd` to find the wiki whose maintenance workspace the session runs in; the contract file keeps `TextBlock`, `ToolRunContext`, `ToolDefinition` and `ToolsService` for exactly this surface [contract][tools].

**Client slots.**

| Slot | Kind | Used for |
|---|---|---|
| `conversation.input.right` | list, session | the OKF chip and the Open pull request button in the composer |
| `conversation.session.header.utilities` | list, session | the same controls in the session header (hidden while a session is blank) |
| `shell.overlay` | list, root | the dialogs |
| `main` | keyed, root | the Team Wiki panel (`okf-team-wiki`) |
| `sidebar.panellist` | list, root | the Team Wiki sidebar entry |

**Client services.** `slots` and `locale` are required; `workspaces` (`create`, `list`), `uiWorkspace` (`connectWorkspace`, `openSession`, optional `startSession`) and `layout` (`selectPanel`) are optional and injected, so the plugin still loads on a shell that lacks them [client-entry].

## What DSH decides that the plugin cannot

- whether a blank session renders header utilities (it does not), which is why the composer copy of the chip exists [readme];
- whether a session's sandbox allows reads under the library folder, which the prompt context depends on [readme];
- the cached system-prompt prefix: a context travels after retained history, so switching bundles does not invalidate it [readme];
- whether and when an automation task's reminder is delivered: reminders run only while DSH is running, after a restart only the latest missed run fires, and a delivered reminder means the message was queued, not that the agent finished [readme-b71e].

## Why it matters here

Almost every "not verified yet" item in the README is about the boundary with DSH rather than the plugin's own code: live pull-request calls, live workspace opening, live reads of `~/.dsh/okf`, the vendored contract version, and whether the Team Wiki tools work from a real automation task [readme-b71e]. See [Conventions, pitfalls and open questions](/concepts/conventions-and-pitfalls.md).

## Related pages

- [Architecture](/concepts/architecture.md) — how the plugin splits along these surfaces.
- [The browser client](/concepts/browser-client.md) — the slots and services in use.
- [Automation tasks and the Team Wiki tools](/concepts/automation-tasks.md) — the `tools` service in use.
- [Team Wiki hub repository](/entities/team-wiki-hub.md) — the other system the plugin writes to.
