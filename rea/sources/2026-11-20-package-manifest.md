---
type: Source
title: package.json manifest and script catalog (capture, 2026-11-20)
description: Identity, engines, bin entries and the npm script catalog of the rea-agents package, with the exact command strings used to build, check, test, verify and document the project.
status: draft
lifecycle: draft
created: 2026-11-20
updated: 2026-11-20
source_url: https://github.com/yishao1224-design/rea/blob/8c4f96ef4759fd78da66c4b9726825ca71d7ee0c/package.json
retrieved: 2026-11-20
tags: [unverified]
generated:
  by: dsh/agent
  at: 2026-11-20
---

# package.json (capture)

Capture of `package.json` (26,248 bytes) at commit
`8c4f96ef4759fd78da66c4b9726825ca71d7ee0c`. The identity block is verbatim;
the script catalog is grouped for reading but every command string is
verbatim. Long verification lanes that only differ by target format are
listed once with their shared shape.

## Identity (verbatim)

```json
{
  "name": "rea-agents",
  "version": "6.7.0",
  "description": "Reverse engineer anything from your terminal or agent with one CLI and MCP server.",
  "license": "MIT",
  "type": "module",
  "packageManager": "npm@11.16.0",
  "bin": { "rea": "scripts/rea.mjs", "rea-agents": "scripts/rea.mjs" },
  "main": null,
  "types": null,
  "engines": { "node": "^22.23.4 || ^24.22.1 || >=26.12.1" }
}
```

Related repository files: `.nvmrc` = `24.22.1`, `.npmrc` = `engine-strict=true`,
`server.json` = MCP Registry entry `io.github.morluto/rea` version `6.7.0`
(npm package `rea-agents`, stdio transport, positional argument `mcp`).

## Install, build, run

| Script | Command |
| --- | --- |
| `prepare` | `node scripts/prepare.mjs` |
| `prepack` | `node scripts/prepack.mjs` |
| `prebuild` / `precheck` | `npm run deps:check` (`node scripts/check-dependency-install.mjs`) |
| `build` | `node scripts/run-exclusive.mjs artifacts node scripts/run-turbo.mjs run artifacts:skills --force` |
| `build:cached` | `node scripts/check-dependency-install.mjs && node scripts/run-exclusive.mjs artifacts node scripts/run-turbo.mjs run artifacts:skills` |
| `compile` | `node scripts/clean-build-output.mjs && node scripts/run-typescript.mjs -p tsconfig.build.json && node scripts/record-catalog-digests.mjs && node scripts/record-mcp-catalog.mjs && node scripts/bundle-runtime.mjs` |
| `start` | `node scripts/rea.mjs mcp` |
| `config:print` | `node scripts/print-mcp-config.mjs` |
| `build:windows-native` | `node scripts/build-windows-native.mjs` |
| `build:ghidra:nativeaot` | `node scripts/build-ghidra-nativeaot.mjs` |
| `build:termux` | `node scripts/run-exclusive.mjs artifacts npm run build:termux:run` |

## Static checks and repository gates

