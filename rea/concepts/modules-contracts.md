---
type: Concept
title: Contracts and tool surface — canonical schemas, effects and catalog identity
description: How REA declares its public surface: one ToolContract per MCP tool with input/output Zod schemas, effect audit and examples; how contracts become advertised JSON Schema, compact or full, and how the catalog is digested and generated.
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
  - resource: https://github.com/yishao1224-design/rea/blob/8c4f96ef4759fd78da66c4b9726825ca71d7ee0c/src/contracts/toolContractTypes.ts
    id: contract-types
    title: src/contracts/toolContractTypes.ts
  - resource: https://github.com/yishao1224-design/rea/blob/8c4f96ef4759fd78da66c4b9726825ca71d7ee0c/src/contracts/toolContracts.ts
    id: contract-inventory
    title: src/contracts/toolContracts.ts
  - resource: https://github.com/yishao1224-design/rea/blob/8c4f96ef4759fd78da66c4b9726825ca71d7ee0c/src/contracts/toolEffects.ts
    id: effects
    title: src/contracts/toolEffects.ts
  - resource: https://github.com/yishao1224-design/rea/blob/8c4f96ef4759fd78da66c4b9726825ca71d7ee0c/src/contracts/errorSchemas.ts
    id: error-schemas
    title: src/contracts/errorSchemas.ts
  - resource: https://github.com/yishao1224-design/rea/blob/8c4f96ef4759fd78da66c4b9726825ca71d7ee0c/src/catalogIdentity.ts
    id: catalog
    title: src/catalogIdentity.ts
  - resource: https://github.com/yishao1224-design/rea/blob/8c4f96ef4759fd78da66c4b9726825ca71d7ee0c/scripts/generate-mcp-tool-catalog.mjs
    id: catalog-generator
    title: scripts/generate-mcp-tool-catalog.mjs
  - resource: https://github.com/yishao1224-design/rea/blob/8c4f96ef4759fd78da66c4b9726825ca71d7ee0c/docs/tool-design.md
    id: tool-design
    title: docs/tool-design.md
generated:
  by: dsh/agent
  at: 2026-11-20
---

# Contracts and tool surface

**Job.** Be the single source of truth for what REA advertises: names,
descriptions, input and output schemas, effects, annotations and examples —
shared by tool registration, CLI aliases, the generated catalog, the packaged
skill and the conformance tests.

The repository's rule: "Bind handlers to named contracts with exact types;
catalog ordering must never select an operation or schema" [tool-design].
`toolContract("name")` resolves a contract by literal name at compile time, so a
handler cannot drift to an unnamed schema [contract-inventory].

## Key files

| Path | Role |
| --- | --- |
| `src/contracts/toolContractTypes.ts` | `ToolContract`, `ToolExample`, `TOOL_KINDS` [contract-types] |
| `src/contracts/toolEffects.ts` | `ToolEffects`, `TOOL_EFFECTS` audit map, derived annotations [effects] |
| `src/contracts/toolContracts.ts` | ordered `TOOL_CONTRACTS` inventory assembled from ~40 modules; `toolContract(name)` [contract-inventory] |
| `src/contracts/<family>/*Contracts.ts` | per-family contracts (`native/`, `managed/`, `javascript/`, `android/`, `firmware/`, `wasm/`, `evm/`, …) |
| `src/contracts/officialToolContracts.ts`, `enhancedToolContracts.ts`, `sessionToolContracts.ts` | the core provider-shaped families |
| `src/contracts/promptContracts.ts` | six guided prompts |
| `src/contracts/toolSchemaMetadata.ts` | advertised input JSON Schema, output schema with metadata |
| `src/contracts/compactInputSchema.ts`, `inputSchemaPresentation.ts` | compact/full presentation and byte budgets |
| `src/contracts/jsonSchemaKeywords.ts`, `jsonSchemaReferences.ts` | dialect and `$ref` handling |
| `src/contracts/errorSchemas.ts` | shared error envelope; generates `docs/error-contract.schema.json` [error-schemas] |
| `src/contracts/toolOutputSchemaGroups.ts`, `toolOutputSchemaPrimitives.ts` | reusable output schema parts |
| `src/contracts/providerSelection.ts` | `REA_ANALYSIS_PROVIDER` selector schema |
| `src/catalogIdentity.ts` | counts + SHA-256 digests over the catalog [catalog] |

## Contract shape

