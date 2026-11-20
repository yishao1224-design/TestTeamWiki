---
type: Concept
title: Setup, doctor, update and uninstall
description: The host-integration surface: how `rea setup` plans and applies additive client registration and skill installation, how `rea doctor` reports readiness, and how update and uninstall keep the installation consistent.
status: draft
lifecycle: draft
confidence: medium
created: 2026-11-20
updated: 2026-11-20
tags: [unverified]
sources:
  - resource: /sources/2026-11-20-readme.md
    id: readme
    title: README capture (2026-11-20)
  - resource: https://github.com/yishao1224-design/rea/blob/8c4f96ef4759fd78da66c4b9726825ca71d7ee0c/src/application/Setup.ts
    id: setup
    title: src/application/Setup.ts
  - resource: https://github.com/yishao1224-design/rea/blob/8c4f96ef4759fd78da66c4b9726825ca71d7ee0c/src/application/SetupPlan.ts
    id: setup-plan
    title: src/application/SetupPlan.ts
  - resource: https://github.com/yishao1224-design/rea/blob/8c4f96ef4759fd78da66c4b9726825ca71d7ee0c/src/application/SupportedClients.ts
    id: clients
    title: src/application/SupportedClients.ts
  - resource: https://github.com/yishao1224-design/rea/blob/8c4f96ef4759fd78da66c4b9726825ca71d7ee0c/src/application/Doctor.ts
    id: doctor
    title: src/application/Doctor.ts
  - resource: https://github.com/yishao1224-design/rea/blob/8c4f96ef4759fd78da66c4b9726825ca71d7ee0c/src/application/Update.ts
    id: update
    title: src/application/Update.ts
  - resource: https://github.com/yishao1224-design/rea/blob/8c4f96ef4759fd78da66c4b9726825ca71d7ee0c/src/application/Uninstall.ts
    id: uninstall
    title: src/application/Uninstall.ts
  - resource: https://github.com/yishao1224-design/rea/blob/8c4f96ef4759fd78da66c4b9726825ca71d7ee0c/docs/installation.md
    id: installation-doc
    title: docs/installation.md
generated:
  by: dsh/agent
  at: 2026-11-20
---

# Setup, doctor, update and uninstall

**Job.** Integrate REA into an agent host without clobbering the host's
configuration, then keep that integration healthy and current.

The repository's operating contract is unusually explicit here: "Configuration
changes must be additive and idempotent, with backups. Installers must not
install or upgrade unrelated software, including Homebrew, Node.js, npm, Java,
or Ghidra. […] `rea setup` must show its planned changes and require approval
before writing files or installing Hopper."

## Key files

| Path | Role |
| --- | --- |
| `src/application/Setup.ts` | `runSetup`: discover → plan → confirm → apply, with cleanup that can downgrade the result to `needs_human` [setup] |
| `src/application/SetupPlan.ts` | `discoverSetupState`, `planSetupActions`, `hopperInstallRefusal` [setup-plan] |
| `src/application/SetupClients.ts`, `SetupHost.ts`, `SetupTypes.ts` | per-client application and host effects |
| `src/application/ClientConfigurationDocument.ts` (59 KB), `ClientConfigurationTransaction.ts`, `ClientConfigurationPublication.ts`, `ClientConfigurationFiles.ts`, `ClientConfigurationCleanup.ts` | JSON/TOML/YAML document edits with atomic publication |
| `src/application/ClientRegistrationStatus.ts`, `ClientRegistrationIdentity.ts`, `ClientRegistrationEnvironment.ts` | what is registered, under which identity, with which env |
| `src/application/SetupSkill.ts`, `SetupRecoveryBackup.ts`, `SetupInstallFailure.ts` | skill install, backups, Hopper install failure handling |
| `src/application/ClaudeDesktopConfigLocation.ts`, `GeminiClientSettings.ts`, `HermesConfigurationYaml.ts`, `DshCordisPatch.ts` | client-specific format adapters |
| `src/application/MacHopper*.ts`, `LinuxHopper*.ts`, `HopperDownload.ts`, `HopperReleaseFeed.ts` | verified Hopper acquisition |
| `src/application/Doctor.ts`, `DoctorDiagnostics.ts`, `DoctorScope.ts`, `DoctorProviderIds.ts` | readiness report [doctor] |
| `src/application/Update.ts`, `UpdateRuntime.ts`, `UpdateMaintenance.ts`, `UpdateCommandProgress.ts` | verified updater [update] |
| `src/application/Uninstall.ts`, `ClientConfigurationCleanup.ts` | removal [uninstall] |
| `src/cli/setupCommands.ts`, `interactiveSetup.ts` | CLI wiring and prompts |

