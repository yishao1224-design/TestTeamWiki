---
type: Source
title: Runtime configuration schema (capture, 2026-11-20)
description: The Zod environment schema, provider-config projection and validation rules that define every REA_*/HOPPER_*/GHIDRA_* runtime setting.
status: draft
lifecycle: draft
created: 2026-11-20
updated: 2026-11-20
source_url: https://github.com/yishao1224-design/rea/blob/8c4f96ef4759fd78da66c4b9726825ca71d7ee0c/src/config/environment.ts
retrieved: 2026-11-20
tags: [unverified]
generated:
  by: dsh/agent
  at: 2026-11-20
---

# src/config/environment.ts and src/config/parseConfig.ts (capture)

Excerpt of the two files that own runtime configuration. Quoted fragments are
verbatim; comments and type-level detail are summarised.

## The environment schema (environment.ts, lines 35-99)

```ts
const environmentSchema = z.object({
  REA_ANALYSIS_PROVIDER: analysisProviderSelectorSchema.default("auto"),
  REA_BRIDGE_MAX_REQUEST_BYTES: bridgeMaxRequestBytesSchema.optional(),
  REA_MCP_MAX_RESPONSE_BYTES: mcpResponseBudgetSchema.optional(),
  REA_ASAR_MAX_HEADER_BYTES: asarMaxHeaderBytesSchema.optional(),
  REA_CAPTURE_DECODER_OLD_GENERATION_MIB: captureDecoderOldGenerationMiBSchema.optional(),
  REA_REGEX_MATCHING_DEADLINE_MS: regexMatchingDeadlineMsSchema.optional(),
  REA_REGEX_WORKER_HEAP_REQUEST_MIB: regexWorkerHeapRequestMiBSchema.optional(),
  REA_MCP_INITIALIZE_BUDGET_MS: mcpInitializeBudgetMsSchema.optional(),
  REA_MCP_TOOLS_LIST_BUDGET_MS: mcpToolsListBudgetMsSchema.optional(),
  REA_MCP_FIRST_CATALOG_BUDGET_MS: mcpFirstCatalogBudgetMsSchema.optional(),
  REA_MCP_DOCTOR_DEADLINE_MS: mcpDoctorDeadlineMsSchema.optional(),
  REA_MCP_INPUT_SCHEMA_PROFILE: mcpInputSchemaProfileSchema.optional(),
  REA_MCP_OUTPUT_SCHEMA_PROFILE: mcpOutputSchemaProfileSchema.optional(),
  REA_IDA_MCP_CONFIG: z.string().min(1).refine(isAbsolute, "REA_IDA_MCP_CONFIG must be absolute").optional(),
  REA_JEB_MCP_TIMEOUT_MS: providerStartupTimeoutSchema.optional(),
  GHIDRA_INSTALL_DIR: z.string().min(1).refine(isAbsolute, ...).optional(),
  JAVA_HOME: z.string().min(1).refine(isAbsolute, ...).optional(),
  REA_GHIDRA_NATIVEAOT_JAR: z.string().min(1).refine(isAbsolute, ...).optional(),
  REA_GHIDRA_STARTUP_TIMEOUT_MS: providerStartupTimeoutSchema.default(DEFAULT_GHIDRA_STARTUP_TIMEOUT_MS),
  REA_HOPPER_STARTUP_TIMEOUT_MS: providerStartupTimeoutSchema.default(DEFAULT_HOPPER_STARTUP_TIMEOUT_MS),
  REA_SWIFTC_COMPILE_TIMEOUT_MS: providerStartupTimeoutSchema.default(DEFAULT_SWIFTC_COMPILE_TIMEOUT_MS),
  REA_GHIDRA_LANGUAGE_ID: ghidraLanguageIdSchema.optional(),
  REA_GHIDRA_COMPILER_SPEC_ID: ghidraCompilerSpecIdSchema.optional(),
  REA_GHIDRA_DISABLED_ANALYZERS: ghidraDisabledAnalyzersSchema.optional(),
  REA_GHIDRA_SEED_FILE: z.string().min(1).refine(isAbsolute, ...).optional(),
  REA_ILSPY_CMD_PATH: z.string().min(1).refine(isAbsolute, ...).optional(),
  HOPPER_LAUNCHER_PATH: z.string().min(1).optional(),
  HOPPER_TARGET_PATH: z.string().min(1).optional(),
  HOPPER_TARGET_KIND: z.enum(["executable", "database"]).default("executable"),
  HOPPER_LOADER_ARGS_JSON: z.string().optional(),
  REA_LOG_LEVEL: processStartConfigurationSchema.shape.REA_LOG_LEVEL,
  REA_REFERENCE_SECRET_PATTERNS_JSON: z.string().default("[]"),
});
```

