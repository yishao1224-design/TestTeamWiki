---
type: Concept
title: JavaScript and Electron analysis — static graph, source maps, recovery, runtime reconciliation
description: The JavaScript/Electron subsystem: artifact reconstruction from directories and ASARs, the worker-isolated parser and semantic analyser, source-map decoding, Wakaru-based source recovery, Electron boundary analysis, V8 runtime observation and reconciliation with static facts.
status: draft
lifecycle: draft
confidence: medium
created: 2026-11-20
updated: 2026-11-20
tags: [unverified]
sources:
  - resource: https://github.com/yishao1224-design/rea/blob/8c4f96ef4759fd78da66c4b9726825ca71d7ee0c/src/application/javascript/JavaScriptApplicationService.ts
    id: js-service
    title: src/application/javascript/JavaScriptApplicationService.ts
  - resource: https://github.com/yishao1224-design/rea/blob/8c4f96ef4759fd78da66c4b9726825ca71d7ee0c/src/application/javascript/JavaScriptArtifactReconstruction.ts
    id: js-reconstruction
    title: src/application/javascript/JavaScriptArtifactReconstruction.ts
  - resource: https://github.com/yishao1224-design/rea/blob/8c4f96ef4759fd78da66c4b9726825ca71d7ee0c/src/composition/javascriptAnalysisWorkerProcess.ts
    id: worker-process
    title: src/composition/javascriptAnalysisWorkerProcess.ts
  - resource: https://github.com/yishao1224-design/rea/blob/8c4f96ef4759fd78da66c4b9726825ca71d7ee0c/src/javascript/sourceMaps/SourceMapDecoder.ts
    id: source-map
    title: src/javascript/sourceMaps/SourceMapDecoder.ts
  - resource: https://github.com/yishao1224-design/rea/blob/8c4f96ef4759fd78da66c4b9726825ca71d7ee0c/src/javascript/recovery/WakaruCommand.ts
    id: wakaru
    title: src/javascript/recovery/WakaruCommand.ts
  - resource: https://github.com/yishao1224-design/rea/blob/8c4f96ef4759fd78da66c4b9726825ca71d7ee0c/src/inspector/V8InspectorProvider.ts
    id: inspector
    title: src/inspector/V8InspectorProvider.ts
  - resource: https://github.com/yishao1224-design/rea/blob/8c4f96ef4759fd78da66c4b9726825ca71d7ee0c/docs/javascript-application-workflows.md
    id: js-doc
    title: docs/javascript-application-workflows.md
  - resource: https://github.com/yishao1224-design/rea/blob/8c4f96ef4759fd78da66c4b9726825ca71d7ee0c/docs/javascript-artifact-reconstruction.md
    id: js-recon-doc
    title: docs/javascript-artifact-reconstruction.md
generated:
  by: dsh/agent
  at: 2026-11-20
---

# JavaScript and Electron analysis

**Job.** Reconstruct a JavaScript/Electron application from its shipped files
without running it, recover modules, imports, exports, routes, IPC boundaries
and native add-on relationships, and — when needed — reconcile those static
facts with what a running runtime actually did.

This is REA's most developed static surface: about 290 files under
`src/domain/javascript/` alone, plus `src/application/javascript/` (76 files),
`src/javascript/` (workers, source maps, recovery) and `src/inspector/`.

## Key files

| Path | Role |
| --- | --- |
| `src/application/javascript/JavaScriptApplicationService.ts` | entry workflow `analyze_javascript_application` → `reconstructJavaScriptArtifact` → cooperative Evidence [js-service] |
| `src/application/javascript/JavaScriptArtifactReconstruction.ts` | directory/ASAR inventory, decoding and admission |
| `src/application/javascript/JavaScriptArtifactGraph*.ts` | module graph construction (bundlers, documents, findings, evidence) |
| `src/application/javascript/JavaScriptSemanticGraph*.ts` | semantic projections and trace service |
| `src/application/javascript/JavaScriptApplicationWorkflowService.ts` | feature trace, version comparison, source-to-bundle |
| `src/domain/javascript/javascriptStaticAnalysis.ts` | static facts per parsed module (`analyzeParsedJavaScriptStaticSourceSteps`) |
| `src/domain/javascript/javascriptSemanticAnalysis.ts` | semantic IR (`analyzeParsedJavaScriptSemanticsSteps`) |
| `src/domain/javascript/electronStatic*.ts` | Electron boundary analysis (preload, IPC, native loading, app paths, fuses) |
| `src/composition/javascriptAnalysisWorkerProcess.ts` | the worker body: parse → static → semantic → projection [worker-process] |
| `src/javascript/analysis/*` | worker protocol, transfer budgets, worker server |
| `src/javascript/sourceMaps/*` | source-map codec in an owned heap-limited Node process [source-map] |
| `src/javascript/recovery/*` | Wakaru invocation, report parsing, publication [wakaru] |
| `src/inspector/*` | passive V8 Inspector discovery and capture [inspector] |
| `src/artifacts/Asar*`, `src/artifacts/javascript/*` | ASAR header/listing and artifact readers |

