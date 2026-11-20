---
type: Concept
title: Process supervision — ownership, deadlines, bounded output and cleanup
description: The process foundation that every provider and runtime tool builds on: owned process groups and run tokens, bounded output capture, deadlines, private runtime roots, worker and debugger lifecycles, and the Windows native controls.
status: draft
lifecycle: draft
confidence: medium
created: 2026-11-20
updated: 2026-11-20
tags: [unverified]
sources:
  - resource: https://github.com/yishao1224-design/rea/blob/8c4f96ef4759fd78da66c4b9726825ca71d7ee0c/src/process/ProcessOwnership.ts
    id: process-ownership
    title: src/process/ProcessOwnership.ts
  - resource: https://github.com/yishao1224-design/rea/blob/8c4f96ef4759fd78da66c4b9726825ca71d7ee0c/src/process/ProviderProcess.ts
    id: provider-process
    title: src/process/ProviderProcess.ts
  - resource: https://github.com/yishao1224-design/rea/blob/8c4f96ef4759fd78da66c4b9726825ca71d7ee0c/src/process/OwnedCommand.ts
    id: owned-command
    title: src/process/OwnedCommand.ts
  - resource: https://github.com/yishao1224-design/rea/blob/8c4f96ef4759fd78da66c4b9726825ca71d7ee0c/src/process/RegexMatcher.ts
    id: regex-matcher
    title: src/process/RegexMatcher.ts
  - resource: https://github.com/yishao1224-design/rea/blob/8c4f96ef4759fd78da66c4b9726825ca71d7ee0c/src/process/LinuxResourceLimits.ts
    id: linux-limits
    title: src/process/LinuxResourceLimits.ts
  - resource: https://github.com/yishao1224-design/rea/blob/8c4f96ef4759fd78da66c4b9726825ca71d7ee0c/native/windows/README.md
    id: windows-readme
    title: native/windows/README.md
  - resource: https://github.com/yishao1224-design/rea/blob/8c4f96ef4759fd78da66c4b9726825ca71d7ee0c/docs/tool-design.md
    id: tool-design
    title: docs/tool-design.md
  - resource: https://github.com/yishao1224-design/rea/blob/8c4f96ef4759fd78da66c4b9726825ca71d7ee0c/AGENTS.md
    id: agents-rules
    title: AGENTS.md
generated:
  by: dsh/agent
  at: 2026-11-20
---

# Process supervision

**Job.** Make every child process, worker and debugger session an *owned*
resource whose identity, output, deadline and removal can be proven — and whose
failed cleanup is retained rather than forgotten. Nearly every provider in REA
depends on this layer.

The repository states the invariant that shapes it: "Reuse `src/process/`
supervision and identity primitives; a PID and executable path alone do not
establish ownership after exit or reuse" [agents-rules]. And: "Keep proof of
ownership separate from validity, editability, and readiness. Malformed owned
data may still be safely removable" [agents-rules].

## Key files

| Path | Role |
| --- | --- |
| `src/process/ProviderProcess.ts` | spawn/supervise owned provider processes; snapshots and stop results |
| `src/process/OwnedCommand.ts` | one-shot owned command with deadline, output limits, cleanup result [owned-command] |
| `src/process/ProcessOwnership.ts` | process groups, run tokens, baselines, sweeps, final verification [process-ownership] |
| `ProcessOwnershipObservation.ts`, `ProcessOwnershipEvidence.ts`, `ProcessOwnershipIdentity.ts`, `ProcessOwnershipProcessTree.ts` | host observation, evidence recording, identity checks, tree walks |
| `DarwinProcessRunTokenReader.ts`, `ProcessRunTokenObservations.ts` | macOS run-token helper and its observations |
| `LinuxProcessStat.ts`, `LinuxProcessStatus.ts`, `LinuxResourceLimits.ts` | Linux identity and prlimit ceilings [linux-limits] |
| `ProviderDeadline.ts`, `ProviderProcessOutput.ts`, `ExecFileOutput.ts`, `ExecFileBoundedOutput.ts` | deadlines and bounded diagnostics |
| `PrivateRuntimeRoot.ts`, `PrivateRuntimeRootUnavailableError.ts` | per-run private directories |
| `OwnedWorkerRuntime.ts`, `JavaScriptWorkerMemoryLimit.ts`, `RegexMatcher.ts` | worker lifetimes and heap requests [regex-matcher] |
| `HopperLease.ts`, `ownedDebuggerSession.ts`, `DetachedPgpSignature.ts` | engine-specific lease and debugger lifecycles |
| `ArtifactAnalysisResources.ts`, `PendingOperations.ts`, `queuedInterrupts.ts`, `SharedStartupAttempt.ts` | shared resource and startup coordination |
| `src/windows/*` (`WindowsNativeLoader`, `WindowsOwnedProcess`, `WindowsPrivateRuntime`) and `src/process/WindowsAuthority.ts`, `WindowsBatchCommand.ts` | Windows host boundary adapters |
| `native/windows/src/*.cc` | Node-API addon: NTFS admission, DACLs, handles, Job Objects [windows-readme] |