## Interfaces

### Supported clients

`SUPPORTED_CLIENT_DEFINITIONS` names the integrations: `claude_code`,
`claude_desktop`, `codex`, `cursor`, `gemini_cli`, `windsurf`, `devin`,
`opencode`, `antigravity`, `copilot_cli`, `commandcode`, `qwen_code`, `kimi`,
`vscode`, `trae`, `workbuddy`, `grok_build`, `omp`, `pi`, `hermes` [clients].
The README advertises Claude Code, Codex, Cursor, Gemini CLI and Grok Build
explicitly and points at `docs/installation.md#supported-agents` for the rest
[readme].

### `rea setup`

Options (from `src/cli/setupCommands.ts`): `--yes` (approve user-owned actions
without prompting), `--install-hopper` (also approve Hopper installation with
`--yes`), `--client` (repeatable), `--skill-client` (repeatable, `shared` or a
client id), `--all-detected`, `--skill`.

Behaviour visible in code and docs:

- Discovery and planning happen before any write; `planSetupActions` produces
  the action list, and the interactive flow renders it for approval.
- Configuration edits are additive and idempotent, with backups
  (`SetupRecoveryBackup`) and rollback evidence (`Setup.configuration`,
  `Setup.recovery` tests).
- Manually configured collections preserved: unknown client-specific entries
  are carried across the child-process boundary
  (`SetupRegistrationEnvironment`, `ClientRegistrationEnvironment`).
- Hopper installation requires a separate approval and uses verified vendor
  downloads; a failed install is reported rather than silently skipped.
- Setup output policy is `agent-only` for the plan, so an agent sees the plan
  and the human approves.

### `rea doctor` and `rea mcp doctor`

`Doctor.ts` collects checks (Node runtime version, configuration validity,
provider launches, Hopper/Ghidra/IDA/Kuna/JEB readiness, installed skill
identity, client registrations, catalog identity) into a scoped report
(`DoctorScope`: all or one area). It uses
`nativeAnalysisConfigurationError` so readiness and analysis admit the same
settings, and `DEFAULT_CATALOG_IDENTITY` to report schema identity. The
in-process MCP doctor (`src/mcpDoctor.ts`,
`REA_MCP_DOCTOR_DEADLINE_MS`) probes a real server handshake with phase budgets
(`REA_MCP_INITIALIZE_BUDGET_MS`, `REA_MCP_TOOLS_LIST_BUDGET_MS`,
`REA_MCP_FIRST_CATALOG_BUDGET_MS`).

### `rea update`

`Update.ts` identifies the owning installation (`npm` prefix or Bun global
directory), looks up the latest version, installs exactly that version, verifies
the installed version, and plans integration maintenance separately [update].
The README adds two constraints: updater commands retain at most 4 MiB of
stdout+stderr combined, and Bun global updates require Bun 1.2.17 or later.
`rea --update` is an alias that accepts only global output flags; Incur's own
installer is rejected so an ambiguous flag cannot bypass the verified workflow.

### `rea uninstall`

`Uninstall.ts` removes REA's registration entries and skill while preserving
unrelated configuration, and reports what it could not clean up.

## Dependencies

- Host filesystem and each client's config format; no network except the
  release metadata lookup and Hopper downloads.
- `src/process` for owned `npm`/`bun`/`brew` invocations and output bounds.
- `src/application/JsonFiles`, `JsonInputFile`, `Utf8JsonInput`,
  `JsonAuthenticationHeadroom` for bounded, authenticated reads.

## Pitfalls

- Setup must be safe to rerun; a second run should be a no-op, not a duplicate
  entry.
- Backups are part of the contract, not a convenience.
- `--yes` approves user-owned actions, but Hopper needs `--install-hopper` as
  well.
- Doctor must keep running on an invalid `REA_LOG_LEVEL` so it can name the
  setting; other commands stop.
- After an update the agent's registration may still point at the old version;
  setup should be refreshed (`npx rea-agents@latest setup`) [readme].

## Related pages

- [CLI module](/concepts/modules-cli.md)
- [MCP server](/concepts/modules-mcp-server.md)
- [Native analysis providers](/concepts/modules-native-providers.md)
- [Build, run, test and deploy](/concepts/build-run-test-deploy.md)
