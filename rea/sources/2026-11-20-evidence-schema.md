---
type: Source
title: Evidence, bundle and unknown schemas (capture, 2026-11-20)
description: The canonical Evidence record schema, its semantic identity rule, the deterministic EvidenceBundle projection and the ResidualUnknown revision record that together form REA's persistence model.
status: draft
lifecycle: draft
created: 2026-11-20
updated: 2026-11-20
source_url: https://github.com/yishao1224-design/rea/blob/8c4f96ef4759fd78da66c4b9726825ca71d7ee0c/src/domain/evidence.ts
retrieved: 2026-11-20
tags: [unverified]
generated:
  by: dsh/agent
  at: 2026-11-20
---

# Evidence schemas (capture)

Excerpts of `src/domain/evidence.ts` (474 lines), `src/domain/evidenceBundle.ts`
(643 lines) and `src/domain/residualUnknown.ts` (402 lines). Field lists and
vocabularies are verbatim; surrounding prose is summarised.

## Evidence record (`evidence.ts`, lines 76-178)

```ts
const evidenceBaseSchema = z.object({
  evidence_id: prefixedDigestSchema("ev"),
  subject: subjectSchema.nullable(),          // name, digest.sha256, local_path, format, architecture
  provider: providerSchema,                    // { id, name, version }
  analysis_profile: analysisProfileSchema.nullable(),
  predicate_type: z.string().min(1),           // default "rea.analysis"
  operation: z.string().min(1),
  parameters: jsonObjectSchema,
  raw_result: jsonValueSchema.nullable(),
  normalized_result: jsonValueSchema,
  confidence: z.enum(["observed", "derived", "inferred"]),
  authority: evidenceAuthoritySchema,
  environment: executionEnvironmentSchema.nullable(),
  limitations: z.array(z.string()),
  relationship_coverage: relationshipCoverageSchema.nullable(),
  locations: z.array(evidenceLocationSchema),
  evidence_links: z.array(prefixedDigestSchema("ev")),
}).strict();
```

Vocabularies owned by Evidence:

- `authority`: `shipped-artifact`, `controlled-replay`, `historical-reference`,
  `external-service`, `analyst-inference` (lines 117-123).
- `subject.format`: `hopper`, `analysis-database`, `mach-o`, `elf`, `pe`,
  `dos-mz`, `dos-com`, `raw-binary`, `zip`, `ipa`, `apk`, `msix`, `appx`,
  `hap`, `hsp`, `app-pack`, `asar`, `godot-pck`, `dmg`, `pkg`, `plist`,
  `javascript`, `source-map`, `directory`, `file`, `unknown`,
  `mach-o-universal`, `javascript-bundle`, `entitlements`, `pyinstaller`
  (lines 44-75).
- `locations.kind`: `address`, `address-range`, `artifact-path`, `file-offset`,
  `file-offset-range` (lines 97-114).
- `environment`: `{ id, platform, architecture, isolation }` with `isolation`
  in `none|process|container|virtual-machine` (lines 125-130).

## Semantic identity (`evidence.ts`, lines 252-289)

`evidence_id` is `ev_<sha256>` over a canonical projection that keeps
`subject.digest/format/architecture`, `provider`, `analysis_profile`,
`predicate_type`, `operation`, `parameters`, `raw_result`,
`normalized_result`, `confidence`, `authority`, `environment`, `limitations`,
`relationship_coverage`, `locations` and `evidence_links`. Display-only
subject fields (`name`, `local_path`) are excluded, so rebinding a display
path does not change identity. `parseEvidence` rejects a syntactically valid
record whose ID does not match its content; `immutableEvidence` additionally
freezes the reachable JSON.

## EvidenceBundle projection (`evidenceBundle.ts`, lines 46-161)

```ts
export const evidenceBundleSchema = z.strictObject({
  artifacts: z.array(artifactManifestSchema),      // digest, format, architecture
  providers: z.array(providerManifestSchema),
  environments: z.array(environmentManifestSchema),
  scenarios: z.array(scenarioManifestSchema),      // evidence_id, operation, authority: "controlled-replay"
  captures: z.array(captureManifestSchema),        // evidence_id, predicate_type
  unknowns: z.array(residualUnknownSchema),
  records: z.array(evidenceSchema),
});
```

`createEvidenceBundle` sorts records by `evidence_id` (Unicode code-point
order), deduplicates manifests, and derives `scenarios` from
`authority === "controlled-replay"` and `captures` from
`predicate_type === "rea.process-capture"`. Unknowns sort by
`unknown_id`, then `revision`. The comment states the bundle order has no
semantics. `evidenceBundleForTarget` filters to one artifact SHA-256 while
retaining mutation Evidence.

## ResidualUnknown revision (`residualUnknown.ts`, lines 79-119)

```ts
const residualUnknownObjectSchema = z.strictObject({
  unknown_id: prefixedDigestSchema("unk"),       // unk_<digest of domain+question+scope>
  revision: z.int().min(1),
  previous_revision_digest: /^[a-f0-9]{64}$/ | null,
  revision_digest: /^[a-f0-9]{64}$/,
  scope_digest: /^[a-f0-9]{64}$/ | null,
  question: text,
  status: z.enum(["open", "investigating", "blocked", "contradicted", "resolved"]),
  severity: z.enum(["low", "medium", "high", "critical"]),
  domain: text,
  comparison_conclusion_scope: /* optional */,
  supporting_evidence_ids: z.array(evidenceIdSchema),
  contradicting_evidence_ids: z.array(evidenceIdSchema),
  required_authority: /* authority | null */,
  required_confidence: z.enum(["observed", "derived", "inferred"]),
  required_environment: environmentRequirementSchema.nullable(),
  recommended_probes: z.array(z.strictObject({ operation: text, rationale: text })),
  relationships: z.array(z.strictObject({ type: z.enum(["depends-on","related-to","contradicts"]), unknown_id })),
  resolution: z.discriminatedUnion("disposition", [
    { disposition: "verified", rationale, evidence_ids: min 1 },
    { disposition: "withdrawn" | "out-of-scope", rationale, evidence_ids: [] },
  ]).nullable(),
  mutation_evidence_ids: z.array(evidenceIdSchema).min(1),
});
```

`unknown_id` is derived from `domain`, whitespace-normalised `question`,
`scope_digest` and optional comparison scope, so the same question about the
same scope reuses one identity across revisions.

## Analysis snapshot envelope (`analysisSnapshot.ts`, lines 177-183)

```ts
export const analysisSnapshotSchema = z.object({
  target: targetSchema,          // path, sha256, kind, format, architecture, entry point
  binding: bindingSchema,        // committed provider identity + profile
  entries: z.array(entrySchema),            // query_id, operation, parameters, execution
  workflow_entries: z.array(workflowEntrySchema),
  evidence_bundle: evidenceBundleSchema,
});
```

Snapshot files carry the same `target`, `binding` and `evidence_bundle` header
(`snapshotFileHeaderSchema`, lines 250-254) so a saved snapshot authenticates
before its entries are replayed.
