---
type: Concept
title: REA overview — what it is, who uses it, main flows
description: An introduction to REA as a local-only reverse-engineering MCP server and CLI: its purpose, its users, the investigation loop it supports, and the set of targets it covers.
status: draft
lifecycle: draft
confidence: medium
created: 2026-11-20
updated: 2026-11-20
tags: [unverified]
sources:
  - resource: /sources/2026-11-20-readme.md
    id: readme
    title: README capture (2026-11-20)
  - resource: /sources/2026-11-20-package-manifest.md
    id: manifest
    title: package.json manifest and script catalog (2026-11-20)
  - resource: https://github.com/yishao1224-design/rea/blob/8c4f96ef4759fd78da66c4b9726825ca71d7ee0c/README.md
    id: readme-file
    title: README.md
  - resource: https://github.com/yishao1224-design/rea/blob/8c4f96ef4759fd78da66c4b9726825ca71d7ee0c/package.json
    id: package-json
    title: package.json
  - resource: https://github.com/yishao1224-design/rea/blob/8c4f96ef4759fd78da66c4b9726825ca71d7ee0c/docs/index.md
    id: docs-index
    title: docs/index.md
  - resource: https://github.com/yishao1224-design/rea/blob/8c4f96ef4759fd78da66c4b9726825ca71d7ee0c/src/cli.ts
    id: cli-entry
    title: src/cli.ts
  - resource: https://github.com/yishao1224-design/rea/blob/8c4f96ef4759fd78da66c4b9726825ca71d7ee0c/src/main.ts
    id: mcp-entry
    title: src/main.ts
generated:
  by: dsh/agent
  at: 2026-11-20
---

# REA overview

REA ("Reverse Engineer Anything") is a local reverse-engineering toolkit shipped
as one npm package, `rea-agents`, that exposes the same analysis workflows
through two adapters: a Model Context Protocol (MCP) server that an agent
connects to, and a one-shot command-line interface. Both adapters call the same
application layer, so a CLI run and an agent tool call produce the same evidence
records [readme].

The project describes itself as "One MCP for reverse engineering across
binaries, applications, and runtime behavior" and states the intended use:
inspect an app without its source, explain how a feature works, show the
evidence, and build a version for your own project [readme].

## What it is for

- **Feature understanding and reimplementation.** The primary user story is an
  agent investigating how an existing application implements something, then
  writing and testing an equivalent implementation.
- **Evidence-bearing answers.** Every result is an `Evidence` record carrying
  the provider, operation, parameters, normalized result, confidence
  (`observed`/`derived`/`inferred`), authority, limitations, source locations
  and links to related Evidence. See [the data model](/concepts/data-model.md).
- **Local-only analysis.** The README states that analysis runs locally and
  that REA does not upload the target; only the agent's model provider receives
  the tool results [readme].
- **Broad format coverage from one entry point.** Native binaries, JavaScript
  and Electron applications, .NET assemblies, websites and saved network
  captures are the headline targets, alongside ~30 further formats (Go, Rust,
  IL2CPP, GPU code objects, WASM, EVM, SQLite, Android, firmware, packets,
  signals, PyInstaller, and more) [readme].

## Who uses it

1. **Coding agents.** `npx rea-agents setup` registers the MCP server and
   installs matching workflow instructions for Claude Code, Codex, Cursor,
   Gemini CLI, Grok Build and other clients, with backups of existing
   configuration [readme]. A project-local registration scope also exists
   (`docs/installation.md`).
2. **People at a terminal.** `npm install --global rea-agents` installs the
   `rea` binary; `npx rea-agents@latest <command>` runs one-off commands
   [readme]. The CLI exposes 168 primary command names plus the
   `compare-bundles` alias [manifest].
3. **Contributors and maintainers.** The repository carries a large
   contributor contract: `AGENTS.md`, `CONTRIBUTING.md`, `docs/testing.md`,
   `docs/tool-design.md`, and real-provider verification lanes for each
   engine.

## Main flows

### 1. Setup and registration

`rea setup` discovers supported clients, plans additive configuration changes,
shows them, asks for approval, writes with backups, optionally installs Hopper
after a separate approval, and installs the bundled skill. `rea doctor` is the
readiness report; `rea update` is the verified updater; `rea uninstall` reverses
registration. See [setup and update](/concepts/modules-setup-and-update.md).

### 2. Agent investigation over MCP

The agent calls a tool such as `open_binary`, `analyze_function` or
`inspect_*`. The MCP adapter translates the call into an application workflow
on one shared `BinarySession`, which routes it to a provider client (Hopper,
Ghidra, IDA, Kuna, or an auxiliary/static provider). The result is recorded in
the session's evidence ledger and returned inline. A target stays open across
turns; `close_binary` ends it and can save an analysis snapshot. See
[MCP server](/concepts/modules-mcp-server.md) and
[providers and sessions](/concepts/modules-providers.md).

### 3. One-shot CLI inspection

Each CLI invocation is a separate process that builds its own session, runs one
command, prints JSON or a table, and exits. Saved evidence and snapshots are the
hand-off between invocations (`--evidence_output`, `evidence-export`,
`inspect-analysis-view`) [cli-entry]. See the [CLI module](/concepts/modules-cli.md).

### 4. Static analysis without an engine

Many targets need no disassembler: JavaScript/Electron static analysis, .NET
metadata, Go/Rust build metadata, IL2CPP metadata, GPU code objects, model
headers, WASM, APK signatures, SQLite, Godot PCK, firmware inventories. These
are ordinary providers and services registered on the same server
[readme-file]. See [static artifacts](/concepts/modules-static-artifacts.md).

### 5. Runtime observation

Process capture, browser/CDP observation, Playwright scenarios, Frida
instrumentation, GDB/Rizin debugger sessions and LLDB-based native call tracing
run or attach to targets. These carry explicit launch/attach effects and
ownership rules, which is why the process layer is treated as its own module.
See [process supervision](/concepts/modules-process-supervision.md).

## What ships

- npm package `rea-agents` version `6.7.0`, MIT, entry `scripts/rea.mjs`,
  engines `^22.23.4 || ^24.22.1 || >=26.12.1` [package-json].
- An MCP Registry descriptor `io.github.morluto/rea` pointing at the npm
  package with stdio transport and the positional argument `mcp`
  (`server.json`, same version).
- A VitePress documentation site under `docs/`, and a separate static website
  under `website/` served at <https://rea.tools/> [docs-index].

## Boundaries the project states

- REA is for lawful research, education, interoperability and software
  preservation; commercial engines (IDA/Hex-Rays, Hopper, JEB) must be legally
  obtained, and REA grants no licence or activation entitlement [readme-file].
- `CONTRIBUTING.md` excludes purpose-built packing/DRM/anti-cheat bypass,
  anti-instrumentation evasion, and exploit automation from official
  distributions, and requires a fail-closed decision when scope or authority is
  unresolved.
- Native analysis is bring-your-own-engine; REA does not install or update
  Ghidra, IDA or JEB [readme].

## Related pages

- [Architecture](/concepts/architecture.md) — components and data flow.
- [Data model](/concepts/data-model.md) — Evidence, bundles, unknowns, snapshots.
- [Build, run, test and deploy](/concepts/build-run-test-deploy.md) — exact commands.
- [Conventions, pitfalls and open questions](/concepts/conventions-pitfalls-open-questions.md).
