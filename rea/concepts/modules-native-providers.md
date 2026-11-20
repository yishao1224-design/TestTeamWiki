---
type: Concept
title: Native analysis providers — Hopper, Ghidra, IDA, Kuna, JEB and debuggers
description: The engine-backed providers that supply pseudocode and assembly, how each is launched and spoken to, their bridges, configuration, and the shared rules for deadlines and cleanup.
status: draft
lifecycle: draft
confidence: medium
created: 2026-11-20
updated: 2026-11-20
tags: [unverified]
sources:
  - resource: https://github.com/yishao1224-design/rea/blob/8c4f96ef4759fd78da66c4b9726825ca71d7ee0c/src/composition/binary.ts
    id: composition
    title: src/composition/binary.ts
  - resource: https://github.com/yishao1224-design/rea/blob/8c4f96ef4759fd78da66c4b9726825ca71d7ee0c/src/hopper/BridgeLauncher.ts
    id: hopper-launcher
    title: src/hopper/BridgeLauncher.ts
  - resource: https://github.com/yishao1224-design/rea/blob/8c4f96ef4759fd78da66c4b9726825ca71d7ee0c/bridge/hopper_bridge.py
    id: hopper-bridge
    title: bridge/hopper_bridge.py
  - resource: https://github.com/yishao1224-design/rea/blob/8c4f96ef4759fd78da66c4b9726825ca71d7ee0c/src/ghidra/GhidraTransport.ts
    id: ghidra-transport
    title: src/ghidra/GhidraTransport.ts
  - resource: https://github.com/yishao1224-design/rea/blob/8c4f96ef4759fd78da66c4b9726825ca71d7ee0c/bridge/ghidra/ReaGhidraBridge.java
    id: ghidra-bridge
    title: bridge/ghidra/ReaGhidraBridge.java
  - resource: https://github.com/yishao1224-design/rea/blob/8c4f96ef4759fd78da66c4b9726825ca71d7ee0c/src/ida/IdaConfiguration.ts
    id: ida-config
    title: src/ida/IdaConfiguration.ts
  - resource: https://github.com/yishao1224-design/rea/blob/8c4f96ef4759fd78da66c4b9726825ca71d7ee0c/src/kuna/KunaProvider.ts
    id: kuna-provider
    title: src/kuna/KunaProvider.ts
  - resource: https://github.com/yishao1224-design/rea/blob/8c4f96ef4759fd78da66c4b9726825ca71d7ee0c/src/jeb/JebProvider.ts
    id: jeb-provider
    title: src/jeb/JebProvider.ts
  - resource: https://github.com/yishao1224-design/rea/blob/8c4f96ef4759fd78da66c4b9726825ca71d7ee0c/docs/native-investigation.md
    id: native-doc
    title: docs/native-investigation.md
generated:
  by: dsh/agent
  at: 2026-11-20
---

# Native analysis providers

**Job.** Supply the deep facts — functions, pseudocode, assembly, symbols,
cross-references, call graphs, types — that static format readers cannot. All
of them implement the same `AnalysisProvider` contract, so the session and the
tool surface do not know which engine answered.

The deep candidates registered by `createBinarySession` are exactly
`HopperProvider`, `GhidraProvider`, `IdaProvider` and `KunaProvider`
[composition]. JEB, Cutter, GDB, Rizin, objdump and Frida are registered
elsewhere: JEB is a composed provider; the others are services behind their own
tools.

## Comparison at a glance

