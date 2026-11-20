---
type: Concept
title: Data model — targets, evidence, profiles, bundles, unknowns and snapshots
description: The entities REA produces and stores, their identity rules, their relationships, and how they move between a session, a saved snapshot and an imported bundle.
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
  - resource: https://github.com/yishao1224-design/rea/blob/8c4f96ef4759fd78da66c4b9726825ca71d7ee0c/src/domain/binaryTargetTypes.ts
    id: target-types
    title: src/domain/binaryTargetTypes.ts
  - resource: https://github.com/yishao1224-design/rea/blob/8c4f96ef4759fd78da66c4b9726825ca71d7ee0c/src/domain/analysisProfile.ts
    id: profile
    title: src/domain/analysisProfile.ts
  - resource: https://github.com/yishao1224-design/rea/blob/8c4f96ef4759fd78da66c4b9726825ca71d7ee0c/src/domain/evidenceBundle.ts
    id: bundle
    title: src/domain/evidenceBundle.ts
  - resource: https://github.com/yishao1224-design/rea/blob/8c4f96ef4759fd78da66c4b9726825ca71d7ee0c/src/domain/residualUnknown.ts
    id: unknown
    title: src/domain/residualUnknown.ts
  - resource: https://github.com/yishao1224-design/rea/blob/8c4f96ef4759fd78da66c4b9726825ca71d7ee0c/src/domain/analysisSnapshot.ts
    id: snapshot
    title: src/domain/analysisSnapshot.ts
  - resource: https://github.com/yishao1224-design/rea/blob/8c4f96ef4759fd78da66c4b9726825ca71d7ee0c/src/application/investigation/EvidenceLedger.ts
    id: ledger
    title: src/application/investigation/EvidenceLedger.ts
generated:
  by: dsh/agent
  at: 2026-11-20
---

# Data model

REA's persistent vocabulary is small: a **target**, an **analysis profile**,
an **Evidence record**, a **residual unknown**, and the containers that group
them (**EvidenceBundle**, **AnalysisSnapshot**). Everything else — module
graphs, comparisons, capture records — is a typed `normalized_result` inside
Evidence.

## Entity map

```
BinaryTarget ──(opens)──> BinarySession ──(binds)──> AnalysisProfile
                             │
                             ├──(records)──> Evidence ──(summary of)──> EvidenceBundle
                             │                   │
                             │                   └──(links)──> Evidence
                             ├──(records)──> ResidualUnknown (revisions)
                             └──(saves)────> AnalysisSnapshot ──contains──> EvidenceBundle
```

## BinaryTarget

A target is identified by content, not by path: `{ path, sourcePath?,
bundleInfoPlist?, sha256 }`, discriminated by `kind` (`executable`,
`database`, `archive`, `artifact`) and `format` (`elf`, `mach-o`,
`mach-o-universal`, `pe`, `dos-mz`, `dos-com`, `raw-binary`, `zip`, `ipa`,
`apk`, `msix`, `appx`, `hap`, `hsp`, `app-pack`, `asar`, `godot-pck`, `dmg`,
`pkg`, `plist`, `javascript`, `source-map`, …) with an `architecture` from the
`BINARY_ARCHITECTURES` list [target-types].

Two rules visible in the types:

- `6502` and `m68k` are `RAW_ONLY_ARCHITECTURES`: header detection never yields
  them, so a detected target's architecture excludes them.
- Families whose ISA/ABI the header alone does not establish (`mips`, `riscv`,
  `powerpc`) carry explicit `ElfFamilyMetadata` (`elfClass`, byte order, type,
  flags) so consumers admit a lane from parsed facts rather than re-reading the
  header.

## AnalysisProfile

`{ provider: {id, name, version}, parameters, digest }`, where `digest` is the
canonical hash of provider + parameters [profile]. An Evidence record's
`analysis_profile`, when present, must name the same provider as its
`provider` field; the schema rejects a mismatch. Profiles are resolved before a
provider client starts, and equal profiles are what allows a snapshot replay to
be trusted.

