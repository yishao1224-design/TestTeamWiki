---
type: Source
title: Build, test and CI configuration (capture, 2026-11-20)
description: Toolchain pins, Turbo task graph, Vitest project ownership, coverage thresholds and the CI job inventory that define how REA is built and verified.
status: draft
lifecycle: draft
created: 2026-11-20
updated: 2026-11-20
source_url: https://github.com/yishao1224-design/rea/blob/8c4f96ef4759fd78da66c4b9726825ca71d7ee0c/vitest.config.ts
retrieved: 2026-11-20
tags: [unverified]
generated:
  by: dsh/agent
  at: 2026-11-20
---

# Build and verification configuration (capture)

Excerpts of `.nvmrc`, `package.json` (`engines`, `packageManager`),
`turbo.json`, `vitest.config.ts`, `scripts/lib/test-projects.mjs`,
`tsconfig.json`/`tsconfig.build.json` and `.github/workflows/ci.yml`.

## Toolchain pins

- `.nvmrc`: `24.22.1`
- `package.json#packageManager`: `npm@11.16.0`
- `package.json#engines.node`: `^22.23.4 || ^24.22.1 || >=26.12.1`
- `.npmrc`: `engine-strict=true`

## TypeScript

`tsconfig.json` (verbatim options, abridged include list):

```json
{
  "compilerOptions": {
    "allowUnreachableCode": false, "allowUnusedLabels": false,
    "exactOptionalPropertyTypes": true, "forceConsistentCasingInFileNames": true,
    "lib": ["ES2023"], "module": "NodeNext", "moduleResolution": "NodeNext",
    "noFallthroughCasesInSwitch": true, "noImplicitOverride": true,
    "noImplicitReturns": true, "noUncheckedIndexedAccess": true,
    "outDir": "dist", "rootDir": ".", "skipLibCheck": true, "strict": true,
    "target": "ES2023", "types": ["node"], "verbatimModuleSyntax": true
  },
  "include": ["src/**/*.ts", "tests/**/*.ts", "vitest.config.ts", "docs/.vitepress/config.ts"]
}
```

`tsconfig.build.json` extends it with `rootDir: "src"`, includes
`src/**/*.ts` and excludes `src/**/*.fixture.ts`, `src/**/*.test.ts` and
`tests`.

## Turbo tasks (`turbo.json`)

- `cacheMaxAge: "7d"`, `cacheMaxSize: "500MB"`,
  `globalDependencies: [".nvmrc", ".npmrc"]`,
  `globalPassThroughEnv: ["GOMAXPROCS", "GOMEMLIMIT", "UV_THREADPOOL_SIZE", "REA_TYPESCRIPT_COMPILER"]`.
- `compile`: depends on `metadata:check`; inputs are `package.json`,
  `package-lock.json`, selected `scripts/*.mjs`, `src/**/*.{cts,mts,ts,tsx}`
  (excluding tests and fixtures), `tsconfig.json`, `tsconfig.build.json`;
  outputs `dist/**`.
- `typecheck`, `lint`, `lint:built` (depends on `compile`), `format:check`,
  `knip` (depends on `compile`), `metadata:check`, `jscpd`, `scan:todos`,
  `test:run`, `docs:check:built`, `artifacts:*`, `evidence:check:built`,
  `mcp-catalog:check:built` each declare their own file inputs.

## Vitest projects (`scripts/lib/test-projects.mjs`)

Source projects (no build prerequisite):

| Project | Include |
| --- | --- |
| `domain` | `src/{contracts,domain}/**/*.test.ts` |
| `services` | `src/application/**/*.test.ts` |
| `adapters` | `src/*.test.ts` and tests under `src/{android,apkSignature,apktool,jeb,adb,artifacts,browser,config,cutter,dotnet,electron,erlang,evm,firmware,flutter,pcap,ghidra,go,gpu,hopper,ida,inspector,javascript,main,native,process,python,reference,rust,server,sqlite,wasm,windows}/**` |
| `composition` | `tests/composition/**/*.test.ts` |
| `conformance` | `tests/conformance/**/*.test.ts` (excludes `realHopperSearch.test.ts`) |
| `evaluation` | `tests/evaluation/**/*.test.ts` |

