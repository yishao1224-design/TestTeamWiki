---
type: Concept
title: Providers and sessions — selection, binding and the target lifecycle
description: The provider abstraction, deterministic deep-provider selection, auxiliary provider composition, and the BinarySession that owns one active target, its profile, its snapshot cache and its evidence records.
status: draft
lifecycle: draft
confidence: medium
created: 2026-11-20
updated: 2026-11-20
tags: [unverified]
sources:
  - resource: https://github.com/yishao1224-design/rea/blob/8c4f96ef4759fd78da66c4b9726825ca71d7ee0c/src/application/AnalysisProvider.ts
    id: provider-interface
    title: src/application/AnalysisProvider.ts
  - resource: https://github.com/yishao1224-design/rea/blob/8c4f96ef4759fd78da66c4b9726825ca71d7ee0c/src/application/binary/AnalysisProviderRegistry.ts
    id: registry
    title: src/application/binary/AnalysisProviderRegistry.ts
  - resource: https://github.com/yishao1224-design/rea/blob/8c4f96ef4759fd78da66c4b9726825ca71d7ee0c/src/application/binary/SessionProviderRouter.ts
    id: router
    title: src/application/binary/SessionProviderRouter.ts
  - resource: https://github.com/yishao1224-design/rea/blob/8c4f96ef4759fd78da66c4b9726825ca71d7ee0c/src/application/binary/BinarySession.ts
    id: session
    title: src/application/binary/BinarySession.ts
  - resource: https://github.com/yishao1224-design/rea/blob/8c4f96ef4759fd78da66c4b9726825ca71d7ee0c/src/application/binary/BinarySessionPort.ts
    id: session-port
    title: src/application/binary/BinarySessionPort.ts
  - resource: https://github.com/yishao1224-design/rea/blob/8c4f96ef4759fd78da66c4b9726825ca71d7ee0c/src/composition/binary.ts
    id: composition
    title: src/composition/binary.ts
  - resource: https://github.com/yishao1224-design/rea/blob/8c4f96ef4759fd78da66c4b9726825ca71d7ee0c/src/application/binary/AnalysisSnapshotCache.ts
    id: snapshot-cache
    title: src/application/binary/AnalysisSnapshotCache.ts
generated:
  by: dsh/agent
  at: 2026-11-20
---

# Providers and sessions

**Job.** Turn "analyse this target" into exactly one bound engine client, keep
that binding stable for the life of the target, and record every observation
the bound client produces.

## Key files

| Path | Role |
| --- | --- |
| `src/application/AnalysisProvider.ts` | `AnalysisProvider`, `AnalysisClient`, `CapabilityDescriptor`, `AnalysisExecution`, `createAnalysisExecution` [provider-interface] |
| `src/application/binary/AnalysisProviderRegistry.ts` | discovers candidates, resolves profiles, selects one deep provider [registry] |
| `src/application/binary/SessionProviderRouter.ts` | immutable route = deep binding + disjoint auxiliary providers [router] |
| `src/application/binary/CompositeProvider.ts` | one client over several providers by operation |
| `src/application/binary/BinarySession.ts` | single active target, serialized transitions, cleanup, snapshots [session] |
| `src/application/binary/BinarySessionPort.ts` | adapter-facing interface [session-port] |
| `src/application/binary/BinarySessionRecords.ts`, `AnalysisSnapshotCache.ts`, `BinarySessionStatus.ts` | record facade, snapshot reuse, status payload |
| `src/composition/binary.ts` | the only production wiring point [composition] |
| `src/application/binary/LazyAnalysisProvider.ts`, `DeferredAnalysisClient.ts` | defer optional/auxiliary loading until used |

## The provider contract

`AnalysisProvider` is a factory plus declaration [provider-interface]:

```ts
interface AnalysisProvider {
  identity(): ProviderIdentity;                 // { id, name, version }
  capabilities(): readonly CapabilityDescriptor[];
  resolveAnalysisProfile?(target, options?): Promise<Result<AnalysisProfileResolution, AnalysisError>>;
  createClient(target, profile?, context?): AnalysisClient;
}
```

A `CapabilityDescriptor` declares, per operation: availability (`available` +
reason), `effects` (`mutatesArtifact`, `launchesProcess`, `mayShowUi`,
`mayAccessNetwork`, `mayWriteFilesystem`, `changesPermissions`,
`requiresRoot`), `limitations`, optional `cachePolicy`
(`snapshot` | `live`) and optional `analysisProfile`.

A deep candidate additionally implements `resolveAnalysisProfile`,
`inspectAvailability`, `inspectTargetSupport` and optional
`inspectReadinessSnapshot`, `hasPendingOwnedResources`,
`closeOwnedResources` (`AnalysisProviderCandidate`).

