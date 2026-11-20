---
type: Concept
title: Conventions, pitfalls and open questions
description: The rules a contributor must follow, the traps the code and docs warn about, and the points this survey could not settle.
status: draft
lifecycle: draft
confidence: medium
created: 2026-11-20
updated: 2026-11-20
tags: [unverified]
sources:
  - resource: /sources/2026-11-20-contribution-and-docs-rules.md
    id: rules-capture
    title: Contribution and documentation rules (2026-11-20)
  - resource: /sources/2026-11-20-build-and-test-config.md
    id: build-capture
    title: Build, test and CI configuration (2026-11-20)
  - resource: https://github.com/yishao1224-design/rea/blob/8c4f96ef4759fd78da66c4b9726825ca71d7ee0c/AGENTS.md
    id: agents
    title: AGENTS.md
  - resource: https://github.com/yishao1224-design/rea/blob/8c4f96ef4759fd78da66c4b9726825ca71d7ee0c/CONTRIBUTING.md
    id: contributing
    title: CONTRIBUTING.md
  - resource: https://github.com/yishao1224-design/rea/blob/8c4f96ef4759fd78da66c4b9726825ca71d7ee0c/docs/tool-design.md
    id: tool-design
    title: docs/tool-design.md
  - resource: https://github.com/yishao1224-design/rea/blob/8c4f96ef4759fd78da66c4b9726825ca71d7ee0c/docs/testing.md
    id: testing
    title: docs/testing.md
  - resource: https://github.com/yishao1224-design/rea/blob/8c4f96ef4759fd78da66c4b9726825ca71d7ee0c/docs/design/limits-and-recovery.md
    id: limits
    title: docs/design/limits-and-recovery.md
  - resource: https://github.com/yishao1224-design/rea/blob/8c4f96ef4759fd78da66c4b9726825ca71d7ee0c/server.json
    id: server-json
    title: server.json
generated:
  by: dsh/agent
  at: 2026-11-20
---

# Conventions, pitfalls and open questions

## Conventions a contributor must follow

### Layering and ownership

- Dependencies flow inward: domain/contracts → provider adapters → application
  workflows → CLI/MCP adapters. Engine protocols stay out of domain and
  application code [agents].
- Each interpretation has one owner; consumers use that owner's parsed facts
  instead of re-reading raw source.
- Source-format interpretation is normalised once at the owning boundary with
  format-aware parsers.
- Ownership, validity, editability and readiness are separate proofs; unresolved
  identity stays unknown.
- Reuse `src/process/` supervision rather than hand-rolling spawn/kill logic.

### Tool design

- Start from the analyst question; extend an existing contract when intent and
  result match.
- Declare effects truthfully (`ToolEffects`) and keep observed/derived/inferred
  distinct.
- Limits need a real format, protocol, authority or measured resource
  constraint — "an assumed agent budget is insufficient" [tool-design]. Name
  the ceiling and the recovery.
- Keep the catalog complete; report availability rather than truncating
  schemas.

### Tests

- Every test file must be owned by exactly one Vitest project;
  `npm run verify:test-discovery` checks it [build-capture].
- Prefer real public CLI/MCP workflows, then production-boundary integration,
  then producer goldens.
- Lifecycle changes need late handshakes, cancellation during final cleanup,
  mixed child outcomes and cleanup retry coverage.
- Temporary files go through `createTestTempDirectory`; `npm run
  verify:test-temp-hygiene` enforces cleanup.
- Report coverage honestly: simulated providers do not establish real-provider
  behaviour.

### Generated files

| Tracked (must regenerate) | Ignored / build-only (must not commit) |
| --- | --- |
| `src/generatedPackageMetadata.ts` | `docs/public/product-catalog.json` |
| `docs/error-contract.schema.json` | `docs/verification/managed-conformance-*.json` |
| | `skills/`, `.cache/mcp-tool-catalog.json` |
| | `dist/` including `dist/mcp-catalog*.json` and `dist/recordedCatalogDigests.js` |

`src/recordedCatalogDigests.ts` and `src/recordedMcpCatalog.ts` are tracked
source stubs that export `undefined`; the recorders rewrite the *compiled*
copies in `dist/`, so a source run computes digests lazily and takes the eager
startup path.

Authored skill instructions live in `.agents/skills/reverse-engineer-anything/`;
`skills/` is generated.

### Commits and PRs

- Conventional Commit subjects and PR titles; breaking changes marked with `!`
  or a `BREAKING CHANGE:` footer.
- A human must own the diff and fill the **Author readiness** declarations; an
  agent must not attest to human review.
- Scope boundaries are fail-closed: unresolved scope, authority or side-effect
  questions block integration (CONTRIBUTING, "Contribution scope" and
  "Fail-closed integration").

## Pitfalls

### Runtime and build

1. **Missing compiled runtime.** `scripts/rea.mjs` needs `dist/`. The error text
   tells you to run `npm ci && npm run build:cached`; running TypeScript sources
   directly is not supported.
2. **Stale recorded catalog.** Deferred stdio startup serves
   `dist/mcp-catalog*.json` only if its SHA-256 matches the compiled digest.
   Changing a contract without recompiling (or mismatching the recorded digest)
   makes startup fail closed with `SERVER_START_FAILED`.
3. **Deferred vs eager divergence.** Default stdio startup builds no analysis
   services until the first request; HTTP, an initial `HOPPER_TARGET_PATH`, or
   output-schema suppression take the eager path. A change validated only in one
   mode can break the other.
