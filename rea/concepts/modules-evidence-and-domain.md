---
type: Concept
title: Evidence and the domain layer — records, ledger, unknowns and comparisons
description: The pure domain layer that defines Evidence, analysis profiles, snapshots, bundles, residual unknowns, error algebra and comparison semantics, plus the session-owned ledger that stores and authenticates them.
status: draft
lifecycle: draft
confidence: medium
created: 2026-11-20
updated: 2026-11-20
tags: [unverified]
sources:
  - resource: /sources/2026-11-20-evidence-schema.md
    id: evidence-capture
    title: Evidence, bundle and unknown schemas (2026-11-20)
  - resource: https://github.com/yishao1224-design/rea/blob/8c4f96ef4759fd78da66c4b9726825ca71d7ee0c/src/domain/evidence.ts
    id: evidence
    title: src/domain/evidence.ts
  - resource: https://github.com/yishao1224-design/rea/blob/8c4f96ef4759fd78da66c4b9726825ca71d7ee0c/src/domain/analysisProfile.ts
    id: profile
    title: src/domain/analysisProfile.ts
  - resource: https://github.com/yishao1224-design/rea/blob/8c4f96ef4759fd78da66c4b9726825ca71d7ee0c/src/domain/residualUnknown.ts
    id: unknown
    title: src/domain/residualUnknown.ts
  - resource: https://github.com/yishao1224-design/rea/blob/8c4f96ef4759fd78da66c4b9726825ca71d7ee0c/src/application/investigation/EvidenceLedger.ts
    id: ledger
    title: src/application/investigation/EvidenceLedger.ts
  - resource: https://github.com/yishao1224-design/rea/blob/8c4f96ef4759fd78da66c4b9726825ca71d7ee0c/src/application/investigation/InvestigationRecords.ts
    id: records
    title: src/application/investigation/InvestigationRecords.ts
  - resource: https://github.com/yishao1224-design/rea/blob/8c4f96ef4759fd78da66c4b9726825ca71d7ee0c/src/domain/analysisSnapshot.ts
    id: snapshot
    title: src/domain/analysisSnapshot.ts
  - resource: https://github.com/yishao1224-design/rea/blob/8c4f96ef4759fd78da66c4b9726825ca71d7ee0c/src/server/registerEvidenceTools.ts
    id: evidence-tools
    title: src/server/registerEvidenceTools.ts
generated:
  by: dsh/agent
  at: 2026-11-20
---

# Evidence and the domain layer

**Job.** Define what a finding *is*, what it is allowed to claim, how it is
identified, and how findings are stored, exported, imported and compared — as
pure data modules with no engine or process knowledge.

`src/domain/` is ~886 files of side-effect-free code. `/src/domain` has no
imports from adapters or application services; the layering rule is enforced by
`npm run verify:module-boundaries` (part of `lint`).

## Key files

| Area | Files |
| --- | --- |
| Evidence record | `evidence.ts`, `evidenceValidation.ts`, `evidenceErrors.ts`, `evidenceRecordSummary.ts`, `evidenceReference.ts`, `digests.ts`, `canonicalDigest.ts`, `canonicalJsonDigestSteps.ts` |
| Analysis commitment | `analysisProfile.ts` [profile] |
| Bundle | `evidenceBundle.ts`, `evidenceCompletionLedger.ts` |
| Unknowns | `residualUnknown.ts`, `unknownRegistryError.ts` |
| Snapshot | `analysisSnapshot.ts` (904 lines) [snapshot] |
| Comparison | `artifactComparison.ts`, `bundleComparison.ts`, `functionComparison*.ts`, `callPath.ts`, `changedBehavior.ts`, `comparisonSemantics.ts` |
| Reconstruction | `reconstructionCoverage*`, `reconstructionObligationLedger*`, `reconstructionReadiness*`, `reconstructionVerification*`, `reconstructionUnknowns.ts` |
| Format value types | `binaryTarget*.ts`, `rawBinary.ts`, `dosMz.ts`, `dosCom.ts`, `artifactGraph.ts`, `native/*`, `javascript/*`, `web*.ts` |
| Errors | `analysisErrorBase.ts`, `analysisErrorCore.ts`, `result.ts`, `analysisErrorProjection.ts` |
| Ledger (application) | `InvestigationRecords.ts`, `EvidenceLedger.ts`, `InvestigationEvidence.ts`, `UnknownEvidence.ts` [ledger] |

## Interfaces

### Evidence

`createEvidence(target, provider, observation)` builds the record, computes
`ev_<sha256>` over the semantic projection, and seals it immutable when the
result is already a sealed JSON tree [evidence-capture]. The observation's
optional fields let a provider state exactly what it knows:
`confidence`, `authority`, `limitations`, `relationshipCoverage`,
`locations`, `evidenceLinks`, `rawResult` and a
`subjectUnavailableReason` used when no target identity exists.