`AnalysisClient` extends `AnalysisOperationPort` with
`executeIfReady`, `runtimeLineageSnapshots`, `requestActivitySnapshots`,
`reuseReadiness`, `operationHealthSnapshot` and `close`. `close` may take
`retainDocument` and `onCleanupFailure`, which is how cleanup failures reach the
calling operation while ownership is retained for retry.

`createAnalysisExecution(result, provider, options)` builds a validated
observation. Passing `rawResult` is explicitly for *a representation distinct
from the result*; omitting it records `null` rather than duplicating the payload.

## Selection

`AnalysisProviderRegistry` is constructed with deep candidates, a default
selector (`REA_ANALYSIS_PROVIDER`, default `auto`) and an optional
"can inspect without a deep provider" predicate [registry].

- Candidate status is a triple of `availability`, `targetSupport` and
  `capabilities`, with `unknown` states that are *not* treated as absence.
- Selection never falls back. With `auto`, exactly one usable candidate is
  selected (`selectionSource: "auto-single-candidate"`); more than one usable
  candidate is an `ambiguous` selection error; zero usable candidates yields a
  null binding plus the full candidate statuses, so the caller sees why each
  provider refused.
- Explicit selection records `selectionSource: "request"`; environment selection
  records `"environment"`.
- Registration order must not affect selection or output.
- Profiles are resolved *before* a client starts, and a reported profile must
  name the same provider as the Evidence record.

`SessionProviderRouter` combines the selected deep provider with auxiliary
providers, which must declare disjoint operations (constructing the router
throws `TypeError` on overlap or duplicate provider IDs). When more than one
provider participates, a `CompositeProvider` dispatches by operation; when none
is bound, an `emptyClient` answers `health` and rejects other operations with
`AnalysisCapabilityUnavailableError` [router].

## The session

`BinarySession` "owns the single active target shared by CLI and MCP adapters"
[session]. Behaviours visible from its code:

- Target transitions are serialized (`#transition` promise chain); a failed
  switch recreates the previous target rather than retaining a shut-down
  client.
- `#shutdownStarted` refuses later target acquisition once process shutdown
  began; in-flight calls are drained (`#drainCalls`) before a transition.
- External mutations are admitted through `withAdmittedExternalMutation`,
  which invalidates cached Evidence first and retires the target afterwards.
- Failed cleanup is retained (`#externalMutationCleanupFailure`) and reported
  with `cleanupIncomplete`, so a later call does not silently look clean.
- Bundle-resource roots (`openBundleResources`) are a separate lane with their
  own factory and cleanup queue.
- `recordEvidence` / `recordEvidenceWithUnknown` / `recordUnknown` delegate to
  `InvestigationRecords` (see [data model](/concepts/data-model.md)).
- `closeWithSnapshot(path, overwrite)` saves the snapshot and closes while
  excluding later provider requests; `exportAnalysisSnapshot` /
  `importAnalysisSnapshot` move snapshots without a provider.

`BinarySessionPort` is the adapter-facing subset: `execute`,
`executeIfProviderReady`, `withAdmittedAnalysis`,
`withAdmittedExternalMutation`, `open`, `openBundleResources`, `close`,
`closeWithSnapshot`, `status`, `activeTarget`, snapshot import/export,
`providerIdentity`, `analysisProfile`, `allowsSnapshotReplay`,
`invalidateSnapshotForExternalMutation`, `recordWorkflowSnapshot`, plus
availability/snapshot-change listeners [session-port].

## Composition

`createBinarySession(config, logger, environment)` builds a registry of
`HopperProvider`, `GhidraProvider`, `IdaProvider`, `KunaProvider`, wraps the
auxiliary declarations in `LazyAnalysisProvider`, and supplies a bundle-resource
factory that lazily imports `ArtifactProvider` and `NativeMacOSProvider`
[composition]. Variants exist for managed-only, artifact-only and native-only
sessions, so a static workflow never pays for engine candidates.

## Dependencies

- Inward: `src/contracts` (operation names), `src/domain`
  (binary target, evidence, errors, profiles), `src/providers`
  (client creation/cleanup helpers).
- Outward: engine adapters in `src/hopper`, `src/ghidra`, `src/ida`,
  `src/kuna`, `src/jeb`, static providers, and `src/process` for supervision.

## Pitfalls

- A provider that claims a capability but cannot attribute it to its own
  identity fails registry construction.
- Auxiliary operations must stay disjoint from deep operations; composing two
  providers with the same operation is a programming error, not a runtime
  fallback.
- Snapshot replay is per-operation (`allowsSnapshotReplay`); live state
  capabilities must not be replayed.
- Cleanup failures must be retained and surfaced; discarding them makes an
  owned resource leak look like success.

## Related pages

- [Native analysis providers](/concepts/modules-native-providers.md)
- [Static artifacts](/concepts/modules-static-artifacts.md)
- [Process supervision](/concepts/modules-process-supervision.md)
- [Architecture](/concepts/architecture.md)
