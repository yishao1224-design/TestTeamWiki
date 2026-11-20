---
type: Concept
title: Architecture — components, communication and where data lives
description: The layered architecture of REA: dispatcher and entry points, MCP and CLI adapters, shared application workflows, provider layers, the process foundation, contracts and domain, plus the runtime data locations.
status: draft
lifecycle: draft
confidence: medium
created: 2026-11-20
updated: 2026-11-20
tags: [unverified]
sources:
  - resource: /sources/2026-11-20-architecture-diagram.md
    id: architecture-capture
    title: docs/architecture.mermaid capture (2026-11-20)
  - resource: /sources/2026-11-20-mcp-server-wiring.md
    id: wiring-capture
    title: MCP server wiring capture (2026-11-20)
  - resource: https://github.com/yishao1224-design/rea/blob/8c4f96ef4759fd78da66c4b9726825ca71d7ee0c/docs/architecture.mermaid
    id: architecture-map
    title: docs/architecture.mermaid
  - resource: https://github.com/yishao1224-design/rea/blob/8c4f96ef4759fd78da66c4b9726825ca71d7ee0c/scripts/rea.mjs
    id: dispatcher
    title: scripts/rea.mjs
  - resource: https://github.com/yishao1224-design/rea/blob/8c4f96ef4759fd78da66c4b9726825ca71d7ee0c/src/composition/binary.ts
    id: composition-binary
    title: src/composition/binary.ts
  - resource: https://github.com/yishao1224-design/rea/blob/8c4f96ef4759fd78da66c4b9726825ca71d7ee0c/src/application/binary/BinarySession.ts
    id: binary-session
    title: src/application/binary/BinarySession.ts
  - resource: https://github.com/yishao1224-design/rea/blob/8c4f96ef4759fd78da66c4b9726825ca71d7ee0c/AGENTS.md
    id: agents-rules
    title: AGENTS.md
generated:
  by: dsh/agent
  at: 2026-11-20
---

# Architecture

