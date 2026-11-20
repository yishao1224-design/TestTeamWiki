---
type: Concept
title: Browser and web analysis — CDP capture, scenarios, historical captures, module tracing
description: The web-facing subsystem: CDP observation of pages and scripts, Playwright-driven scenarios, HAR/SAZ/mitmproxy/qlog historical captures, bundle analysis, source-location tracing, network-capture projection and serving a recorded environment.
status: draft
lifecycle: draft
confidence: medium
created: 2026-11-20
updated: 2026-11-20
tags: [unverified]
sources:
  - resource: https://github.com/yishao1224-design/rea/blob/8c4f96ef4759fd78da66c4b9726825ca71d7ee0c/src/browser/CdpBrowserProvider.ts
    id: cdp-provider
    title: src/browser/CdpBrowserProvider.ts
  - resource: https://github.com/yishao1224-design/rea/blob/8c4f96ef4759fd78da66c4b9726825ca71d7ee0c/src/browser/PlaywrightBrowserScenarioProvider.ts
    id: scenario-provider
    title: src/browser/PlaywrightBrowserScenarioProvider.ts
  - resource: https://github.com/yishao1224-design/rea/blob/8c4f96ef4759fd78da66c4b9726825ca71d7ee0c/src/browser/history/HistoricalCaptureDecoder.ts
    id: history
    title: src/browser/history/HistoricalCaptureDecoder.ts
  - resource: https://github.com/yishao1224-design/rea/blob/8c4f96ef4759fd78da66c4b9726825ca71d7ee0c/src/application/WebNetworkCaptureService.ts
    id: network-service
    title: src/application/WebNetworkCaptureService.ts
  - resource: https://github.com/yishao1224-design/rea/blob/8c4f96ef4759fd78da66c4b9726825ca71d7ee0c/src/application/WebSourceLocationService.ts
    id: source-location
    title: src/application/WebSourceLocationService.ts
  - resource: https://github.com/yishao1224-design/rea/blob/8c4f96ef4759fd78da66c4b9726825ca71d7ee0c/src/application/environment/environmentReplay.ts
    id: environment-replay
    title: src/application/environment/environmentReplay.ts
  - resource: https://github.com/yishao1224-design/rea/blob/8c4f96ef4759fd78da66c4b9726825ca71d7ee0c/docs/browser-scenario-contract.md
    id: scenario-doc
    title: docs/browser-scenario-contract.md
  - resource: https://github.com/yishao1224-design/rea/blob/8c4f96ef4759fd78da66c4b9726825ca71d7ee0c/docs/web-runtime.md
    id: runtime-doc
    title: docs/web-runtime.md
generated:
  by: dsh/agent
  at: 2026-11-20
---

# Browser and web analysis

**Job.** Observe a website or Electron renderer — either live through the Chrome
DevTools Protocol (CDP) or Playwright, or after the fact through saved network
captures — and return page structure, scripts, network activity, execution
coverage and source attribution as Evidence.

## Key files

| Path | Role |
| --- | --- |
| `src/browser/CdpBrowserProvider.ts` | passive/attached CDP observation of a page target [cdp-provider] |
| `src/browser/CdpPageCapture*.ts`, `CdpCapture*.ts` | documents, scripts, workers, storage, screenshots, event listeners, network |
| `src/browser/execution/*` | precise V8 coverage, runtime initiators, execution windows, listener selection |
| `src/browser/PlaywrightBrowserScenarioProvider.ts` | declarative scenario execution [scenario-provider] |
| `src/browser/PlaywrightScenario*.ts`, `ScenarioActionSequence.ts` | actions, artifacts, storage, service workers, deadlines |
| `src/browser/history/*` | HAR, Fiddler SAZ, mitmproxy, qlog, node-capture decoders and redaction [history] |
| `src/browser/modules/*` | native module resolution and module artifacts |
| `src/browser/WebBundleAnalysisWorker*.ts` | worker-isolated bundle analyzer |
| `src/browser/PngVisualDiff.ts` | screenshot comparison |
| `src/application/WebNetworkCaptureService.ts` | capture inspection, API-surface and environment manifests [network-service] |
| `src/application/WebSourceLocationService.ts` | source-position tracing [source-location] |
| `src/application/WebModuleTraceService.ts` | native import/re-export tracing from an exported script |
| `src/application/WebRuntimeService.ts`, `WebScriptExportService.ts` | runtime observation and script export |
| `src/application/environment/*` | serve/export/compare a recorded environment [environment-replay] |