- `parseEvidence` rejects an ID/content mismatch (tamper detection).
- `immutableEvidence` freezes the reachable JSON and remembers it in a
  `WeakMap`, so later reads are not re-parsed.
- `rebindEvidenceToTargetPath` may change only the display subject, and only
  when digest + format + architecture match.

### Analysis profile

A commitment is `{ provider: {id,name,version}, parameters, digest }` where
`digest` is the canonical hash of provider + parameters [profile]. Equality
compares both digest and provider. The profile is resolved *before* a provider
client starts, and an Evidence record reporting a profile whose provider
differs from its `provider` field is rejected.

### Snapshot

`analysisSnapshotSchema` = `{ target, binding, entries[], workflow_entries[],
evidence_bundle }` [snapshot]. Entries carry `query_id`, `operation`,
`parameters` and an `execution` block, so a saved snapshot can replay eligible
reads and page retained Evidence without reopening the target. Snapshot files
repeat `target`, `binding` and `evidence_bundle` in a header that is
authenticated before entries are trusted.

### Bundle and ledger

`EvidenceBundle` is a deterministic projection: sorted records, deduplicated
manifests, and derived `scenarios`/`captures` arrays. `EvidenceLedger` [ledger]:

- `record` is idempotent: identical content returns `"duplicate"`, conflicting
  content at the same ID returns `EvidenceIntegrityError`.
- `import` validates the whole bundle, checks ID/revision conflicts, then
  commits atomically; a record-only bundle avoids rebuilding the unknown graph.
- Unknowns are stored as **revision histories** keyed by `(unknown_id,
  revision)`; `previous_revision_digest` chains them, and mutations are
  accompanied by `rea.residual-unknown-mutation` Evidence.

`InvestigationRecords` [records] is the facade the session holds; it implements
the narrow reader/writer ports that tool handlers receive
(`EvidenceReader`, `EvidenceWriter`, `EvidenceUnknownWriter`,
`UnknownRegistryPort`) so no tool gets lifecycle authority.

### Unknowns

A `ResidualUnknown` is a *question*, not a guess: immutable identity from
domain + normalized question + scope digest, status
(`open|investigating|blocked|contradicted|resolved`), severity, required
authority/confidence/environment, recommended probes, typed relationships
(`depends-on`, `related-to`, `contradicts`), and a resolution that must cite
Evidence for `verified`. `verifyUnknownResolution` checks registry consistency
and reports `truthVerified` separately — consistency is not truth.

### Comparison and reconstruction

- `compare_artifacts`, `compare_functions`, `compare_bundles`,
  `find_changed_behavior`, `build_call_path`,
  `correlate_static_and_runtime`, `verify_reconstruction` are structured
  comparisons over Evidence, each with its own dimension/coverage schema
  (`functionComparisonDimensions`, `comparisonSemantics`,
  `comparisonStatus`, `relationshipCoverage`).
- Reconstruction tooling (`docs/reconstruction-coverage.md`,
  `reconstruction-obligation-ledgers.md`, `reconstruction-readiness.md`)
  tracks what a reimplementation has and has not covered; readiness verifiers
  are development conformance tools, not proof of behaviour.

### Evidence tools

`export_evidence_bundle`, `import_evidence_bundle`, `get_evidence_bundle`
(in-memory), `inspect_evidence_bundle` (file-based, CLI
`inspect-evidence-bundle`) and the `inspect_analysis_view` projections are the
caller-facing surface over this layer [evidence-tools].

## Dependencies

- `canonicalize` and Node `crypto` for RFC 8785-style canonical JSON and
  SHA-256 IDs.
- Zod for every schema; `.strict()` is the norm so unknown fields are rejected.
- No imports upward: domain modules must not import from `application/`,
  `server/` or provider directories.

## Pitfalls

- **Identity is content, not location.** Re-observing the same facts from the
  same provider yields the same `evidence_id`; adding a display path must not.
- **Absence is not a fact.** Missing coverage stays `unknown`; the domain has a
  dedicated `residualUnknown` model rather than assuming "none".
- **Sealing is expensive.** `createEvidence` avoids re-parsing sealed payloads;
  re-parsing a 300 MB bundle to check a digest is a real performance bug.
- **Order is not semantics.** Bundle arrays are sorted for determinism;
  consumers must not infer sequence from them.
- Comparing floats/JSON: `exactJsonNumber` and `jsonSerialization` exist because
  naive `JSON.stringify` comparisons lose value identity.

## Related pages

- [Data model](/concepts/data-model.md)
- [Providers and sessions](/concepts/modules-providers.md)
- [MCP server](/concepts/modules-mcp-server.md)
- [Conventions, pitfalls and open questions](/concepts/conventions-pitfalls-open-questions.md)