| Provider | Where it runs | Transport / bridge | Main configuration |
| --- | --- | --- | --- |
| Hopper | Local Hopper app (macOS; Linux demo/package) | Unix socket to `bridge/hopper_bridge.py` running inside Hopper | `HOPPER_LAUNCHER_PATH`, `REA_HOPPER_STARTUP_TIMEOUT_MS`, `HOPPER_LOADER_ARGS_JSON`, `HOPPER_TARGET_PATH` |
| Ghidra | Local Ghidra install, headless | Unix socket, or authenticated loopback TCP on Windows, to `bridge/ghidra/ReaGhidraBridge.java` | `GHIDRA_INSTALL_DIR`, `JAVA_HOME`, `REA_GHIDRA_STARTUP_TIMEOUT_MS`, language/compiler-spec overrides, seeds, disabled analyzers, `REA_GHIDRA_NATIVEAOT_JAR` |
| IDA | Caller's IDA + upstream `ida-pro-mcp` | MCP stdio or loopback HTTP registration | `REA_IDA_MCP_CONFIG` (absolute JSON file) |
| Kuna | Local Kuna CLI | One-shot owned commands with project cache | `REA_KUNA_COMMAND`, `REA_KUNA_CACHE_DIR`, `REA_KUNA_CACHE_MAX_BYTES`, `REA_KUNA_TIMEOUT_MS`, `REA_KUNA_PROJECT_TIMEOUT_MS` |
| JEB | Caller-started JEB client serving MCP | MCP over HTTP | `REA_JEB_MCP_URL` (default `http://127.0.0.1:8425/mcp`), `REA_JEB_MCP_TIMEOUT_MS` |
| Cutter | Local Cutter with REA bridge | Socket to `bridge/cutter/rea_bridge.py` | `REA_CUTTER_BRIDGE_DIR`, `REA_BRIDGE_MAX_REQUEST_BYTES` |
| GDB / Rizin | Host debugger binaries | Persistent session protocols (GDB/MI, Rizin commands) | session tools; owned process supervision |
| GNU objdump | Host `objdump` | One-shot owned command | caller-selected `objdump` path and operation in the tool input |
| Frida | Target device/process | Frida instrumentation sessions | `list_frida_devices`, `instrument_with_frida`, … |

## Hopper

- `BridgeLauncher` launches or attaches to Hopper and loads
  `bridge/hopper_bridge.py` inside the app's Python runtime; the Node side then
  speaks a framed protocol over a Unix socket
  (`HopperSocketConnection`, `HopperRequestQueue`, `HopperResponseStream`).
- macOS launcher candidates are probed from the home directory first, then the
  system path (`MAC_HOPPER_SYSTEM_LAUNCHER`); Linux uses
  `linuxHopperLauncherCandidates` and falls back to
  `REA_MANAGED_LINUX_HOPPER_LAUNCHER` (`src/config/parseConfig.ts`).
- Setup may install Hopper after a separate approval, using verified vendor
  downloads (`MacHopper`, `LinuxHopperPublisher`, `HopperDownload`).
- A first-run macOS dialog (demo mode or licence) must be completed in the same
  user session before unattended use; the README FAQ says REA starts Hopper
  when an operation needs it.
- Startup deadline is `REA_HOPPER_STARTUP_TIMEOUT_MS`; the transcript keeps
  distinct launch, handshake, request and cleanup stages
  (`HopperOperationStage`, `HopperDiagnostics`).

## Ghidra

- Discovery and launch: `GhidraInstallation`, `GhidraInstallationPolicy`,
  `GhidraJavaLaunch`, `GhidraJavaProbe`, `GhidraLauncher`; the bridge is the
  packaged `HeadlessScript` `ReaGhidraBridge.java`, with optional extension jars
  (`ReaGhidraNoReturnFix`, `ReaGhidraApplySeeds`, `ReaGhidraAnalysisOptions`,
  `ReaGhidraPrepareRaw`, `ReaGhidraPrepareCom`, MIPS soft-float, NativeAOT,
  PSP).
- Transport: `GhidraTransport` supports `unix-socket` and
  `authenticated-loopback-tcp`; when the socket path would exceed
  `sockaddr_un` limits (103 bytes on macOS, 107 on Linux) it relocates the
  socket into a private `/tmp/rea-ghidra-socket-*` root [ghidra-transport].
- Databases are ephemeral: REA does not modify executable bytes or control a
  GUI. Windows Ghidra support has no mutation authority (AGENTS.md).
- Profiles: `GhidraAnalysisProfile`, plus language/compiler-spec overrides
  (`REA_GHIDRA_LANGUAGE_ID`, `REA_GHIDRA_COMPILER_SPEC_ID`, which requires the
  language), disabled analyzers, seed files, and format-specific profiles
  (`GhidraMipsProfile`, `GhidraPeProfile`, `GhidraPowerPcProfile`,
  `GhidraRiscvProfile`, `GhidraDosSnapshot`, `GhidraRawBinaryLanguage`).
