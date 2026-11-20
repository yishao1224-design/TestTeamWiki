---
type: Concept
title: MCP server adapter — transport, tools, delivery and identity
description: How REA serves its tool catalog over MCP: entry and startup modes, server classes, tool registration families, result delivery budgets, schema profiles, untrusted-content marking and the catalog identity tools report.
status: draft
lifecycle: draft
confidence: medium
created: 2026-11-20
updated: 2026-11-20
tags: [unverified]
sources:
  - resource: /sources/2026-11-20-mcp-server-wiring.md
    id: wiring-capture
    title: MCP server wiring capture (2026-11-20)
  - resource: https://github.com/yishao1224-design/rea/blob/8c4f96ef4759fd78da66c4b9726825ca71d7ee0c/src/main.ts
    id: main-entry
    title: src/main.ts
  - resource: https://github.com/yishao1224-design/rea/blob/8c4f96ef4759fd78da66c4b9726825ca71d7ee0c/src/main/deferredRuntime.ts
    id: deferred-runtime
    title: src/main/deferredRuntime.ts
  - resource: https://github.com/yishao1224-design/rea/blob/8c4f96ef4759fd78da66c4b9726825ca71d7ee0c/src/server/EvidenceMcpServerCore.ts
    id: server-core
    title: src/server/EvidenceMcpServerCore.ts
  - resource: https://github.com/yishao1224-design/rea/blob/8c4f96ef4759fd78da66c4b9726825ca71d7ee0c/src/server/toolResultEncoding.ts
    id: result-encoding
    title: src/server/toolResultEncoding.ts
  - resource: https://github.com/yishao1224-design/rea/blob/8c4f96ef4759fd78da66c4b9726825ca71d7ee0c/src/catalogIdentity.ts
    id: catalog-identity
    title: src/catalogIdentity.ts
  - resource: https://github.com/yishao1224-design/rea/blob/8c4f96ef4759fd78da66c4b9726825ca71d7ee0c/src/server/http/serveStreamableHttp.ts
    id: http-transport
    title: src/server/http/serveStreamableHttp.ts
  - resource: https://github.com/yishao1224-design/rea/blob/8c4f96ef4759fd78da66c4b9726825ca71d7ee0c/docs/mcp-contracts.md
    id: mcp-doc
    title: docs/mcp-contracts.md
generated:
  by: dsh/agent
  at: 2026-11-20
---

# MCP server adapter

**Job.** Serve REA's analysis capabilities to an MCP client as a stable,
schema-described tool catalog, translate each call into a shared application
workflow, and deliver results that stay within the client's wire budget while
marking target-derived content as untrusted.

## Key files

| Path | Role |
| --- | --- |
| `src/main.ts` | entry point; config → deferred/eager; stdio and `--http` entrypoints; shutdown latching |
| `src/main/eagerRuntime.ts` | session + transport + shutdown composition |
| `src/main/deferredRuntime.ts` | recorded-catalog serving and on-first-request materialization [deferred-runtime] |
| `src/main/transport.ts` | optional provider loading, per-instance server ownership, cleanup aggregation |
| `src/main/shutdown.ts`, `startup.ts`, `processShutdown.ts` | lifecycle helpers |
| `src/server/EvidenceMcpServer.ts` | canonical `EvidenceMcpServer` (subclass of the core) |
| `src/server/EvidenceMcpServerCore.ts` | SDK wrapper: tool registration, input/output schema presentation, result replacement, request tracking [server-core] |
| `src/server/DeferredEvidenceMcpServer.ts` | catalog-only server that materializes the real runtime |
| `src/server/RequestTrackingMcpServer.ts` | tracks pending `tools/call` so shutdown can wait |
| `src/server/register*Tools.ts` | ~60 registration modules, one per capability family |
| `src/server/ToolResultDelivery.ts`, `toolResultEncoding.ts`, `toolResult.ts` | budgets, encoding, continuation hints [result-encoding] |
| `src/server/http/*` | streamable HTTP listener, `node:http` bridge, request ownership |
| `src/catalogIdentity.ts` | counts and SHA-256 catalog digests [catalog-identity] |

## Startup modes

1. **Deferred (default for stdio).** When `dist/mcp-catalog.json` (or
   `-full.json`) exists and `RECORDED_MCP_CATALOG_SHA256` is present, stdio
   startup serves the recorded schemas without constructing analysis services.
   The first tool or prompt request calls `materialize()`, which runs
   `runEager` against the *same* SDK server [wiring-capture].
2. **Eager.** Used when no recorded catalog exists, the transport is HTTP, an
   initial `HOPPER_TARGET_PATH` is set, output schemas are suppressed
   (`REA_MCP_OUTPUT_SCHEMA_PROFILE=off`), or the caller injected providers.
3. **HTTP.** `rea mcp --http` parses `MCP_HOST`/`MCP_PORT`, serves streamable
   HTTP, and requires every server instance to be a
   `RequestTrackingMcpServer` so the listener cannot race tool completion
   [http-transport].

## Interfaces

### Tools

Tool registration is per family, each module binding named contracts:

