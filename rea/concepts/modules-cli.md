---
type: Concept
title: CLI adapter — one-shot commands, output, exit behaviour
description: The `rea` command-line adapter: entry point and dispatcher, command registration, shared analysis workflow binding, global output flags, logging and exit-status rules.
status: draft
lifecycle: draft
confidence: medium
created: 2026-11-20
updated: 2026-11-20
tags: [unverified]
sources:
  - resource: /sources/2026-11-20-package-manifest.md
    id: manifest
    title: package.json manifest and script catalog (2026-11-20)
  - resource: https://github.com/yishao1224-design/rea/blob/8c4f96ef4759fd78da66c4b9726825ca71d7ee0c/src/cli.ts
    id: cli-entry
    title: src/cli.ts
  - resource: https://github.com/yishao1224-design/rea/blob/8c4f96ef4759fd78da66c4b9726825ca71d7ee0c/scripts/rea.mjs
    id: dispatcher
    title: scripts/rea.mjs
  - resource: https://github.com/yishao1224-design/rea/blob/8c4f96ef4759fd78da66c4b9726825ca71d7ee0c/src/cliCommandNames.ts
    id: command-names
    title: src/cliCommandNames.ts
  - resource: https://github.com/yishao1224-design/rea/blob/8c4f96ef4759fd78da66c4b9726825ca71d7ee0c/src/cliOutput.ts
    id: cli-output
    title: src/cliOutput.ts
  - resource: https://github.com/yishao1224-design/rea/blob/8c4f96ef4759fd78da66c4b9726825ca71d7ee0c/src/composition/directAnalysis.ts
    id: direct-analysis
    title: src/composition/directAnalysis.ts
  - resource: https://github.com/yishao1224-design/rea/blob/8c4f96ef4759fd78da66c4b9726825ca71d7ee0c/docs/cli.md
    id: cli-doc
    title: docs/cli.md
generated:
  by: dsh/agent
  at: 2026-11-20
---

# CLI adapter

**Job.** Give a person or a script the same inspection workflows the MCP server
gives an agent, one process per command, with results on stdout and diagnostics
on stderr.

## Key files

| Path | Role |
| --- | --- |
| `src/cli.ts` | `createCli()` — builds the Incur CLI, registers every command module, normalizes argv |
| `src/cliCommandNames.ts` | Canonical command names (`CLI_COMMANDS`, 168 entries), aliases, and the MCP-tool ↔ CLI mapping tables |
| `src/cli/*.ts` | ~60 command modules (`setupCommands`, `coreAnalysisCommands`, `artifactCommands`, `browserCommands`, …) |
| `src/cliOutput.ts` | Global output flags, argv normalization, `--format=value` splitting |
| `src/cli/cliResultOutput.ts`, `jsonOutput.ts`, `tokenWindow.ts`, `analysisEvidenceDelivery.ts` | Result rendering, JSON envelopes, token windows, `--evidence_output` handling |
| `src/application/DirectAnalysis.ts` | Shared one-shot workflow: open → execute → close with cleanup |
| `src/composition/directAnalysis.ts` | Binds `DirectAnalysis` to lazily imported session factories |
| `scripts/rea.mjs` | Executable dispatcher (MCP vs CLI, SIGPIPE handling) |

## Interfaces

### Command registration

`createCli(selectedEnvironment, resultOutput?, fridaInstrumentation?)` snapshots
the environment, creates a `DirectAnalysis`, builds `Cli.create(...)` with
`version: PRODUCT_IDENTITY.packageVersion` and an MCP registration hint, then
calls the `register*Commands` functions [cli-entry]. Registration is lazy with
respect to engines: the comment on `createCli` states that analysis commands
acquire and close their own sessions, and that bare `mcp`/`--mcp` are
intercepted by the dispatcher before `cli.ts` loads.

The command inventory is versioned as data: `CLI_COMMAND_NAMES` has 168 primary
names plus the alias `compare-bundles` [command-names]. The same file records
which CLI commands map to a differently named MCP operation
(`CLI_COMMAND_TOOL_ALIASES`, e.g. `analyze` → `binary_overview`) and which MCP
operations have no dedicated CLI command
(`MCP_TOOLS_WITHOUT_DEDICATED_CLI`, e.g. `open_binary`, `binary_session`,
session-control and prompt-side tools).