## Ownership model

- A spawn records a `runId`, leader PID, process group id, expected command,
  expected parent PID and (for captures) a prelaunch identity baseline
  (`OwnedProcessGroup`). Token-swept resources carry one
  `ProcessOwnershipEvidenceOwner` ledger for their whole lifetime
  [process-ownership].
- Cleanup may signal only members with token proof. Observation adds candidates
  (`sampledProcessGroupIds`) but "does not authorize signaling".
- Exit observations use process start identity (and macOS run tokens), not just
  a live PID; `ProcessLifecycle.ts` holds the settlement budget, verification
  interval and grace window used by final sweeps.
- `OwnedCommandFailure` distinguishes `cancelled`, `timeout`, `output-limit`
  and `process`, and retains the process snapshot, cleanup failure and
  preparation failure.

## Output and deadlines

- Retained stderr (and stdout unless a sink consumes it) is bounded by
  `diagnosticBytes`; `ProcessOutputTokenScanner` scans for tokens without
  unbounded retention.
- `docs/tool-design.md` states the shared supervisor keeps at most 64 KiB of
  additional diagnostics from an owned stop entry, including pending cleanup and
  retries, drains for up to one second after a verified stop, and marks output
  incomplete when closure cannot be observed [tool-design].
- Deadlines come in layers: provider startup
  (`REA_HOPPER_STARTUP_TIMEOUT_MS`, `REA_GHIDRA_STARTUP_TIMEOUT_MS`,
  `REA_JEB_MCP_TIMEOUT_MS`), per-command timeouts in tool inputs, regex matching
  (`REA_REGEX_MATCHING_DEADLINE_MS`), MCP startup phase budgets
  (`REA_MCP_*_BUDGET_MS`) and the client's own tool deadline. They are
  deliberately independent.
- Linux one-shot producers can be pinned with soft and hard prlimit ceilings
  (`address_space_bytes`, `file_bytes`, `cpu_seconds`) via
  `LinuxResourceLimits` [linux-limits].

## Workers

- `RegexMatcher` runs matching in a `worker_threads` worker with a
  `maxOldGenerationSizeMb` request from `REA_REGEX_WORKER_HEAP_REQUEST_MIB`
  (default 128 MiB). The comment documents the subtlety: a positive
  `--max-old-space-size` on the REA process (including `NODE_OPTIONS`)
  overrides the request, and `--max-old-space-size-percentage` makes the
  effective heap unestablished rather than assumed [regex-matcher].
- `OwnedWorkerRuntime` terminates and joins workers; the JavaScript analysis and
  web bundle workers are children of this runtime.

## Windows

- `src/windows/` contains the host boundary adapters (`WindowsNativeLoader`,
  `WindowsOwnedProcess`, `WindowsPrivateRuntime`); `src/process/WindowsAuthority.ts`
  and `WindowsBatchCommand.ts` hold the authority and batch-command rules.
- `native/windows/src/` implements NTFS admission, protected DACLs, opaque
  handles and atomic Job Object assignment in a Node-API addon
  (`addon.cc`, `filesystem.cc`, `process.cc`, `authority.hpp`,
  `handle_close.hpp`).
- The addon is **not** built by `npm ci` or `npm run build:cached`; building it
  requires Linux, MinGW-w64 and `tar`, and its artifacts stay uncommitted
  [windows-readme]. Windows workflows that need Job Object-based command
  locking therefore need that build first.

## Cleanup rules the layer enforces

1. Every owner is explicit; a consumer must use the owner's parsed facts.
2. A failed cleanup is reported and retained for retry, not dropped.
3. Cancellation must still await finalization; active-execution cancellation
   alone does not cover finalization.
4. Late handshakes, cancellation during final cleanup, mixed child outcomes and
   cleanup retry are all expected test cases (`docs/testing.md`
   "lifecycle regressions").

## Pitfalls

- Treating a successful launcher exit as proof the target stopped.
- Treating a PID plus executable path as ownership after exit or reuse.
- Assuming output closed because the process exited.
- Adding a speculative race/security guard without a concrete failure mode,
  likelihood and complexity argument — AGENTS.md asks for user buy-in first.

## Related pages

- [Providers and sessions](/concepts/modules-providers.md)
- [Native analysis providers](/concepts/modules-native-providers.md)
- [Conventions, pitfalls and open questions](/concepts/conventions-pitfalls-open-questions.md)
- [Build, run, test and deploy](/concepts/build-run-test-deploy.md)
