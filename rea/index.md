---
okf_version: "0.2"
---

# Bundle Index

* [Bundle Schema & Conventions](SCHEMA.md) - Authoritative conventions for this OKF bundle — types, tags, frontmatter templates, lifecycle rules.

# Concepts

* [Architecture — components, communication and where data lives](concepts/architecture.md) - The layered architecture of REA: dispatcher and entry points, MCP and CLI adapters, shared application workflows, provider layers, the process foundation, contracts and domain, plus the runtime data locations.
* [Browser and web analysis — CDP capture, scenarios, historical captures, module tracing](concepts/modules-browser-and-web.md) - The web-facing subsystem: CDP observation of pages and scripts, Playwright-driven scenarios, HAR/SAZ/mitmproxy/qlog historical captures, bundle analysis, source-location tracing, network-capture projection and serving a recorded environment.
* [Build, run, test and deploy — exact commands](concepts/build-run-test-deploy.md) - The pinned toolchain, install/build/run commands, the test projects and verification lanes, the repository gates, and how releases and documentation are published.
* [CLI adapter — one-shot commands, output, exit behaviour](concepts/modules-cli.md) - The `rea` command-line adapter: entry point and dispatcher, command registration, shared analysis workflow binding, global output flags, logging and exit-status rules.
* [Contracts and tool surface — canonical schemas, effects and catalog identity](concepts/modules-contracts.md) - How REA declares its public surface: one ToolContract per MCP tool with input/output Zod schemas, effect audit and examples; how contracts become advertised JSON Schema, compact or full, and how the catalog is digested and generated.
* [Conventions, pitfalls and open questions](concepts/conventions-pitfalls-open-questions.md) - The rules a contributor must follow, the traps the code and docs warn about, and the points this survey could not settle.
* [Data model — targets, evidence, profiles, bundles, unknowns and snapshots](concepts/data-model.md) - The entities REA produces and stores, their identity rules, their relationships, and how they move between a session, a saved snapshot and an imported bundle.
* [Evidence and the domain layer — records, ledger, unknowns and comparisons](concepts/modules-evidence-and-domain.md) - The pure domain layer that defines Evidence, analysis profiles, snapshots, bundles, residual unknowns, error algebra and comparison semantics, plus the session-owned ledger that stores and authenticates them.
* [JavaScript and Electron analysis — static graph, source maps, recovery, runtime reconciliation](concepts/modules-javascript-and-electron.md) - The JavaScript/Electron subsystem: artifact reconstruction from directories and ASARs, the worker-isolated parser and semantic analyser, source-map decoding, Wakaru-based source recovery, Electron boundary analysis, V8 runtime observation and reconciliation with static facts.
* [MCP server adapter — transport, tools, delivery and identity](concepts/modules-mcp-server.md) - How REA serves its tool catalog over MCP: entry and startup modes, server classes, tool registration families, result delivery budgets, schema profiles, untrusted-content marking and the catalog identity tools report.
* [Native analysis providers — Hopper, Ghidra, IDA, Kuna, JEB and debuggers](concepts/modules-native-providers.md) - The engine-backed providers that supply pseudocode and assembly, how each is launched and spoken to, their bridges, configuration, and the shared rules for deadlines and cleanup.
* [Process supervision — ownership, deadlines, bounded output and cleanup](concepts/modules-process-supervision.md) - The process foundation that every provider and runtime tool builds on: owned process groups and run tokens, bounded output capture, deadlines, private runtime roots, worker and debugger lifecycles, and the Windows native controls.
* [Providers and sessions — selection, binding and the target lifecycle](concepts/modules-providers.md) - The provider abstraction, deterministic deep-provider selection, auxiliary provider composition, and the BinarySession that owns one active target, its profile, its snapshot cache and its evidence records.
* [REA overview — what it is, who uses it, main flows](concepts/overview.md) - An introduction to REA as a local-only reverse-engineering MCP server and CLI: its purpose, its users, the investigation loop it supports, and the set of targets it covers.
* [Setup, doctor, update and uninstall](concepts/modules-setup-and-update.md) - The host-integration surface: how `rea setup` plans and applies additive client registration and skill installation, how `rea doctor` reports readiness, and how update and uninstall keep the installation consistent.
* [Static artifacts — format readers and analysis services](concepts/modules-static-artifacts.md) - The non-engine half of REA: artifact readers and extractors, native format parsers, managed assemblies, and the per-format analysis services (Go, Rust, IL2CPP, GPU, models, Python, Erlang, WASM, EVM, SQLite, APK signatures, Android, firmware, packets, signals).

# Sources

* [Architecture diagram (capture, 2026-11-20)](sources/2026-11-20-architecture-diagram.md) - Verbatim copy of docs/architecture.mermaid, the repository's own component and dependency map for entry points, adapters, application layer, provider layers and contracts.
* [Build, test and CI configuration (capture, 2026-11-20)](sources/2026-11-20-build-and-test-config.md) - Toolchain pins, Turbo task graph, Vitest project ownership, coverage thresholds and the CI job inventory that define how REA is built and verified.
* [Contribution and documentation rules (capture, 2026-11-20)](sources/2026-11-20-contribution-and-docs-rules.md) - The repository's operating contract (AGENTS.md), developer setup and PR rules (CONTRIBUTING.md), tool-design rules and the docs guide map.
* [Evidence, bundle and unknown schemas (capture, 2026-11-20)](sources/2026-11-20-evidence-schema.md) - The canonical Evidence record schema, its semantic identity rule, the deterministic EvidenceBundle projection and the ResidualUnknown revision record that together form REA's persistence model.
* [MCP server wiring (capture, 2026-11-20)](sources/2026-11-20-mcp-server-wiring.md) - How the MCP entry point chooses deferred or eager startup, composes the BinarySession, starts the transport and registers tool families in createServer.
* [REA README (capture, 2026-11-20)](sources/2026-11-20-readme.md) - Excerpt of the repository README describing what REA is, who it is for, the setup/CLI quick start, supported target formats, and the documentation map.
* [Runtime configuration schema (capture, 2026-11-20)](sources/2026-11-20-configuration.md) - The Zod environment schema, provider-config projection and validation rules that define every REA_*/HOPPER_*/GHIDRA_* runtime setting.
* [package.json manifest and script catalog (capture, 2026-11-20)](sources/2026-11-20-package-manifest.md) - Identity, engines, bin entries and the npm script catalog of the rea-agents package, with the exact command strings used to build, check, test, verify and document the project.