Built projects (require `npm run build:cached` / `test:prepare`):

| Project | Include |
| --- | --- |
| `boundary` | `tests/boundary/**/*.test.ts` plus `realHopperSearch.test.ts`; excludes `cli/`, `mcp/`, `process/` |
| `cli-boundary` | `tests/boundary/cli/**/*.test.ts` |
| `process-boundary` | `tests/boundary/process/**/*.test.ts` |
| `mcp-boundary` | `tests/boundary/mcp/**/*.test.ts` |
| `acceptance` | `tests/acceptance/**/*.test.ts` |
| `process-global` | `tests/process-global/**/*.test.ts` |

`projectForTest(path)` throws unless exactly one project owns a path.

## Runner policy (`vitest.config.ts`)

- `MAX_TEST_WORKERS = Math.min(2, availableParallelism())`; `retry: 0`;
  `testTimeout: 30_000` (`60_000` under coverage).
- Pool per project: `domain`/`services`/`composition`/`mcp-boundary`/
  `conformance`/`evaluation` use `threads` with `isolate: false`;
  `adapters`/`boundary`/`acceptance`/`process-global` use `forks`;
  `process-boundary` sets `fileParallelism: false`;
  `cli-boundary` sets `execArgv: ["--max-old-space-size=384"]`
  (`512` under coverage).
- `runner: "./tests/processOwnershipRunner.ts"`.
- Coverage: v8 provider, `include: ["src/**"]`; thresholds when not sharded:
  statements 65, branches 60, functions 60, lines 68; `src/domain/**` 80/75/75/80;
  `src/contracts/**` 85/80/80/85. Sharded runs use zero thresholds.
- Optional `REA_CI_TEST_HISTORY` switches in `tests/ciTestSequencer.ts`;
  `TMPDIR` is set to the realpath of `os.tmpdir()`.

## CI jobs (`.github/workflows/ci.yml`, job ids)

`changes`, `static`, `static-deep`, `test-source`, `test-shard`,
`reverse-tools-quality`, `build`, `sqlite-runtime`, `test`,
`conformance-fixtures`, `windows-native-build`, `package-e2e`,
`apple-artifacts`, `reconstruction-readiness`, `windows-curated`, `docs`,
`browser`, `binary_layout`, `evm`, `apk_signature`, `javascript_recovery`,
`recorded_crash`, `web_network`, `web_runtime`, `web_source_map`, `agentenv`,
`nativeaot`, `ghidra_bridge`, `website`, `required`.

Workflow triggers: `push`, `pull_request`, `schedule`, `workflow_dispatch`,
`workflow_call`; concurrency group switches between `feedback` and
`qualification` (the latter for scheduled/manual runs, `source_sha` input, or
the `ci:full` label).

Real-provider workflows in `.github/workflows/`:
`real-hopper.yml`, `real-hopper-linux.yml`, `real-ghidra-windows.yml`,
`real-ida` (script-driven), `real-browser.yml`, `real-cutter.yml`,
`real-evm-interface.yml`, `real-go-binary.yml`, `real-apk-signature.yml`,
`real-binary-layout.yml`, `real-javascript-recovery.yml`,
`real-packet-capture.yml`, `real-recorded-crash.yml`,
`real-signal-recording.yml`, `real-termux.yml`, `real-web-network-captures.yml`,
`real-web-runtime.yml`, `real-web-source-map.yml`, `real-agentenv.yml`,
`mutation.yml`, `nativeaot-fixtures.yml`, `ghidra-bridge-sources.yml`,
`sqlite-runtime.yml`, `ci-full-pr.yml`, `ci-timings.yml`, `pr-readiness.yml`,
`pr-title.yml`, `release.yml`, `release-recovery.yml`, `pages.yml`,
`website-pages.yml`, `website-check.yml`, `cache-retention.yml`.

## Release pipeline (`.github/workflows/release.yml`, head)

Jobs are keyed on `release-proposal` (push to `main`, Release Please), plus
prepare/publish phases for a frozen `release/VERSION` branch. The workflow
declares `HUSKY: "0"` and the same Turbo cache variables as CI, with remote
writes restricted to the default branch.
