---
type: Concept
title: Static artifacts — format readers and analysis services
description: The non-engine half of REA: artifact readers and extractors, native format parsers, managed assemblies, and the per-format analysis services (Go, Rust, IL2CPP, GPU, models, Python, Erlang, WASM, EVM, SQLite, APK signatures, Android, firmware, packets, signals).
status: draft
lifecycle: draft
confidence: medium
created: 2026-11-20
updated: 2026-11-20
tags: [unverified]
sources:
  - resource: https://github.com/yishao1224-design/rea/blob/8c4f96ef4759fd78da66c4b9726825ca71d7ee0c/src/artifacts/ArtifactProvider.ts
    id: artifact-provider
    title: src/artifacts/ArtifactProvider.ts
  - resource: https://github.com/yishao1224-design/rea/blob/8c4f96ef4759fd78da66c4b9726825ca71d7ee0c/src/artifacts/SafeOutputTree.ts
    id: safe-output
    title: src/artifacts/SafeOutputTree.ts
  - resource: https://github.com/yishao1224-design/rea/blob/8c4f96ef4759fd78da66c4b9726825ca71d7ee0c/src/native/NativeMacOSProvider.ts
    id: native-provider
    title: src/native/NativeMacOSProvider.ts
  - resource: https://github.com/yishao1224-design/rea/blob/8c4f96ef4759fd78da66c4b9726825ca71d7ee0c/src/dotnet/ManagedStaticProvider.ts
    id: managed-provider
    title: src/dotnet/ManagedStaticProvider.ts
  - resource: https://github.com/yishao1224-design/rea/blob/8c4f96ef4759fd78da66c4b9726825ca71d7ee0c/src/composition/apkSignature.ts
    id: apksig-composition
    title: src/composition/apkSignature.ts
  - resource: https://github.com/yishao1224-design/rea/blob/8c4f96ef4759fd78da66c4b9726825ca71d7ee0c/src/apkSignature/ApkSignatureService.ts
    id: apksig-service
    title: src/apkSignature/ApkSignatureService.ts
  - resource: https://github.com/yishao1224-design/rea/blob/8c4f96ef4759fd78da66c4b9726825ca71d7ee0c/src/application/android/AndroidAnalysisService.ts
    id: android-service
    title: src/application/android/AndroidAnalysisService.ts
  - resource: https://github.com/yishao1224-design/rea/blob/8c4f96ef4759fd78da66c4b9726825ca71d7ee0c/src/application/firmware/FirmwareAnalysisService.ts
    id: firmware-service
    title: src/application/firmware/FirmwareAnalysisService.ts
  - resource: https://github.com/yishao1224-design/rea/blob/8c4f96ef4759fd78da66c4b9726825ca71d7ee0c/docs/index.md
    id: docs-index
    title: docs/index.md
generated:
  by: dsh/agent
  at: 2026-11-20
---

# Static artifacts

**Job.** Read files — no engine, and often no execution — and return typed
facts: inventories, digests, headers, metadata, declarations, members, decoded
resources. Most of this surface is registered as its own service and tool family
rather than as a deep provider, so it works on any host where the input exists.

## 1. Artifact readers and extraction

`src/artifacts/` is the boundary that turns a path into bounded reads:

- `ArtifactProvider` — the provider that exposes inventory, classification,
  extraction and per-format inspection operations [artifact-provider].
- Readers: `ZipArtifactReader`, `AsarArtifactReader`,
  `GodotPckArtifactReader`, `NativeDmgArtifactReader`,
  `MachOSliceArtifactReader`, `PyInstallerArtifactReader`,
  `DirectoryArtifactReader`, `DesktopApplicationReader`.
- `ArtifactReader`, `readStableArtifact`, `ArtifactHash`, `readArtifactFileChunks`,
  `OrderedReads`, `BoundedFileBytes` — bounded, identity-checked reads.
- `SafeOutputTree`, `SafeOutputTreeCreationFailure`, `SafeOutputTreeReadback`,
  `SafeOutputTreeCleanup` — extraction into a new REA-owned directory, refusing
  to overwrite caller files [safe-output].
- `ArtifactResourceScope`, `AnalysisCleanupLease`, `ArtifactFileOperation` — the
  retained-resource ownership used by all readers.
- `src/artifacts/apple/*` — plists, keyed archives (NSKeyedArchiver), NIB
  archives, Interface Builder, asset catalogs, Mach-O load commands, dylib
  resolution.
- `src/artifacts/pyinstaller/*` — CArchive, PYZ store, marshal subset, Python
  bytecode.

The desktop-application workflow (`inspect_desktop_application`) is built on
this: bounded directory and header inspection to produce runtime candidates,
supporting files, component markers and next probes.

## 2. Native format parsers

`src/native/` reads binary structure without an engine:

- `metadata/BinaryContainer`, `ElfImage`, `MachoImage`, `PeImage`,
  `readBinaryContainer`, `MachoSliceSelection`, `NativeSectionDirectory`.
- `parsers/`: codesign, demangle, dyld info, lipo, Mach-O UUID, otool output,
  plist.
