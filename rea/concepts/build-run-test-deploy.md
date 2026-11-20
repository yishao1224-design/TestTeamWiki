---
type: Concept
title: Build, run, test and deploy — exact commands
description: The pinned toolchain, install/build/run commands, the test projects and verification lanes, the repository gates, and how releases and documentation are published.
status: draft
lifecycle: draft
confidence: high
created: 2026-11-20
updated: 2026-11-20
tags: [unverified]
sources:
  - resource: /sources/2026-11-20-package-manifest.md
    id: manifest
    title: package.json manifest and script catalog (2026-11-20)
  - resource: /sources/2026-11-20-build-and-test-config.md
    id: build-config
    title: Build, test and CI configuration (2026-11-20)
  - resource: /sources/2026-11-20-contribution-and-docs-rules.md
    id: rules
    title: Contribution and documentation rules (2026-11-20)
  - resource: https://github.com/yishao1224-design/rea/blob/8c4f96ef4759fd78da66c4b9726825ca71d7ee0c/package.json
    id: package-json
    title: package.json
  - resource: https://github.com/yishao1224-design/rea/blob/8c4f96ef4759fd78da66c4b9726825ca71d7ee0c/turbo.json
    id: turbo
    title: turbo.json
  - resource: https://github.com/yishao1224-design/rea/blob/8c4f96ef4759fd78da66c4b9726825ca71d7ee0c/vitest.config.ts
    id: vitest
    title: vitest.config.ts
  - resource: https://github.com/yishao1224-design/rea/blob/8c4f96ef4759fd78da66c4b9726825ca71d7ee0c/.github/workflows/ci.yml
    id: ci
    title: .github/workflows/ci.yml
  - resource: https://github.com/yishao1224-design/rea/blob/8c4f96ef4759fd78da66c4b9726825ca71d7ee0c/docs/releasing.md
    id: releasing
    title: docs/releasing.md
  - resource: https://github.com/yishao1224-design/rea/blob/8c4f96ef4759fd78da66c4b9726825ca71d7ee0c/docs/testing.md
    id: testing-doc
    title: docs/testing.md
generated:
  by: dsh/agent
  at: 2026-11-20
---

# Build, run, test and deploy

All commands below are quoted from `package.json` scripts and the repository's
own guides [manifest]; the test project table is from
`scripts/lib/test-projects.mjs` [build-config].

## Toolchain

- Node from `.nvmrc`: **24.22.1**; `package.json#packageManager`:
  **npm@11.16.0**; `.npmrc` sets `engine-strict=true`.
- Supported runtime engines for the published package:
  `^22.23.4 || ^24.22.1 || >=26.12.1`.
- TypeScript is ESM (`"type": "module"`, `module: NodeNext`) and strict
  (`exactOptionalPropertyTypes`, `noUncheckedIndexedAccess`,
  `verbatimModuleSyntax`).

## Install and build

```bash
npm ci                      # locked deps + Husky hooks; does not build
npm run build:cached        # compile the runtime and bundle the packaged skill
npm run build               # same tasks, forced to rerun
npm run compile             # raw pipeline: clean → tsc -p tsconfig.build.json
                            #   → record catalog digests → record MCP catalog
                            #   → bundle runtime (outputs dist/)
```

Notes from the repository:

- `compile` is what writes `dist/mcp/main.js`, `dist/cli-runtime/*` and
  `dist/mcp-catalog*.json`; `scripts/rea.mjs` refuses to run without them.
- After changing `package.json`, the lockfile or the managed skill version, run
  `npm run metadata:generate` before building.
- Turbo caches deterministic builds and static checks; `cacheMaxAge` is 7 days
  and `cacheMaxSize` 500 MB [turbo].
- Build/generation commands hold a SQLite transaction around the Turbo graph;
  `.cache/rea-command-locks/` must not be removed while commands run.

## Run

```bash
node scripts/rea.mjs mcp        # npm start — MCP over stdio
node scripts/rea.mjs --help     # CLI help
npm run config:print            # print an MCP registration snippet
```

Installed forms:

```bash
npm install --global rea-agents   # provides `rea` and `rea-agents`
rea --help
npx -y rea-agents@latest analyze-javascript-application /absolute/path/to/app --json
npx rea-agents@latest setup       # register with an agent
rea update                        # verified update of the owning installation
```

`rea mcp --http` serves streamable HTTP (host/port from flags or `MCP_HOST` /
`MCP_PORT`); `rea mcp doctor` runs the in-process MCP readiness probe.