## Evidence

The record described in the captured schema: subject (name, SHA-256, format,
architecture, local path), provider, profile, predicate type, operation,
parameters, raw and normalized results, confidence, authority, environment,
limitations, relationship coverage, locations and evidence links
[evidence-capture].

- **Identity:** `ev_<sha256>` over the semantic projection; display-only subject
  fields are excluded. `parseEvidence` rejects a mismatched ID.
- **Confidence** is epistemic (`observed`, `derived`, `inferred`); **authority**
  is provenance (`shipped-artifact`, `controlled-replay`,
  `historical-reference`, `external-service`, `analyst-inference`).
- **Limitations** are mandatory strings: an empty array means "none known to
  the producer", not "proven complete".
- **Relationship coverage** describes only the provider-observed direct
  relationship set (`complete` means unbounded *read*, not a claim about every
  runtime edge).

## ResidualUnknown

A question with immutable identity (`unk_<digest of domain + normalized
question + scope_digest>`), an optimistic revision chain, lifecycle status,
severity, required authority/confidence/environment, recommended probes, typed
relationships and a resolution that must cite Evidence for `verified`
[unknown]. Every revision is accompanied by `mutation_evidence_ids`, so the
question's history is itself evidence.

## EvidenceBundle

`{ artifacts, providers, environments, scenarios, captures, unknowns, records }`
[bundle]. It is a *projection*, not a container with its own identity:

- records sorted by `evidence_id`; manifests deduplicated;
- `scenarios` derived from `authority: "controlled-replay"`;
- `captures` derived from `predicate_type: "rea.process-capture"`;
- unknown revisions sorted by `unknown_id`, then `revision`.

`evidenceBundleForTarget(bundle, sha256)` filters a bundle to one artifact while
retaining the mutation evidence that explains its unknowns.

## AnalysisSnapshot

`{ target, binding, entries, workflow_entries, evidence_bundle }` [snapshot].
`binding` is the committed provider identity + profile. Entries are replay
candidates (`query_id`, `operation`, `parameters`, `execution`); a snapshot
file repeats `target`, `binding` and `evidence_bundle` in a header that is
authenticated first.

Why both bundle and snapshot exist: a bundle is portable **evidence** (import
into another session, compare, archive); a snapshot additionally carries the
**query bindings** that let REA answer repeated reads without reopening the
target.

## Session state

`BinarySession` holds the live entities plus caches:

| State | Owner | Cleared by |
| --- | --- | --- |
| active binding (target + profile + client) | `BinarySession` | target switch, `close_binary`, shutdown |
| evidence ledger | `InvestigationRecords` | `clear()`, process exit |
| snapshot cache | `AnalysisSnapshotCache` | external mutation, close |
| availability listeners | `BinarySession` | shutdown |

## Wire shapes

- **MCP tool result:** the canonical Evidence JSON (or a projection) inline,
  plus `_meta.provenance` when the content is target-derived.
- **CLI:** JSON on stdout (optionally through `--filter-output` or a token
  window), diagnostics on stderr.
- **`binary_session`:** counts, digests, active target, provider status and tool
  availability — the runtime authority for "what is this server".
- **Evidence bundle file:** the bundle schema above, with `records` sorted and
  every record self-authenticating.

## Invariants worth remembering

1. Content identity, not location: paths are display metadata.
2. Absence is unknown, not "none".
3. Observed, derived and inferred are distinct and must not be flattened.
4. Bundle order has no semantics.
5. A resolution without Evidence is not a resolution.
6. A snapshot replay is only valid for the same target identity **and** the same
   analysis profile.

## Related pages

- [Evidence and the domain layer](/concepts/modules-evidence-and-domain.md)
- [Providers and sessions](/concepts/modules-providers.md)
- [Contracts and tool surface](/concepts/modules-contracts.md)
- [Conventions, pitfalls and open questions](/concepts/conventions-pitfalls-open-questions.md)
