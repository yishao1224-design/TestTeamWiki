---
type: Source
title: MCP server wiring (capture, 2026-11-20)
description: How the MCP entry point chooses deferred or eager startup, composes the BinarySession, starts the transport and registers tool families in createServer.
status: draft
lifecycle: draft
created: 2026-11-20
updated: 2026-11-20
source_url: https://github.com/yishao1224-design/rea/blob/8c4f96ef4759fd78da66c4b9726825ca71d7ee0c/src/server/createServer.ts
retrieved: 2026-11-20
tags: [unverified]
generated:
  by: dsh/agent
  at: 2026-11-20
---

# MCP server wiring (capture)

Excerpts of `src/main.ts` (206 lines), `src/main/eagerRuntime.ts` (70 lines),
`src/main/deferredRuntime.ts` (243 lines), `src/main/transport.ts` (214 lines)
and `src/server/createServer.ts` (1,231 lines).

## Entry selection (`main.ts`, lines 87-114)

```ts
export const run = async (dependencies = runtimeDependencies()): Promise<number> => {
  const environment = snapshotEnvironment(dependencies.env);
  const config = parseConfig(environment);
  if (!config.ok) { dependencies.writeStderr(...); return 1; }
  const fullOutputCatalog =
    environment.REA_MCP_OUTPUT_SCHEMA_PROFILE === undefined ||
    environment.REA_MCP_OUTPUT_SCHEMA_PROFILE === "full";
  if (RECORDED_MCP_CATALOG_SHA256 !== undefined &&
      dependencies.transport !== "http" &&
      config.value.hopperTargetPath === undefined &&
      fullOutputCatalog &&
      dependencies.createServer === undefined &&
      dependencies.loadOptionalProviders === undefined) {
    return runDeferred({ ...dependencies, env: environment }, config.value);
  }
  return (await import("./main/eagerRuntime.js")).runEager(...);
};
```

Deferred startup applies when a recorded catalog exists, the transport is
stdio, no initial `HOPPER_TARGET_PATH` is set and output schemas are not
suppressed. `runEntrypoint` redacts unexpected startup causes behind
`SERVER_START_FAILED` (lines 117-133).

## Deferred startup (`deferredRuntime.ts`, lines 24-46, 177-243)

`readCatalog(profile)` reads `../mcp-catalog.json` or
`../mcp-catalog-full.json`, hashes the bytes and compares them with
`RECORDED_MCP_CATALOG_SHA256[profile]`; a mismatch throws
`SERVER_START_FAILED`. `runDeferred` serves those schemas from
`DeferredEvidenceMcpServer` and materializes the real runtime on the first tool
request by calling `runEager` with a `createServer` override that reuses the
same SDK server (`initializeRuntime`, lines 48-114).

## Eager startup (`eagerRuntime.ts`, lines 24-70)

```ts
export const runEager = async (dependencies, selectedConfig?): Promise<number> => {
  const environment = snapshotEnvironment(dependencies.env);
  const config = selectedConfig === undefined ? parseConfig(environment) : ok(selectedConfig);
  if (!config.ok) { dependencies.writeStderr(...); return 1; }
  const logger = createLogger("mcp", config.value.logLevel);
  const serverLogger = logger.child({ layer: "server" });
  const session = createBinarySession(config.value, logger, environment);
  const opened = await openInitialTarget(session, config.value, serverLogger, dependencies.writeStderr);
  if (!opened.ok) return opened.exitCode;
  const transport = await startMcpTransport(dependencies, session, {
    logger, serverLogger, environment,
    delivery: createToolResultDelivery(config.value.mcpMaxResponseBytes),
    loadOptionalProviders: dependencies.loadOptionalProviders ??
      (() => loadOptionalObservationProviders(createOptionalObservationFactories(environment))),
  });
  if (!transport.ok) return 1;
  const shutdown = createShutdown({
    handle: transport.handle, closeAndroid: transport.closeAndroid,
    closeOptionalProviders: transport.closeOptionalProviders, session, dependencies, serverLogger,
  });
  dependencies.captureShutdown?.(shutdown.shutdown);
  return 0;
};
```

`openInitialTarget` (`main/startup.ts`) returns `{ ok: true }` when
`config.hopperTargetPath` is undefined; otherwise it opens the target and
closes the session on failure.

## Transport (`transport.ts`, lines 32-213)

- Optional providers load first; a failure logs a warning and leaves affected
  tools unavailable instead of failing the server (lines 46-64).