## Validation rules (environment.ts)

- Environment variable names and values containing NUL bytes are rejected
  before the schema strips unknown keys (lines 134-156).
- `parseEnvironmentValues` parses the schema, then decodes
  `HOPPER_LOADER_ARGS_JSON` (`parseLoaderArgs`) and
  `REA_REFERENCE_SECRET_PATTERNS_JSON` (`parseStringArray`) (lines 176-187).
- Zod messages name the constraint, never the rejected value; the error is a
  `ConfigurationError` carrying `settings: [{ setting, constraint }]`
  (lines 157-170).
- `parseEnvironment` adds one cross-field rule: `REA_GHIDRA_COMPILER_SPEC_ID`
  requires `REA_GHIDRA_LANGUAGE_ID`, because Ghidra's `analyzeHeadless`
  accepts `-cspec` only with `-processor` (lines 199-214).
- `parseJebMcpTimeoutMs` parses `REA_JEB_MCP_TIMEOUT_MS` at the optional JEB
  provider boundary so an invalid value cannot block unrelated startup
  (lines 101-120).

## Provider projection (parseConfig.ts, lines 69-149)

`parseConfig` calls `parseEnvironment`, then builds `AppConfig`:

```ts
export const parseConfig = (environment): Result<AppConfig, ConfigurationError> => {
  const parsed = parseEnvironment(nativeAnalysisEnvironment(environment));
  if (!parsed.ok) return parsed;
  const env = parsed.value;
  return ok({
    analysisProvider: env.REA_ANALYSIS_PROVIDER,
    ghidraInstallDir: env.GHIDRA_INSTALL_DIR,
    ghidraJavaHome: env.JAVA_HOME,
    ghidraStartupTimeoutMs: env.REA_GHIDRA_STARTUP_TIMEOUT_MS,
    ilspyCmdPath: env.REA_ILSPY_CMD_PATH,
    hopperLauncherPath:
      env.HOPPER_LAUNCHER_PATH ??
      defaultHopperLauncherPath(process.platform, homeDirectoryFromEnvironment(environment, process.platform)),
    hopperStartupTimeoutMs: env.REA_HOPPER_STARTUP_TIMEOUT_MS,
    hopperTargetPath: env.HOPPER_TARGET_PATH,
    hopperTargetKind: env.HOPPER_TARGET_KIND,
    hopperLoaderArgs: env.loaderArgs,
    logLevel: env.REA_LOG_LEVEL,
    referenceSourcePolicy: { secretPatterns: env.referenceSecretPatterns },
    bridgeMaxRequestBytes: /* default or { maximumBytes, source: "setting" } */,
    regexMatchingDeadline: /* default or { deadlineMs, source: "setting" } */,
    regexWorkerHeapRequest: /* default or { heapMiB, source: "setting" } */,
  });
};
```

Notes carried by the comments in `parseConfig.ts`:

- `nativeAnalysisEnvironment` removes `REA_JEB_MCP_TIMEOUT_MS` before native
  configuration is validated (lines 50-56).
- `defaultHopperLauncherPath` probes platform candidates and falls back to
  `MAC_HOPPER_SYSTEM_LAUNCHER` / `REA_MANAGED_LINUX_HOPPER_LAUNCHER`
  (lines 28-43).
- Doctor uses `nativeAnalysisConfigurationError` so readiness and analysis
  admit the same settings (lines 58-67).

## Environment variables referenced outside the schema

Not part of `environmentSchema` but read by composition code or provider
adapters, and therefore part of the runtime contract:
`REA_JEB_MCP_URL` (default `http://127.0.0.1:8425/mcp`),
`REA_WABT_BIN_DIRECTORY`, `REA_BINWALK_COMMAND`, `REA_UNBLOB_COMMAND`,
`REA_WAKARU_COMMAND`, `REA_PWNTOOLS_PYTHON`, `REA_SIGROK_COMMAND`,
`REA_SIGROK_PRLIMIT_COMMAND`, `REA_TSHARK_COMMAND`,
`REA_TSHARK_PRLIMIT_COMMAND`, `REA_CAPINFOS_COMMAND`,
`REA_BROWSER_EXECUTABLE`, `PLAYWRIGHT_BROWSERS_PATH`.

Injection rule used by `src/server/createServer.ts` (lines 277-364): a
caller-supplied tool counts as configured only when its variable holds an
absolute path; otherwise the capability is reported unavailable for that host
rather than probed at startup.