- `elf/` (notes), `pe/` (certificates, resources), `exports/`,
  `AppleObjc*`, `AppleSwift*`, `AppleDispatch*` (Objective-C/Swift/dispatch
  metadata), `NativeCallTarget`, `LldbCallTracer`, `NativeUiObservation`,
  `SignatureTargetBinding`.
- `NativeMacOSProvider` exposes these as capabilities, including
  `inspect_native_*` operations and bundle-resource lanes [native-provider].

## 3. Managed code (.NET)

`src/dotnet/ManagedStaticProvider.ts` reads PE/CLI metadata and CIL without
loading or executing the assembly [managed-provider]: identity and deployment
form, member inventory, IL, declared native dependencies, comparisons and
managed/native boundary verification. `REA_ILSPY_CMD_PATH` is an optional
absolute ILSpy command path. IL2CPP is explicitly out of scope here; it has its
own metadata reader.

## 4. Per-format analysis services

Each of these is a service class with an `inspect*` method, a contract module in
`src/contracts/<format>/`, a composition factory, a CLI command and one or more
MCP tools.

| Format | Service / module | Inputs and dependencies |
| --- | --- | --- |
| Go binaries | `src/application/go/GoBinaryService` + `src/go/` | built-in ELF/PE/thin Mach-O parsing; no Go install |
| Rust binaries | `RustBinaryService` + `src/rust/` | cargo-auditable sections |
| IL2CPP metadata | `Il2cppMetadataService` + `src/il2cpp/` | `global-metadata.dat` versions 24/27/29/31/39; optional matching binary and Ghidra seeds |
| GPU code objects | `src/application/gpu/` + `src/gpu/` | ELF/PE/fatbin/CUBIN/PTX |
| Model artifacts | `ModelArtifactService` + `src/model/` | GGUF/safetensors headers; optional full-file hash |
| Python bytecode / PyInstaller | `PythonPycService`, `PyInstallerArtifactReader` | `.pyc` headers; CArchive/PYZ |
| Erlang BEAM | `BeamService` + `src/erlang/` | IFF chunks, atoms, exports |
| WASM | `src/wasm/` (`WabtArtifactProvider`) | caller-supplied WABT 1.0.42 via `REA_WABT_BIN_DIRECTORY` |
| EVM bytecode | `src/evm/` | EVMole/WASM in a bounded owned worker; local raw/hex carrier, Linux x64 |
| APK signature | `ApkSignatureService` + `src/apkSignature/` | in-process signing-block/v1 reader, no device, no execution [apksig-service] |
| SQLite | `SqliteDatabaseService` + `src/sqlite/` | local DB/WAL snapshot; Node `node:sqlite` |
| PCAP summaries | `PcapAnalysisService` + `src/pcap/` | caller `tshark`/`capinfos` |
| Packet fields | `src/captures/` | TShark 4.x + prlimit on Linux |
| Signal recordings | `src/signals/` | sigrok-cli 0.7.2 + prlimit on Linux |
| Firmware | `FirmwareAnalysisService` + `src/firmware/` | Binwalk/Unblob on Linux, private extraction [firmware-service] |
| Android APK | `AndroidAnalysisService` + `src/android/` | pinned headless JADX + full JDK [android-service] |
| Android resources | `ApktoolResourceAnalysisService` + `src/apktool/` | caller Apktool launcher |
| Android devices | `AdbDeviceAnalysisService` + `src/adb/` | caller adb |
| Flutter | `FlutterBuildAnalysisService` + `src/flutter/` | static APK parsing |
| Harmony | `HarmonyApplicationService` + `src/domain/harmony/harmonyApplication.ts` | static HAP/HSP/APP inspection and application-graph projection |
| JEB projects | `JebAnalysisService` + `src/jeb/` | external JEB MCP endpoint |
| Desktop apps | `DesktopApplicationService` + `src/artifacts/Desktop*` | built-in |

`docs/index.md` maps these to the guide that documents each one [docs-index].

## Cross-cutting patterns

- **Capability availability is host-verified.** A format whose tool is missing
  is reported unavailable with remediation, not silently absent; injected
  services are assumed to own their availability
  (`src/server/createServer.ts`, `installSessionToolAvailability`).
- **Static means static.** The README separates "static inspection" from
  "runtime capture": static readers do not run the application; runtime
  providers state their launch/attach effects.
- **Deterministic ordering and digests.** Inventories sort deterministically,
  evidence uses SHA-256 subject digests, and extractors write only into new
  directories.
- **Partial facts are normal.** Metadata readers return what the format
  establishes and mark the rest unknown (for example, model hashing defaults to
  headers only).

## Pitfalls

- Format recognition is not admission: a recognised header can yield useful
  facts while later structure remains unresolved.
- Extraction output directories must be new; `SafeOutputTree` refuses to merge
  into caller content.
- Windows source checkouts cannot build the native addon that the Windows
  native controls need (see [build, run, test and deploy](/concepts/build-run-test-deploy.md)).
- Several formats require Linux-only helpers (prlimit, Binwalk/Unblob,
  sigrok-cli); the README's table is the authority on host support.

## Related pages

- [JavaScript and Electron](/concepts/modules-javascript-and-electron.md)
- [Browser and web analysis](/concepts/modules-browser-and-web.md)
- [Providers and sessions](/concepts/modules-providers.md)
- [Data model](/concepts/data-model.md)