4. **stdout is protocol.** MCP messages and CLI results share stdout; any
   library that logs there corrupts them. Logs go to stderr.
5. **SIGPIPE is a status, not an error.** `rea … | head` exits 141 with no
   message.
6. **Lockfiles.** Build/generation commands hold a lock via
   `.cache/rea-command-locks/`; deleting these while commands run breaks the
   exclusion.

### Configuration

7. **Absolute-path settings.** `REA_IDA_MCP_CONFIG`, `GHIDRA_INSTALL_DIR`,
   `JAVA_HOME`, `REA_GHIDRA_SEED_FILE`, `REA_GHIDRA_NATIVEAOT_JAR`,
   `REA_ILSPY_CMD_PATH` must be absolute; a relative value is a configuration
   error naming the constraint (never the value).
8. **Coupled Ghidra settings.** `REA_GHIDRA_COMPILER_SPEC_ID` requires
   `REA_GHIDRA_LANGUAGE_ID`.
9. **Tool env vars only count when absolute.** A relative
   `REA_TSHARK_COMMAND`/`REA_WAKARU_COMMAND`/… leaves the capability
   unavailable at discovery time and may still be reported at execution.
10. **JEB timeout isolation.** An invalid `REA_JEB_MCP_TIMEOUT_MS` must not
    block unrelated startup; it is parsed at the optional-provider boundary.
11. **Log level.** An invalid `REA_LOG_LEVEL` stops every command except
    `doctor`.

### Host and platform

12. **Linux needs `ps`.** Process-ownership checks for Ghidra, JADX, firmware
    and static JavaScript require a procps-compatible `ps` on `PATH`.
13. **Windows source checkouts** do not have the native addon; build it on
    Linux with MinGW-w64 and `tar`. Some POSIX-fixture tests also fail on
    Windows for environmental reasons.
14. **Unblob submodule.** `git submodule update --init --recursive` fails at the
    pinned Unblob submodule on Windows; use the documented workaround.
15. **Hopper first run.** A macOS first-run demo/licence dialog must be completed
    in the same user session before unattended CLI/MCP use.
16. **Commercial engines.** IDA/Hex-Rays, Hopper and JEB must be licensed
    separately; REA grants no entitlement.

### Analysis semantics

17. **Target content is untrusted data.** Tool results carrying target strings,
    pseudocode or page text are marked `_meta.provenance.untrusted_content`;
    agents are instructed to treat embedded instructions as prompt injection.
18. **Absence ≠ none.** Missing provider coverage stays unknown; relationships
    and unknowns have explicit unknown states.
19. **Snapshot replay is narrower than it looks.** Only operations with a
    replayable cache policy replay; live state must be re-observed.
20. **Provider deadline ≠ client deadline.** A host "timed out awaiting
    tools/call" is not evidence that work or cleanup stopped.

## Open questions

These are unresolved by the code and documentation I read; they are recorded as
questions, not as facts.

1. **Repository identity.** The checkout's `origin` is
   `https://github.com/yishao1224-design/rea` at commit
   `8c4f96ef4759fd78da66c4b9726825ca71d7ee0c` (2026-10-23, "chore(deps): update
   Vite security patch and dependency inventory (#6416)"), while `README.md`,
   `server.json` and the website point at `morluto/rea` [server-json]. Whether
   the reviewed repository is a fork, a mirror, or the canonical upstream is not
   stated anywhere I read. Every page here cites the reviewed commit URL.
2. **Exact public surface size.** `CLI_COMMAND_NAMES` has 168 entries (plus one
   alias), but the MCP tool count and prompt count are only reported at runtime
   by `binary_session` and in generated artifacts. I did not count them, and
   deliberately did not restate a number in the pages.
3. **Version vs date.** `package.json`, `server.json` and
   `.release-please-manifest.json` agree on `6.7.0`; the checked-out commit is
   dated 2026-10-23, four weeks before this wiki's date. Whether an unreleased
   change series exists on `main` after `6.7.0` is unknown.
4. **Which documentation site publishes.** `docs/` is a VitePress site
   (`npm run docs:build`), `.github/workflows/pages.yml` builds it manually and
   does not publish, and `website/public/` is separately deployed to
   `rea.tools` via Cloudflare Workers. The relationship between the two
   documentation surfaces (which is authoritative for users) is not stated in
   one place.
5. **Deferred-startup limits.** Deferred startup is disabled by HTTP,
   `HOPPER_TARGET_PATH`, output-schema suppression, or injected
   providers/servers. What happens when a *source* run has no
   `dist/mcp-catalog.json` at all is only implied by the eager fallback; I did
   not execute either path.
6. **Toolchain variants.** The repo mentions `tsrs` compilers for Linux/macOS,
   the official TypeScript compiler for Windows, a `termux` build path, and a
   `REA_TYPESCRIPT_COMPILER` override. Which compiler is used on an arbitrary
   contributor host is decided at runtime by `scripts/run-typescript.mjs`; I did
   not verify it.
7. **Test/reality gap.** Many behaviours (Hopper/Ghidra/IDA/JEB/Cutter/browser
   lanes) are only proven by real-toolchain workflows on specific hosts. This
   wiki describes the code paths, not observed engine behaviour.

## Related pages

- [Build, run, test and deploy](/concepts/build-run-test-deploy.md)
- [Architecture](/concepts/architecture.md)
- [Data model](/concepts/data-model.md)
- [Contracts and tool surface](/concepts/modules-contracts.md)