### Global output flags

`src/cliOutput.ts` defines the flags that apply to every command
[cli-output]:

- `--format <toon|json|yaml|md|jsonl>` (Incur's formats; default is Incur's own)
- `--filter-output <path>` — project a field out of the result
- `--token-limit` / `--token-offset` — token windows for text results
- Incur builtins removed before command parsing: `--json`, `--full-output`,
  `--token-count`, `--llms`, `--llms-full`, `--help/-h`, `--schema`, `--update`

Constraint enforced by the message constant: token windows cannot be combined
with structured output — use `--filter-output` or command pagination instead.

### Exit status

- Ordinary command failure → non-zero status from the command.
- stdout closed by the reader (`rea … | head`) → status **141** (128 + SIGPIPE),
  with no error printed [dispatcher].
- A configuration error (`REA_LOG_LEVEL`) stops every command except `doctor`,
  which keeps running so the readiness report can name the setting.
- The dispatcher prints a build instruction when the compiled runtime is
  missing: `npm ci && npm run build:cached`.

### Logging

`REA_LOG_LEVEL` selects `trace|debug|info|warn|error|fatal|silent`. CLI logging
is opt-in (the logger defaults to `silent` in `createCli`), and all logs go to
stderr, so stdout carries only results [cli-entry]. `src/cliLogging.ts` records
the command invocation; `logCliCommand` is used by setup/maintenance commands.

### Evidence hand-off between commands

A one-shot process cannot keep session state, so the CLI is explicit about
carrying results forward [cli-doc]:

```bash
rea analyze-javascript-application /absolute/path/to/app --json > app-evidence.json
jq -c '{source: {kind: "inline", evidence: .}, view: {kind: "summary"}}' app-evidence.json > app-view.json
rea inspect-analysis-view app-view.json
```

`{"source": {"kind": "evidence-file", "path": "app-evidence.json"}, …}` lets
`inspect-analysis-view` read and authenticate a saved file instead of charging
it as inline input. Native summaries use
`--detail summary --evidence_output FILE` and later page the retained parent.

## Dependencies

- **Inward:** `src/composition/directAnalysis.ts` → `src/application/DirectAnalysis.ts`
  → `BinarySession` → provider registry → engine adapters. Command modules call
  those workflows and pass ports; provider construction stays in `src/composition/`.
- **Outward:** Incur (CLI framework), pino (logging), Node `process` streams.
- **Boundary:** `src/application/ClientRegistrationIdentity.ts`,
  `SupportedClients.ts`, `Setup*` and `Uninstall` for setup/maintenance
  commands (see [setup and update](/concepts/modules-setup-and-update.md)).

## Verified command surface (sample)

`rea --help` lists the families; the canonical names include `analyze`,
`inspect`, `decompile`, `function`, `xrefs`, `trace`, `search`, `providers`,
`capabilities`, `doctor`, `setup`, `update`, `uninstall`, plus per-format
commands (`inspect-go-binary`, `inspect-il2cpp-metadata`, `inspect-apk-signature`,
`inspect-sqlite-database`, `inspect-artifact`, `extract-artifact`,
`analyze-javascript-application`, `capture-browser-scenario`,
`inspect-web-network-capture`, `capture-process`, `evidence-export`,
`environment-serve`, …) [command-names].

README-documented examples: `npx -y rea-agents@latest analyze-javascript-application
/absolute/path/to/app --json`, `npm install --global rea-agents`, `rea update`
[manifest].

## Pitfalls

- stdout is protocol/result only. Anything written there by a library corrupts
  JSON consumers; diagnostics belong on stderr.
- The dispatcher routes `mcp add` to Incur but `mcp --http` to the HTTP entry,
  so argument order matters for MCP subcommands.
- Compiled runtime is required; `scripts/rea.mjs` does not run TypeScript
  sources. Use `npm run build:cached` from a checkout.
- `--json` is an Incur builtin removed from every argv position; command-local
  options named `--json` would not survive.

## Related pages

- [MCP server](/concepts/modules-mcp-server.md)
- [Providers and sessions](/concepts/modules-providers.md)
- [Build, run, test and deploy](/concepts/build-run-test-deploy.md)
- [Contracts and tool surface](/concepts/modules-contracts.md)