## Test

### Projects

| Project | Command selector | Files |
| --- | --- | --- |
| `domain` | `--project domain` | `src/{contracts,domain}/**/*.test.ts` |
| `services` | `--project services` | `src/application/**/*.test.ts` |
| `adapters` | `--project adapters` | `src/*.test.ts` + provider/adapter dirs |
| `composition` | `--project composition` | `tests/composition/**` |
| `conformance` | `--project conformance` | `tests/conformance/**` |
| `evaluation` | `--project evaluation` | `tests/evaluation/**` |
| `boundary` | `--project boundary` | `tests/boundary/**` except cli/mcp/process |
| `cli-boundary` | `--project cli-boundary` | `tests/boundary/cli/**` |
| `process-boundary` | `--project process-boundary` | `tests/boundary/process/**` |
| `mcp-boundary` | `--project mcp-boundary` | `tests/boundary/mcp/**` |
| `acceptance` | `--project acceptance` | `tests/acceptance/**` |
| `process-global` | `--project process-global` | `tests/process-global/**` |

Source projects need no build; built projects do.

### Commands

```bash
npm run test:prepare                       # generate catalogs/evidence the suite needs
npm run test:local -- PATH...              # dirty source tests, no build
npm run test:focused -- PATH...            # exact test files (builds compiled lanes)
npm run test:changed                       # source tests since merge-base with origin/main
npm run test:fast                          # all source projects
npm run test:boundary                      # boundary + cli + process + mcp + process-global
npm run test:mcp                           # MCP boundary only
npm run test:acceptance                    # compiled CLI/MCP acceptance workflows
npm run test:integration                   # all built projects
npm run test:python                        # LLDB/mitmproxy producers + analysis-efficiency checks
npm run test:watch                         # watch source projects
npm run test:coverage                      # v8 coverage, thresholds enforced
npm run test:ci                            # coverage + JUnit output
npm run test:ci:shard:run                  # shard for CI (zero thresholds) → test:ci:merge
npm run test:mutation:reverse-tools        # Stryker mutation runs (scheduled/manual)
```

Coverage thresholds when not sharded: 65 statements / 60 branches / 60
functions / 68 lines overall; `src/domain/**` 80/75/75/80;
`src/contracts/**` 85/80/80/85 [vitest].

Test process policy: at most 2 workers, `retry: 0`, 30 s timeout (60 s under
coverage), `process-boundary` serial, `cli-boundary` capped at 384 MiB heap
(512 MiB under coverage), and a custom `tests/processOwnershipRunner.ts`.

Temporary files must use `createTestTempDirectory`; verify hygiene with

```bash
npm run verify:test-temp-hygiene
```

### Checks and gates

```bash
npm run typecheck                  # tsc --noEmit via scripts/run-typescript.mjs
npm run lint                       # oxlint report + module-boundary verification
npm run lint:details               # oxlint + verify:module-boundaries, full output
npm run format:check               # oxfmt --check .
npm run knip                       # unused files/exports
npm run jscpd                      # duplication report
npm run scan:todos                 # list TODO/FIXME/HACK markers in src, tests, scripts, bridge
npm run docs:check                 # generated docs + README consistency
npm run verify:architecture-guards
npm run verify:test-discovery
npm run verify:module-boundaries

npm run check:fast                 # cached typecheck + lint (pre-push)
npm run check                      # typecheck lint:built format:check knip metadata:check + guards
npm run check:changed              # check:fast + test:changed
npm run check:pr                   # optional full local gate
npm run check:ci                   # check:ci:fast + check:ci:deep
```

`check:pr` is the complete local deterministic gate: typecheck, lint, format,
knip, metadata, `test:run`, `test:python`, `docs:check`, `jscpd`, `scan:todos`.

### Real-provider lanes

Engine behaviour is not proven by simulated providers. Each engine and format
has a `verify:*` script (see [the manifest capture](/sources/2026-11-20-package-manifest.md)),
usually `npm run build:cached && node scripts/verify-<target>.mjs`, e.g.
`npm run verify:ghidra`, `npm run verify:hopper`, `npm run verify:ida`,
`npm run verify:kuna`, `npm run verify:jeb`, `npm run verify:browser`,
`npm run verify:android`, `npm run verify:package`. Docs state package or
simulated-provider checks do not establish engine behaviour [testing-doc].

## CI