## Interfaces

### Live observation

- `list_browser_targets`, `inspect_web_page`, `observe_web_execution`,
  `inspect_web_event_listeners`, `capture_web_screenshot`,
  `compare_web_screenshots`, `discover_webmcp_tools`,
  `capture_browser_scenario`.
- Attached CDP use requires a caller-supplied loopback endpoint and target id;
  the browser stays externally owned, and REA detaches its own connection when
  done. Precise coverage resets counters and disables optimisation while armed,
  which changes JIT/timing behaviour — the documentation states this authority
  explicitly [runtime-doc].
- Scenario capture is declarative: fixed action vocabulary, explicit HTTP(S)
  targets, deterministic settings, explicit storage seeds, provider-owned
  liveness deadlines. `command_timeout_ms` (default 20,000 ms) bounds launch,
  CDP handshake, target lookup and environment/storage commands; it does not
  change action, wait or navigation timeouts, and `capture-browser-scenario` has
  no CLI flag for it [scenario-doc].
- Service workers are blocked by default; `environment.service_workers: allow`
  opts in. Blocking rejects captures when the selected page is controlled,
  including after navigation, and the rejection does not roll back browser
  effects [scenario-doc].

### Historical captures

`inspect_web_network_capture` / `inspect-web-network-capture` accept a capture
and a format. Supported readers: HAR, Fiddler Everywhere SAZ, mitmproxy flow
files (via the bundled `bridge/mitmproxy/capture.py` on Linux), qlog, and a
node capture adapter. `CaptureRedaction` and `BrowserScenarioSecrets` strip
declared secrets; `recordedTransportUrl` and `qlogCapture` own URL/transport
semantics.

### Projections

`inspect-analysis-view` turns retained capture Evidence into named views
(`src/domain/analysisView/analysisView.ts`): `summary`, `page`,
`environment-manifest`, `environment-state`, `api-surface`, `javascript-routes`,
`protobuf`, `qlog`, `pcap`, `capture`, `source-to-bundle`, plus format-specific
facets. `environment-serve` then
serves an environment manifest as an MCP server answering from the observed
state (or replaying records with `--replay-only`), and `environment-export`
writes the same pair as a bundle; nothing is fetched from the recorded
application [environment-replay].

### Scripts and modules

`export_web_scripts` writes selected scripts to an owned directory with
containment checks; `trace_web_module_imports` resolves native ES imports and
re-exports using a caller-supplied Chromium for `import.meta.resolve`, executing
only a trusted REA resolver stub with page requests locally fulfilled
(`src/contracts/webScriptToolContracts.ts`); `trace_web_source_location` maps a
location to captured source.

## Dependencies

- `REA_BROWSER_EXECUTABLE` (absolute) for a caller-supplied Chromium-family
  browser; REA installs no browser.
- Playwright for scenario automation; CDP is spoken directly for passive and
  precise-coverage work.
- `src/process` for owned browser processes and worker runtimes.
- `src/domain/web*.ts` for capture value types, diffs and schemas.

## Pitfalls

- CDP line numbers are zero-based and columns are UTF-16; documented
  conversions to script-relative positions must be applied, and unknown
  coordinates preserved.
- Nested/overlapping coverage counters must not be summed.
- A `command_timeout_ms` raise does not raise action or navigation timeouts, and
  the service-worker registration check keeps a fixed 10,000 ms deadline.
- Capture authority differs: a controlled replay is not proof that the recorded
  application runs the same way today.

## Related pages

- [JavaScript and Electron](/concepts/modules-javascript-and-electron.md)
- [Static artifacts](/concepts/modules-static-artifacts.md)
- [Process supervision](/concepts/modules-process-supervision.md)
- [Data model](/concepts/data-model.md)