REA is a TypeScript (ESM, `NodeNext`) codebase with a single declared layering
rule: *dependencies flow inward: domain semantics and contracts, provider
adapters, shared application workflows, then CLI/MCP adapters* [agents-rules].
The repository's own component map is
[`docs/architecture.mermaid`](https://github.com/yishao1224-design/rea/blob/8c4f96ef4759fd78da66c4b9726825ca71d7ee0c/docs/architecture.mermaid)
and is captured verbatim in the bundle [architecture-capture].

## Layers, top to bottom

### 1. Executable dispatch

`scripts/rea.mjs` is the only `bin` entry. It reads the package version into
`REA_PACKAGE_VERSION`, inspects `argv`, and loads one of three compiled
runtimes [dispatcher]:

| Invocation | Runtime |
| --- | --- |
| `rea mcp` / `rea --mcp` | `dist/mcp/main.js` → `runEntrypoint()` (stdio) |
| `rea mcp --http …` | `dist/mcp/main.js` → `runMcpHttpEntrypoint()` |
| `rea mcp doctor …` | `dist/mcp/main.js` + `dist/cli-runtime/mcpDoctor.js` |
| anything else | `dist/cli-runtime/cli.js` → Incur CLI |

The dispatcher also handles environment quirks (empty `USERPROFILE` on Windows,
Playwright's cache path on Termux), verifies the compiled runtime exists, and
maps a closed stdout pipe to exit status 141.

### 2. Adapters

- **MCP adapter** — `src/main.ts` plus `src/main/`: parses configuration,
  chooses deferred or eager startup, opens an optional initial target, starts
  the stdio or streamable-HTTP transport, and installs shutdown handlers
  [wiring-capture].
- **CLI adapter** — `src/cli.ts` plus `src/cli/`: builds an Incur CLI, registers
  ~60 command modules, and drives one-shot workflows through
  `composition/directAnalysis.ts`.
- **MCP request surface** — `src/server/`: `EvidenceMcpServer` (extends
  `EvidenceMcpServerCore`, which wraps `RequestTrackingMcpServer`), one
  `register*Tools.ts` module per capability family, prompts, delivery budgets
  and result encoding.
- **Host installation adapters** — `src/application/Setup*`, `SetupHost`,
  `Uninstall`, `Update*`, and the platform-specific Hopper installers
  (`MacHopper`, `LinuxHopper`).

### 3. Composition

`src/composition/` is the only production wiring point. Factories construct
fresh providers, sessions and optional adapters *without* starting engines or
acquiring targets; `createBinarySession` is the single function both adapters
use, which is what keeps CLI and MCP behaviour identical [composition-binary].
Notable factories: `binary.ts`, `directAnalysis.ts`,
`optionalObservationProviders.ts`, `auxiliaryAnalysisProviders.ts`, and one
small module per provider (`ghidra.ts`, `hopper.ts`, `ida.ts`, `jeb.ts`,
`wasm.ts`, `evm.ts`, …).

### 4. Application workflows

`src/application/` holds the shared workflows and the target lifecycle:

- `BinarySession` owns exactly one active target, serializes target
  transitions, binds a provider client, records Evidence, and exposes
  snapshot/import/export and status operations [binary-session].
- `AnalysisProviderRegistry` + `SessionProviderRouter` select one deep provider
  deterministically and merge it with disjoint auxiliary providers.
- Format-specific services (Android, firmware, browser, JavaScript, managed,
  process capture, environment serving, …) sit here.

### 5. Provider layers

Engine adapters under `src/hopper/`, `src/ghidra/`, `src/ida/`, `src/kuna/`,
`src/jeb/`, `src/cutter/`, `src/gdb/`, `src/rizin/`, `src/objdump/`,
`src/frida/`; static format adapters under `src/artifacts/`, `src/native/`,
`src/dotnet/`, `src/go/`, `src/rust/`, `src/il2cpp/`, `src/gpu/`, `src/model/`,
`src/python/`, `src/erlang/`, `src/wasm/`, `src/evm/`, `src/apkSignature/`,
`src/sqlite/`, `src/pcap/`, `src/signals/`, `src/captures/`, `src/firmware/`,
`src/android/`, `src/apktool/`, `src/adb/`, `src/flutter/`, `src/browser/`,
`src/inspector/`, `src/electron/`, `src/reference/`.

Bridge code that must run inside an engine lives outside `src/` in `bridge/`:
`hopper_bridge.py` (inside Hopper), `ghidra/ReaGhidraBridge.java` (packaged
Ghidra `HeadlessScript`), `cutter/rea_bridge.py`, `android/ReaJadxBridge.java`,
`mitmproxy/capture.py`, `native/rea_lldb_tracer.py`,
`pwndbg/core_context.py`, `pwntools/*`, `nativeaot/*.java`.

### 6. Foundation: process, host and containers

`src/process/` provides owned process groups, run tokens, deadlines, bounded
output capture, private runtime roots and cleanup retry. `src/windows/` plus
`native/windows/src/*.cc` add a Node-API addon for NTFS admission, protected
DACLs and Job Objects. `src/filesystem/`, `src/config/` and `src/logger.ts`
complete the substrate. See
[process supervision](/concepts/modules-process-supervision.md).

### 7. Contracts and domain

`src/contracts/` holds caller-visible Zod schemas and their JSON Schema
projections; `src/domain/` holds pure, side-effect-free modules (evidence,
comparisons, snapshots, error algebra, format readers' value types). Neither
layer imports adapters [agents-rules]. See
[contracts and tool surface](/concepts/modules-contracts.md) and
[data model](/concepts/data-model.md).

## How the parts talk

- **MCP client → adapter.** JSON-RPC over stdio by default, or streamable HTTP
  when `rea mcp --http` is used. The HTTP listener is imported lazily so the
  default stdio entry never loads it [wiring-capture].
- **Adapter → session.** Every tool handler receives a narrow port
  (`evidenceById`, `activeTarget`, `recordEvidence`,
  `recordEvidenceWithUnknown`, `withAdmittedAnalysis`) rather than the concrete
  session.
- **Session → provider.** A provider client is created per target binding; the
  session routes operations through an immutable `SessionProviderRoute` and
  merges deep and auxiliary capabilities with `CompositeProvider`.
- **Provider → engine.** Long-lived engines speak line/framed protocols over
  Unix domain sockets (Hopper, Ghidra on POSIX) or Windows transport; one-shot
  tools are spawned as owned commands with deadlines and bounded output.
- **Engine → Evidence.** Providers normalize engine output into typed values;
  `createEvidence` builds the record and computes its semantic ID.
- **Session → ledger.** `recordEvidence` writes into `InvestigationRecords` →
  `EvidenceLedger`, which rejects conflicting content and supports atomic
  bundle import/export.
- **Shutdown.** The transport, server instances and providers are closed in an
  idempotent chain; failures are aggregated and reported rather than swallowed.

## Where data lives

REA is a local, stateless-service design: there is no database and no server
state beyond one process.

| Data | Location | Lifetime |
| --- | --- | --- |
| Evidence records and unknowns | in-memory `EvidenceLedger` inside the session | process lifetime (or until the session closes/clears) |
| Target binding and provider client | `BinarySession` fields | until target switch or `close_binary` |
| Cached analysis snapshots | `AnalysisSnapshotCache` in the session | process lifetime; exportable as JSON |
| Saved snapshots and evidence bundles | caller-chosen files (`--evidence_output`, `evidence_export`, `closeWithSnapshot`) | persistent |
| Generated catalogs | `dist/mcp-catalog.json`, `dist/mcp-catalog-full.json` next to the compiled runtime | build artifact |
| Configuration | `process.env` snapshot (`snapshotEnvironment`) | process lifetime |
| Logs | stderr via pino; stdout is reserved for protocol/results | — |
| Provider working files | REA-owned private runtime roots (per-run directories) | cleaned on close, with retry ownership |

The `binary_session` MCP tool is the runtime authority for catalog identity,
counts, tool availability and the active target; docs deliberately do not
freeze those numbers.

## Why the layering matters operationally

- `createServer` constructs ~40 services eagerly but does not start engines, so
  a server can answer `tools/list` on a host with no providers installed.
- Deferred startup (`runDeferred`) skips even that construction: it serves the
  build-recorded catalog, then materializes the same server object on the first
  tool request.
- Because both adapters go through `composition/binary.ts`, a fix to session,
  profile or cleanup semantics applies to CLI and MCP at once.

## Related pages

- [Overview](/concepts/overview.md)
- [Providers and sessions](/concepts/modules-providers.md)
- [Native analysis providers](/concepts/modules-native-providers.md)
- [Conventions, pitfalls and open questions](/concepts/conventions-pitfalls-open-questions.md)