`.github/workflows/ci.yml` job ids: `changes`, `static`, `static-deep`,
`test-source`, `test-shard`, `reverse-tools-quality`, `build`, `sqlite-runtime`,
`test`, `conformance-fixtures`, `windows-native-build`, `package-e2e`,
`apple-artifacts`, `reconstruction-readiness`, `windows-curated`, `docs`,
`browser`, `binary_layout`, `evm`, `apk_signature`, `javascript_recovery`,
`recorded_crash`, `web_network`, `web_runtime`, `web_source_map`, `agentenv`,
`nativeaot`, `ghidra_bridge`, `website`, `required` [ci].

- Triggers: `push`, `pull_request`, `schedule`, `workflow_dispatch`,
  `workflow_call`.
- The `changes` job classifies the diff; the `ci:full` label or a scheduled /
  manual run switches the concurrency group from `feedback` to `qualification`.
- Static reports are uploaded as `static-check-reports` /
  `static-deep-check-reports`; built test runs are sharded and merged.
- Dedicated real-toolchain workflows (`real-*.yml`) cover Hopper (macOS and
  Linux), Ghidra (including Windows, MIPS, PPC, RISC-V, 6502, M68K, DOS, COM,
  raw, NativeAOT, Swift, switch tables), IDA, Kuna, JEB, Cutter, Frida, ADB,
  Apktool, Flutter, firmware, pcap/packets, signals, recorded crashes, EVM,
  APK signatures, Go binaries, binary layout, every browser lane,
  agent environments and Termux.

## Release and deploy

1. Pushes to `main` open/refresh a Release Please PR with the next version and
   changelog [releasing].
2. Merging that PR starts `.github/workflows/release.yml`. The workflow
   validates the merged release metadata and ancestry, then runs **full CI
   qualification at that exact merge SHA** before publishing.
3. Publication is npm (`rea-agents`) and the MCP Registry (`server.json`,
   `io.github.morluto/rea`, npm package, stdio transport, argument `mcp`).
   A rehearsal installs the retained tarball into an empty prefix and exercises
   CLI help plus a real stdio MCP catalog/session before the publish job runs.
4. A post-publication verifier installs from npm and repeats the checks,
   including an update from an older stable release.
5. `release-recovery.yml` (manual) republishes registry metadata for an already
   published npm release without new tags or bytes.

Documentation and website:

```bash
npm run docs:generate   # generated reference documents (ignored outputs)
npm run docs:dev        # VitePress dev server
npm run docs:build      # VitePress production build (checks links)
npm run docs:preview    # preview the built site at /rea/
```

- `.github/workflows/pages.yml` is a manual VitePress build; it does not publish.
- The public site <https://rea.tools/> serves `website/public/` through
  Cloudflare Workers (`website/wrangler.toml`).
- The separate GitHub Pages host uses only the manual
  `.github/workflows/website-pages.yml` on `main`.

## Host-specific prerequisites

- Linux: a procps-compatible `ps` is required for process-ownership checks by
  Ghidra, JADX, firmware tools and static JavaScript analysis.
- Android APK analysis: headless JADX plus a full JDK.
- Android devices: caller-supplied `adb`. Apktool: caller-supplied launcher.
- WASM: caller-supplied WABT 1.0.42. JS recovery: caller-supplied Wakaru on
  Linux x64. Firmware: Binwalk/Unblob on Linux. Packets: TShark 4.x + prlimit.
  Signals: sigrok-cli 0.7.2 + prlimit. Offline ELF/crash: pwntools Python.
- Browser: a caller-supplied Chrome-family executable;
  `REA_BROWSER_EXECUTABLE` (absolute) for native module resolution.

## Windows source checkouts

- `native/windows/build/` is git-ignored and **not** produced by `npm ci` or
  `npm run build:cached`; the Node-API addon requires Linux + MinGW-w64 + `tar`
  to build.
- Some source tests fail on Windows for POSIX-path / `#!/bin/sh` / rename
  reasons; compare against the base revision or run on Linux/WSL.
- `git submodule update --init --recursive` fails at the pinned Unblob
  submodule; a documented workaround exists in `docs/firmware-analysis.md`.

## Related pages

- [CLI module](/concepts/modules-cli.md)
- [Contracts and tool surface](/concepts/modules-contracts.md)
- [Conventions, pitfalls and open questions](/concepts/conventions-pitfalls-open-questions.md)
- [Overview](/concepts/overview.md)