`registerOfficialTools`, `registerEnhancedTools`, `registerNativeTools`,
`registerArtifactTools`, `registerManagedTools`, `registerManagedWorkflowTools`,
`registerReverseEngineeringTools`, `registerSessionTools`,
`registerAnalysisViewTool`, `registerBinaryDiagnosticsTools`,
`registerFunctionComparisonTool`, `registerArtifactComparisonTool`,
`registerBundleComparisonTool`, `registerProcessComparisonTool`,
`registerCaptureTools`, `registerSignalTools`, `registerBrowserTools`,
`registerBrowserScenarioTool`, `registerElectronTools`,
`registerJavaScriptRuntimeObservationTools`, `registerWebScriptTool`,
`registerWebModuleTool`, `registerWebSourceLocationTool`,
`registerWebRuntimeTools`, `registerWebNetworkCaptureTool`,
`registerJavaScriptRecoveryTool`, `registerApplicationTools`,
`registerAndroidTools`, `registerApktoolTools`, `registerAdbTools`,
`registerPcapTools`, `registerFlutterTools`, `registerJebTools`,
`registerFirmwareTools`, `registerWasmTools`, `registerApkSignatureTools`,
`registerEvmTools`, `registerSqliteTools`, `registerGoTools`,
`registerIl2cppTools`, `registerRustTools`, `registerPythonPycTool`,
`registerErlangTools`, `registerModelArtifactTools`,
`registerGpuCodeObjectsTools`, `registerBinaryDiagnosticsTools`,
`registerDesktopApplicationTool`, `registerFridaTools`, `registerCutterTools`,
`registerGdbTools`, `registerRizinDebugTools`, `registerSessionRecordTools`,
`registerSessionStatusTool`, `registerCloseLifecycleTool`.

Session-level tools (`src/contracts/sessionToolContracts.ts`) are the
lifecycle backbone: `open_binary`, `open_bundle_resources`, `close_binary`,
`binary_session`, `export_evidence_bundle`, `import_evidence_bundle`,
`get_evidence_bundle`, `capture_process_scenario`,
`compare_process_captures`, `compare_artifacts`, `compare_functions`,
`compare_bundles`, `get_navigation_context`, `inspect_address_context`.

### Prompts

Six guided prompts are registered from `src/contracts/promptContracts.ts`:
`investigate_feature`, `compare_application_versions`, `verify_reconstruction`,
`trace_crash`, `audit_residual_unknowns`, `prepare_bounded_process_capture`.
They are optional: `docs/tool-design.md` tells contributors to prefer direct
tools over prescribed sequences.

### Result delivery

- `REA_MCP_MAX_RESPONSE_BYTES` sets the response budget; it defaults to the
  SDK's `STDIO_DEFAULT_MAX_BUFFER_SIZE` [result-encoding].
- Successful results repeat JSON as `structuredContent` and escaped text; error
  results carry text plus `isError`.
- Oversize results are projected through `mcpResponseOverflowProjection` rather
  than truncated mid-JSON; native analysis results offer an
  `inspect_analysis_view` continuation (`retainedEvidenceContinuation`).
- Results whose content comes from the target carry
  `_meta.provenance.untrusted_content = true` and
  `source: "analysis_target"` [result-encoding].

### Schema profiles

| Setting | Values | Effect |
| --- | --- | --- |
| `REA_MCP_INPUT_SCHEMA_PROFILE` | `compact` (default) / `full` | compact presentation stays within `COMPACT_INPUT_SCHEMA_BUDGET_BYTES`; the full profile advertises complete field guidance |
| `REA_MCP_OUTPUT_SCHEMA_PROFILE` | `full` / `off` | `off` removes client-facing output schemas from `tools/list` while SDK registrations stay canonical |

Both profiles keep the same tool set and canonical server-side validation
[server-core].

### Catalog identity

`createCatalogIdentity()` reports `counts` (`cli_commands`, `mcp_tools`,
`mcp_prompts`) and lazily computed `digests`
(`tools_sha256`, `prompts_sha256`, `combined_sha256`) [catalog-identity].
The digest covers every field that changes what a client receives, including
`returns_target_content`; this is why a server that predates the
untrusted-content framing must not digest equal to a current one. Digests are
recorded at build time (`scripts/record-catalog-digests.mjs`) for the four
supported profile combinations and recomputed lazily for source runs or custom
budgets.

`binary_session` is documented as the runtime authority for catalog identity
and current tool availability; docs are explicitly not authoritative
[mcp-doc].

## Dependencies

- MCP SDK: `@modelcontextprotocol/server` (`serveStdio`, `McpServer`, JSON-RPC
  message types).
- Inward: `src/contracts` (schemas), `src/domain` (errors, evidence),
  `src/composition` (services), `src/application/investigation`
  (evidence writers).
- Outward: `src/main` transport and shutdown, pino logger, Node HTTP server.

## Pitfalls

- Deferred and eager servers must behave identically; the recorded catalog must
  be regenerated whenever tool schemas change (`npm run compile` records it,
  `npm run mcp-catalog:check` verifies it).
- Undeclared behavior changes in a contract field that affects delivery
  invalidate aligned-catalog checks.
- A request cancelled by the client must still release its pending entry; the
  core exports `forgetPendingMcpRequest` for that path [server-core].
- HTTP servers must track tool completion; `main.ts` throws a `TypeError`
  otherwise.

## Related pages

- [Contracts and tool surface](/concepts/modules-contracts.md)
- [Providers and sessions](/concepts/modules-providers.md)
- [Data model](/concepts/data-model.md)
- [Build, run, test and deploy](/concepts/build-run-test-deploy.md)
