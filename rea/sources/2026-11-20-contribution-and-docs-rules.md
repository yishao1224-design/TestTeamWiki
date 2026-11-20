---
type: Source
title: Contribution and documentation rules (capture, 2026-11-20)
description: The repository's operating contract (AGENTS.md), developer setup and PR rules (CONTRIBUTING.md), tool-design rules and the docs guide map.
status: draft
lifecycle: draft
created: 2026-11-20
updated: 2026-11-20
source_url: https://github.com/yishao1224-design/rea/blob/8c4f96ef4759fd78da66c4b9726825ca71d7ee0c/AGENTS.md
retrieved: 2026-11-20
tags: [unverified]
generated:
  by: dsh/agent
  at: 2026-11-20
---

# Contribution and documentation rules (capture)

Excerpts of `AGENTS.md` (11,895 bytes), `CONTRIBUTING.md` (22,527 bytes),
`docs/tool-design.md` (14,514 bytes) and `docs/index.md` (71 lines).

## AGENTS.md — repository guidelines (headings and key rules)

Headings: *Product and Authority*, *Architecture and Cleanup*, *Performance*,
*Development*, *Tool and Test Changes*, *Issue Reports*, *Pull Request
Readiness*.

Verbatim rules worth preserving:

> Dependencies flow inward: domain semantics and contracts, provider adapters,
> shared application workflows, then CLI/MCP adapters. Keep engine protocols and
> provider-specific code out of domain/application layers.

> Give each interpretation and resource a clear owner. Consumers must use that
> owner's parsed facts rather than reinterpret raw source.

> Keep proof of ownership separate from validity, editability, and readiness.
> Malformed owned data may still be safely removable. Unresolved identity or
> applicability must remain unknown rather than be treated as absent, foreign,
> or aligned.

> Configuration changes must be additive and idempotent, with backups.
> Installers must not install or upgrade unrelated software.

> `rea setup` must show its planned changes and require approval before writing
> files or installing Hopper.

> Do not integrate capabilities while material scope, authority, side-effect, or
> required verification questions remain unresolved.

Performance priorities, in order: (1) agent tokens and context; (2) analysis
time; (3) CPU and memory. Cached live state must be freshness-checked before it
is reported as current.

Development commands listed: `npm ci`, `npm run build:cached`,
`npm run test:local -- PATH...`, `npm run test:focused -- PATH...`,
`npm run check:fast`, `npm run docs:check`, optional `npm run check:pr`.
Ignored outputs named: `docs/public/product-catalog.json`,
`docs/verification/managed-conformance-*.json`, `skills/`,
`.cache/mcp-tool-catalog.json` (test metadata). Tracked generated metadata:
`src/generatedPackageMetadata.ts`.

## CONTRIBUTING.md — development setup (verbatim commands)

```bash
npm ci
npm run check:fast
```

> `npm ci` installs the exact dependencies and prepares the Husky hooks without
> building the project. Run `npm run build:cached` when you need the standalone
> CLI or MCP server; it compiles the runtime and bundles the authored skill.
> `npm run build` forces those tasks to rerun.

> After a package, lockfile, or managed-skill version change, run
> `npm run metadata:generate` before building.

Focused-change example:

```bash
npm run test:focused -- src/config.test.ts
npm run check:fast
```

> Before requesting review, run relevant tests and `npm run check`; add
> `npm run docs:check` when contracts or generated metadata change.

> Packaging/setup/distribution changes additionally require
> `npm run verify:package` and `npm pack --dry-run`.

Other stated constraints:

- Windows source checkouts do not build `native/windows/build/`; the native
  addon build requires Linux, MinGW-w64 and `tar`.
- `git submodule update --init --recursive` fails at the pinned Unblob
  submodule on Windows; a documented workaround exists in
  `docs/firmware-analysis.md`.
- Pre-commit formats and lints staged files; pre-push runs `check:fast`.
- `REA_LOG_LEVEL` selects `trace|debug|info|warn|error|fatal|silent`; logs go
  to stderr, MCP defaults to `info`, one-shot CLI logging is opt-in.
- Build/generation commands hold a SQLite transaction around the Turbo graph;
  `.cache/rea-command-locks/` databases must not be removed while commands run.
- Temporary files go through `createTestTempDirectory`; never glob-clean
  shared `/tmp/rea-*`.

## docs/tool-design.md — shape and limits table

| Shape | Use it for | Result |
| --- | --- | --- |
| `inspect` | facts about one target, object or resource | relevant fields, locations, facet availability |
| `search` / `list` | candidate discovery | stable ordering and context, pagination when needed |
| `trace` | relationships across evidence | typed edges, supporting facts, unresolved paths |
| `compare` | explicitly paired artifacts or observations | identity, comparable coverage, supported deltas |
| `workflow` | a recurring outcome needing composition | inline answer, contributing evidence, partial facets |
| `observe` / `capture` | runtime behavior | authority, launch/attach effects, lifecycle, cleanup status |

> Add limits only for real format, protocol, authority, or measured resource
> constraints. […] An assumed agent budget is insufficient justification for a
> limit.

> Keep observed, derived, and inferred relationships distinct.

## docs/index.md — guide map (table, condensed)

| Task | Guides |
| --- | --- |
| Identify a desktop installation | `desktop-applications.md` |
| Inspect native functions, references, types | `native-investigation.md`, `go-binaries.md`, `rust-binaries.md`, `python-pyc.md`, `pe-certificates.md`, `elf-notes.md`, `erlang-beam.md`, `binary-diagnostics.md`, `recorded-crashes.md` |
| Select a native provider or import format | `ghidra-provider.md`, `windows-ghidra-p0.md`, `ghidra-dos.md`, `raw-binary.md`, `ghidra-mips.md`, `ghidra-psp.md`, `ghidra-nativeaot.md`, `ida-provider.md`, `kuna-provider.md`, `jeb-analysis.md` |
| Analyze JavaScript or Electron | `javascript-artifact-reconstruction.md`, `javascript-application-workflows.md`, `javascript-application-graph.md`, `javascript-recovery.md` |
| Compare static JS with runtime | `javascript-runtime-observation.md`, `javascript-runtime-reconciliation.md`, `electron-observation.md`, `electron-fuses.md` |
| Assemblies, packages, firmware | `godot-pck.md`, `managed-code-analysis.md`, `python-packaged-applications.md`, `android-analysis.md`, `apk-signature.md`, `apktool-resource-analysis.md`, `adb-device-analysis.md`, `apple-application-analysis.md`, `harmony-analysis.md`, `firmware-analysis.md`, `wasm-artifacts.md`, `evm-bytecode.md` |
| Browser scenarios | `browser-observation.md`, `browser-scenario-contract.md`, `web-runtime.md`, `web-network-captures.md` |
| Serve recorded behavior | `agent-environments.md` |
| Trace captured website source | `web-source-location.md`, `website-module-trace.md`, `website-script-export.md` |
| Capture and compare a command | `process-capture.md` |
| CLI/MCP and evidence retention | `cli.md`, `mcp-contracts.md`, `mcp-prompts.md`, `reconstruction-obligation-ledgers.md` |

Development guides named in the same file: `testing.md`,
`client-verification.md`, `tool-design.md`, `design/limits-and-recovery.md`,
`releasing.md`, `reconstruction-readiness.md`, `migration-6.2.md`,
`roadmap.md`.