- Deadlines: `REA_GHIDRA_STARTUP_TIMEOUT_MS`; `GhidraRequestQueue` serializes
  requests; `GhidraDoctor` reports readiness.

## IDA

- REA does not embed IDA. `REA_IDA_MCP_CONFIG` names an absolute JSON
  registration for the upstream `ida-pro-mcp`, either
  `{type:"stdio", command, args, env}` or `{type:"http", url, headers}` with a
  loopback URL and no credentials in the URL [ida-config].
- `mode` is `attached` (default) or `headless`; `protocol: "native"` requires
  attached mode. Request timeout defaults to 300,000 ms and is capped at the
  Node timer maximum.
- IDA analysis is read-only: REA never saves or closes an attached GUI
  database (AGENTS.md).

## Kuna

- `KunaProvider` drives the caller's Kuna decompiler as an owned command-line
  tool: install discovery (`KunaInstallation`), project export/cache
  (`KunaProjectExport`, `KunaProjectCache`, `KunaProjectReader`), reference and
  string queries, run slots (`KunaRunSlots`) and a doctor report.
- Configuration is validated in `KunaConfiguration`: `REA_KUNA_COMMAND` must be
  an absolute path, `REA_KUNA_CACHE_DIR` an absolute directory,
  `REA_KUNA_CACHE_MAX_BYTES` bounds the cache, and
  `REA_KUNA_TIMEOUT_MS` / `REA_KUNA_PROJECT_TIMEOUT_MS` bound calls.
- Regex-heavy queries use the shared regex worker with
  `REA_REGEX_MATCHING_DEADLINE_MS` and `REA_REGEX_WORKER_HEAP_REQUEST_MIB`.

## JEB

- `JebProvider` connects to a **caller-started** JEB client that serves MCP at
  `REA_JEB_MCP_URL` (default `http://127.0.0.1:8425/mcp`) [jeb-provider]; REA
  never installs or launches JEB.
- `REA_JEB_MCP_TIMEOUT_MS` bounds catalog discovery *and* each `tools/call`; it
  is parsed at the optional-provider boundary so an invalid value cannot block
  unrelated startup.

## Debuggers and dynamic engines

- `start_gdb_session` / `gdb_session_status` / `close_gdb_session` and
  `debug-with-gdb` wrap persistent GDB/MI sessions (`src/gdb/`), with owned
  process lifecycle and target state.
- `start_rizin_debug_session` and `debug-with-rizin` wrap persistent Rizin
  sessions (`src/rizin/`); `execute-rizin-command` runs the static profile.
- `inspect-with-objdump` and `list-objdump-architectures` expose a GNU
  objdump command profile.
- Cutter tools (`list_cutter_sessions`, `cutter_command`) talk to a bridge
  service with capacity and request limits (`CutterBridgeClient`,
  `CutterBridgeService`).
- Frida tools (`list_frida_devices`, `list_frida_processes`,
  `instrument_with_frida`, plus `start/load/resume/unload/close_frida_session`)
  are instrumentation, not read-only analysis; the port is injected into the
  CLI and server separately from the provider registry.

## Cross-cutting rules

- Every engine integration reports effects truthfully (`launchesProcess`,
  `mutatesArtifact`, `mayShowUi`, `mayAccessNetwork`, …) in its capability
  descriptors; the MCP session and `binary_session` surface them.
- REA does not install or update Ghidra, IDA or JEB. Hopper is the only engine
  setup may install, and only after explicit approval.
- A failed cleanup is retained for retry; provider cleanup errors are
  aggregated, never silently dropped.
- Provider startup deadlines are independent of the MCP client's own tool
  deadline; the recovery instructions in the server tell the agent not to
  confuse the two.

## Related pages

- [Providers and sessions](/concepts/modules-providers.md)
- [Process supervision](/concepts/modules-process-supervision.md)
- [MCP server](/concepts/modules-mcp-server.md)
- [Build, run, test and deploy](/concepts/build-run-test-deploy.md)