```ts
interface ToolContract<Name, InputSchema, OutputSchema> {
  name: Name;
  title: string;
  description: string;             // the agent-facing instruction
  kind: ToolKind;                  // family, e.g. "native-provider", "session"
  inputSchema: InputSchema;        // Zod
  outputSchema: OutputSchema;      // Zod, projected to JSON Schema
  effects: ToolEffects;
  annotations: ToolAnnotations;    // readOnly/destructive/idempotent/openWorld hints
  returnsTargetContent: boolean;   // results may carry target-derived text
  examples: readonly ToolExample[];
}
```

`ToolEffects` covers `mutatesTarget`, `mutatesSession`, `writesFilesystem`,
`launchesProcess`, `accessesNetwork`, `changesUiState`, `mayDiscardData`,
`executesForeignCode`, `mutatesDevice` and `idempotent` [effects]. Two rules are
encoded in the comments rather than in types:

- `executesForeignCode` means code REA does not own — the target, a
  caller-selected executable, or an app loaded into one — but *not* REA's own
  pinned helpers or an operator-configured analysis engine.
- `mutatesSession` includes additive Evidence recording and unknown updates,
  because those change what a later call observes.

`TOOL_EFFECTS` is an explicit per-tool audit map; contract tests compare it
against the contract inventory rather than assuming a default.

## From contract to wire

1. **Registration.** Each `register*Tools` module binds a contract with
   `toolContract("<name>")` and passes the parsed input to one application
   workflow.
2. **Advertisement.** `advertisedInputJsonSchema(contract)` and
   `toolOutputSchemaWithMetadata(contract)` project Zod to JSON Schema
   (draft-2020-12). The compact profile then rewrites the advertised input
   schema within `COMPACT_INPUT_SCHEMA_BUDGET_BYTES` without changing server
   validation.
3. **Result encoding.** `returnsTargetContent` drives the
   `_meta.provenance.untrusted_content` marker at delivery time, and the
   instructions string warns the agent about prompt injection.
4. **Identity.** `createCatalogIdentity()` digests the sorted catalog
   (including effects and `returns_target_content`) plus prompts and CLI
   command names, and reports counts [catalog]. `binary_session` exposes these
   to the client.

## Generated artifacts and their guards

| Output | Produced by | Checked by |
| --- | --- | --- |
| `dist/mcp-catalog.json`, `dist/mcp-catalog-full.json` | `scripts/record-mcp-catalog.mjs` during `compile` | `npm run mcp-catalog:check` |
| recorded digest profiles (`dist/recordedCatalogDigests.js`) | `scripts/record-catalog-digests.mjs` | same compile step |
| `docs/public/product-catalog.json` | `scripts/generate-product-catalog.mjs` (ignored output) | `docs:check` |
| `docs/error-contract.schema.json` | `scripts/generate-error-schema.mjs` (tracked) | `docs:check`, `metadata:check` |
| packaged skill | `scripts/generate-skill-metadata.mjs` (ignored output) | `docs:check` |
| test catalog `.cache/mcp-tool-catalog.json` | `scripts/generate-mcp-tool-catalog.mjs` | `test:prepare` |

The tracked/ignored split matters: `src/generatedPackageMetadata.ts` and
`docs/error-contract.schema.json` are reviewed source that must be regenerated;
`docs/public/product-catalog.json`, `docs/verification/managed-conformance-*.json`
and `skills/` are build outputs that must not be committed.

## Conventions enforced by tests

Contract and boundary tests (`src/contracts/*.test.ts`,
`tests/boundary/mcp/*`) check, among others:

- every advertised schema is valid under its declared dialect
  (`toolSchemaValidity`),
- input references resolve (`toolInputReferences`),
- output references resolve and output schemas carry semantic constraints,
- compact and full profiles agree on structure (`compactInputSchema`),
- tuple and fixed-shape schemas stay in parity (`tupleSchemaParity`),
- every tool in `TOOL_EFFECTS` has a contract and vice versa (`toolEffects`),
- every documented example validates against its schema
  (`advertisedInputExamples`, `toolContractExamples`).

## Pitfalls

- Adding a tool without an effect entry, example and output schema breaks the
  catalog/effect audits.
- Changing a description or schema changes the catalog digest, which invalidates
  the recorded identity and can make a stale server report misaligned.
- Compact presentation is a *view*: never validate caller input against the
  compacted schema.
- `returnsTargetContent` is a security-relevant declaration, not a hint; an
  under-declared tool can pass target text to an agent without the untrusted
  marker.

## Related pages

- [MCP server](/concepts/modules-mcp-server.md)
- [CLI module](/concepts/modules-cli.md)
- [Conventions, pitfalls and open questions](/concepts/conventions-pitfalls-open-questions.md)
- [Build, run, test and deploy](/concepts/build-run-test-deploy.md)