## Interfaces

### Analysis entry points

- MCP tool `analyze_javascript_application`; CLI
  `rea analyze-javascript-application PATH --json`; generic `rea analyze PATH`
  selects this workflow for directories and `.asar` files when neither
  `--provider` nor `--snapshot` is given [js-doc].
- Related MCP tools/commands: `trace_application_feature`,
  `compare_application_versions`, `compare_source_to_bundle`,
  `compare_javascript_export_shapes`, `trace_javascript_semantics`,
  `recover_javascript_sources`, `observe_javascript_runtime`,
  `reconcile_javascript_runtime`, `list_javascript_runtime_targets`,
  plus Electron tools (`list_electron_targets`, `inspect_electron_page`,
  `inspect_electron_fuses`, `capture_electron_scenario`).

### Isolation and budgets

The semantic analysis runs in an owned worker process, not in the MCP server:

- `JavaScriptAnalysisWorker` / `JavaScriptAnalysisWorkerServer` speak a framed
  protocol; the parent offers a result budget per module
  (`javaScriptTransferByteBudget`) and the child refuses to start semantic work
  when the parent reserved zero capacity [worker-process].
- Per-module failures degrade to static facts with an
  `AnalysisResourceConstraintError` naming `semantic_result_budget_bytes`, so a
  partial graph is still returned.
- `src/process/OwnedWorkerRuntime.ts` and
  `src/process/JavaScriptWorkerMemoryLimit.ts` own the worker lifetime and heap
  ceiling.
- Source-map decoding is a second owned process (`SourceMapCodecProcess`) with
  its own protocol, so a hostile map cannot grow the server heap.

### Recovered source

Recovered-source extraction (`recover-javascript-sources` /
`recover_javascript_sources`) is delegated to a caller-supplied Wakaru binary:
`REA_WAKARU_COMMAND` must be an absolute path to Wakaru ^1.13.0 on Linux x64
(verified with 1.14.0) [wakaru]. REA refuses rather than installing it. Recovery
publishes into an owned output directory with file ownership and cancellation
tracked (`WakaruPublication`, `RecoveryFiles`).

### Runtime reconciliation

`src/inspector/` discovers V8 Inspector endpoints and captures scripts and
target locations passively; `JavaScriptRuntimeObservationService` records
observations, and `JavaScriptRuntimeReconciliationService` matches static graph
nodes to runtime facts, keeping unmatched nodes explicitly unknown rather than
absent.

### Large results

A real application's Evidence can be hundreds of megabytes. The documented
pattern is to save it and project a view:

```bash
rea analyze-javascript-application /absolute/path/to/app --json > app-evidence.json
jq -c '{source: {kind: "inline", evidence: .}, view: {kind: "summary"}}' app-evidence.json > app-view.json
rea inspect-analysis-view app-view.json
```

A 334 MB Obsidian Evidence file yields roughly a 10 KB summary view; a page view
(`{"kind": "page", "collection": "modules", …}`) pages module rows [js-doc].

## Dependencies

- `src/artifacts` for ASAR/zip/directory reads and bounded hashing.
- `src/domain/javascript` for parsing and semantic IR (pure).
- `src/process` for owned worker and decoder processes.
- `src/browser`/`src/inspector` for runtime observation and CDP helpers.
- `REA_ASAR_MAX_HEADER_BYTES` (`src/config/asarHeaderAdmission.ts`) bounds ASAR
  header admission; an invalid value is refused before analysis.

## Pitfalls

- `moduleResolutionModeForJavaScriptReference` distinguishes static imports
  from dynamic/require references; conflating them changes the graph.
- Bundler registrations (`javascriptBundlerRegistrationIdentity`,
  `javascriptStaticAnalysisBundler`) exist because bundled code reuses one
  module registry; the graph must not treat registration order as ownership.
- Receiver/prototype reasoning is modelled explicitly (`javascriptSemantic*`
  files); a resolved call is not automatically a unique target — ambiguity must
  stay visible.
- Computed imports and bundler IDs remain unknown by contract.

## Related pages

- [Browser and web analysis](/concepts/modules-browser-and-web.md)
- [Static artifacts](/concepts/modules-static-artifacts.md)
- [Process supervision](/concepts/modules-process-supervision.md)
- [Data model](/concepts/data-model.md)