| Script | Command |
| --- | --- |
| `typecheck` | `node scripts/run-typescript.mjs --noEmit` |
| `lint` | `node scripts/ci/static-check-report.mjs lint && npm run verify:module-boundaries` |
| `lint:details` | `oxlint && npm run verify:module-boundaries` |
| `lint:fix` | `oxlint --fix && npm run verify:module-boundaries` |
| `lint:built` | `npm run lint` |
| `format` | `oxfmt .` |
| `format:check` | `oxfmt --check .` |
| `knip` | `knip --cache` |
| `jscpd` | `node scripts/ci/static-check-report.mjs duplicates` |
| `jscpd:details` | `jscpd src/ tests/ --no-tips` |
| `scan:todos` | `node scripts/scan-todos.mjs` |
| `check:fast` | `node scripts/run-turbo.mjs run typecheck lint --output-logs=errors-only` |
| `check` | `node scripts/run-exclusive.mjs artifacts node scripts/run-turbo.mjs run typecheck lint:built format:check knip metadata:check && npm run verify:architecture-guards` |
| `check:changed` | `npm run check:fast && npm run test:changed` |
| `check:pr` | `node scripts/run-exclusive.mjs artifacts node scripts/run-turbo.mjs run typecheck lint:built format:check knip metadata:check test:run && npm run test:python && npm run docs:check && npm run jscpd && npm run scan:todos` |
| `check:ci:fast` | `node scripts/run-exclusive.mjs artifacts npm run check:ci:fast:tasks` |
| `check:ci:deep` | `node scripts/run-exclusive.mjs artifacts npm run check:ci:deep:tasks && npm run verify:architecture-guards && npm run verify:test-discovery && npm run test:python` |
| `verify:architecture-guards` | `node scripts/verify/architecture-guards.mjs` |
| `verify:module-boundaries` | `node scripts/verify-module-boundaries.mjs` |
| `verify:test-discovery` | `node scripts/verify-test-discovery.mjs` |

## Tests

| Script | Command |
| --- | --- |
| `test` | `npm run test:with-build` |
| `test:prepare` | `node scripts/check-dependency-install.mjs && node scripts/run-exclusive.mjs artifacts node scripts/run-turbo.mjs run artifacts:mcp-catalog artifacts:product-catalog artifacts:managed-evidence` |
| `test:run` | `vitest run` |
| `test:local` | `node scripts/run-development-tests.mjs local` |
| `test:focused` | `node scripts/run-development-tests.mjs focused` |
| `test:changed` | `node scripts/run-development-tests.mjs changed` |
| `test:fast` | `node scripts/ci/test-group.mjs source` |
| `test:built` | `node scripts/ci/test-group.mjs built` |
| `test:boundary` | `vitest run --project boundary --project cli-boundary --project process-boundary --project mcp-boundary --project process-global` |
| `test:mcp` | `vitest run --project mcp-boundary` |
| `test:acceptance` | `vitest run --project acceptance` |
| `test:integration` | `vitest run --project boundary --project cli-boundary --project process-boundary --project mcp-boundary --project process-global --project acceptance` |
| `test:watch` | `vitest --project domain --project services --project adapters --project composition --project conformance --project evaluation --changed` |
| `test:watch:all` | `vitest --changed` |
| `test:python` | `python3 -m unittest bridge/native/test_rea_lldb_tracer.py bridge/pwndbg/test_core_context.py bridge/mitmproxy/test_capture.py -v && python3 -m unittest discover -s experiments/analysis-efficiency -p 'test_*.py' -v` |
| `test:coverage` | `vitest run --coverage.enabled --coverage.provider=v8` |
| `test:ci` | `npm run test:coverage -- --reporter=default --reporter=junit --outputFile.junit=test-results.xml` |
| `test:ci:shard:run` | `vitest run --coverage.enabled --coverage.provider=v8 --coverage.thresholds.lines=0 ... --reporter=default --reporter=blob` |
| `test:ci:merge` | `vitest --merge-reports --coverage.enabled --coverage.provider=v8` |
| `test:mutation:reverse-tools` | `node scripts/prepare-reverse-tools-mutation.mjs && stryker run stryker.config.mjs && stryker run stryker.semantics.config.mjs` |
| `verify:test-temp-hygiene` | `npm run test:prepare && node scripts/verify-test-temp-hygiene.mjs` |

## Artifacts and generated documentation