- `OwnedMcpServers` owns every server instance the SDK factory creates,
  including a discarded discovery probe (lines 66-74, 110-139).
- `createServer` is called per instance with `{ kind: "session", session }`,
  the shared logger, environment, delivery and optional providers
  (lines 119-131).
- Startup failure runs `session.close()`, `closeAndroid()`,
  `closeServers()` and `closeOptionalProviders()` with `Promise.allSettled`
  and reports each failure separately (lines 140-190).
- `close` awaits `prepareShutdown` before closing the transport and servers
  (lines 197-209).

## Server construction (`createServer.ts`, lines 392-421)

```ts
const createMcpServer = (session, delivery, compactPresentation, omitOutputSchemas) =>
  new EvidenceMcpServer(
    { name: PRODUCT_IDENTITY.mcpServerKey, version: PRODUCT_IDENTITY.packageVersion },
    { capabilities: {}, instructions: `${session === undefined ? ACTIVE_TARGET_INSTRUCTIONS : TARGET_FREE_INSTRUCTIONS} ${RECOVERY_INSTRUCTIONS} ${UNTRUSTED_CONTENT_INSTRUCTIONS}` },
    session === undefined ? undefined : (evidence) => session.recordEvidence(evidence),
    delivery, compactPresentation, omitOutputSchemas,
    createCatalogIdentity(compactPresentation, omitOutputSchemas ? "off" : "full"),
  );
```

The two instruction constants (lines 195-204) tell the agent that results
include inline Evidence with coverage and limitations, that an opened target
persists across turns, to `close_binary` when finished, and how to distinguish
a REA provider deadline from a host deadline. `RECOVERY_INSTRUCTIONS` is long
and explicitly forbids repeating an unchanged call.

## Construction pattern (`createServer.ts`, lines 428-830)

`createServer(source, options)`:

1. Snapshots the environment, resolves output-schema and delivery profiles.
2. Creates every service: `evmInterfaceModule`, `createSqliteDatabaseService`,
   `createBinaryLayoutService`, `createRecordedCrashService`,
   `createWebSourceLocationService`, `createWebModuleTraceService`,
   `createDesktopApplicationService`, `createGoBinaryService`,
   `createRustBinaryService`, `createBeamService`,
   `createModelArtifactService`, `createGpuCodeObjectsService`,
   `createIl2cppMetadataService`, PE/ELF services, `createElectronFuseService`,
   `createPythonPycService`, `createWebNetworkCaptureService`,
   `createJavaScriptRecoveryProvider`, `ArtifactResourceScope`,
   `createReverseEngineeringService`, `FirmwareAnalysisService`,
   `GdbSessionManager`, `RizinDebugSessionManager`,
   `createAndroidAnalysisProvider`, `createPcapAnalysisProvider`,
   `createFlutterBuildAnalysisProvider`, `FridaInstrumentationManager`,
   `wasmArtifactModule`, `apkSignatureModule`, `createPacketCaptureService`,
   `createSignalRecordingService`, `createWebRuntimeService`,
   `createApktoolResourceAnalysisProvider`, `createJebAnalysisProvider`,
   `createAdbDeviceAnalysisProvider`, `CutterBridgeService`.
3. Builds `toolContext` with `evidenceById`, `activeTarget`, `recordEvidence`,
   `recordEvidenceWithUnknown` and `withAdmittedAnalysis`.
4. Registers tool families (`registerDesktopApplicationTool`,
   `registerModelArtifactTools`, `registerGpuCodeObjectsTools`,
   `registerBinaryAnalysisTools`, `registerReverseEngineeringTools`,
   `registerFridaTools`, `registerCutterTools`, `registerGdbTools`,
   `registerRizinDebugTools`, `registerCaptureTools`, `registerSignalTools`,
   `registerConfiguredAnalysisTools`, `registerAdbTools`, `registerPcapTools`,
   `registerFlutterTools`, `registerApktoolTools`, `registerObservationTools`,
   `registerGuidedPrompts`, `registerSessionTools`).
5. Wraps `server.close` and `server.server.onclose` so provider cleanup runs
   once and failures are aggregated (`closeProviders`, lines 544-609).

`installSessionToolAvailability` (lines 268-390) publishes a host-verified
availability policy per optional capability; injected services are assumed to
own their own availability, and `evmLimiterAvailability` is exposed only when
EVM is host-supported and not injected.