| Script | Command |
| --- | --- |
| `metadata:generate` | `node scripts/run-exclusive.mjs artifacts node scripts/generate-package-metadata.mjs` |
| `metadata:check` | `node scripts/generate-package-metadata.mjs --check` |
| `mcp-catalog:generate` | `node scripts/check-dependency-install.mjs && node scripts/run-exclusive.mjs artifacts node scripts/run-turbo.mjs run artifacts:mcp-catalog` |
| `artifacts:mcp-catalog` | `node scripts/generate-mcp-tool-catalog.mjs` |
| `artifacts:skills` | `node scripts/generate-skill-metadata.mjs` |
| `artifacts:product-catalog` | `node scripts/generate-product-catalog.mjs` |
| `artifacts:managed-evidence` | `node scripts/generate-completion-ledger.mjs` |
| `artifacts:error-schema` | `node scripts/generate-error-schema.mjs` |
| `docs:generate` | `node scripts/check-dependency-install.mjs && node scripts/run-exclusive.mjs artifacts npm run docs:prepare` |
| `docs:check` | `node scripts/check-dependency-install.mjs && node scripts/run-exclusive.mjs artifacts node scripts/run-turbo.mjs run docs:check:built && npm run test:local -- tests/conformance/readmeConsistency.test.ts` |
| `docs:dev` | `vitepress dev docs` |
| `docs:build` | `vitepress build docs` |
| `licenses:generate` | `node scripts/generate-third-party-inventory.mjs` |
| `licenses:check` | `node scripts/generate-third-party-inventory.mjs --check` |

## Real-provider verification lanes (names only; each runs a built runtime)

`verify:hopper`, `verify:hopper:linux`, `verify:hopper:deadlines`, `verify:hopper:fat`,
`verify:ghidra`, `verify:ghidra:seeds`, `verify:ghidra:mips`, `verify:ghidra:ppc`,
`verify:ghidra:riscv`, `verify:ghidra:6502`, `verify:ghidra:m68k`,
`verify:ghidra:mips-shared`, `verify:ghidra:mips-relocatable`, `verify:ghidra:swift`,
`verify:ghidra:switch`, `verify:ghidra:dos`, `verify:ghidra:com`, `verify:ghidra:raw`,
`verify:ghidra:address-context`, `verify:ghidra:cross-format`, `verify:ghidra:windows`,
`verify:ghidra:nativeaot`, `verify:ida`, `verify:kuna`, `verify:jeb`, `verify:cutter`,
`verify:frida`, `verify:frida:bindings`, `verify:android`, `verify:apktool`,
`verify:adb`, `verify:apk-signature`, `verify:pcap`, `verify:flutter`,
`verify:firmware`, `verify:managed`, `verify:agentenv:environment`,
`verify:browser`, `verify:browser:dom`, `verify:browser:network`,
`verify:browser:scripts`, `verify:browser:modules`, `verify:browser:runtime`,
`verify:browser:source-maps`, `verify:electron`, `verify:inspector`,
`verify:javascript-runtime`, `verify:javascript:digests`,
`verify:javascript:output`, `verify:javascript:module-references`,
`verify:javascript:native-handoffs`, `verify:javascript:retained-facts`,
`verify:javascript:recovery`, `verify:application-workflows`,
`verify:web:network-captures`, `verify:web:http3`, `verify:web:qlog`,
`verify:binary:layout`, `verify:go:binary`, `verify:il2cpp:metadata`,
`verify:wasm:artifact`, `verify:wasm:package`, `verify:wasm-interface-comparison`,
`verify:evm:interface`, `verify:solidity:metadata`, `verify:recorded:crash`,
`verify:signal-recording`, `verify:packet-capture`, `verify:apple-dispatch`,
`verify:keyed-archive`, `verify:macos-bundle`, `verify:interface-builder`,
`verify:asset-catalog`, `verify:native-ui`, `verify:native-calls`,
`verify:readiness`, `verify:fixtures`, `verify:package`,
`verify:package:windows`, `verify:published`, `verify:release-artifact`,
`verify:setup:linux`, `verify:windows-native`, `verify:windows-native:artifact`,
`verify:packed-bridge`, `verify:qwen-client`, `verify:gemini-client`,
`verify:opencode-client`, `verify:grok-client`, `verify:commandcode-client`,
`verify:omp-client`, `verify:copilot-client`, `verify:claude-client`,
`verify:pi-client`, `verify:hermes-client`, `verify:deepseek-client`,
`verify:agent`, `probe:windows-capabilities`.

Most start with `npm run build:cached && node scripts/<verifier>.mjs`.
