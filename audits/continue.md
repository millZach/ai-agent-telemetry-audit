# Privacy & Telemetry Forensic Audit: Continue (`continuedev/continue`)

## Metadata

| Field | Value |
|---|---|
| **Slug** | `continue` |
| **Platform** | Continue |
| **Group** | B |
| **Repository** | `continuedev/continue` (`https://github.com/continuedev/continue`) |
| **License** | `Apache-2.0` |
| **Status** | `Active` (archive notice in `README.md:19`: *"no longer actively maintained and is read-only for all users"*) |
| **Branch** | `main` |
| **Commit SHA** | `5522c6f44ca0ac3528b37244818fbfa39b5af470` |
| **Commit Date** | `2026-07-20T21:00:09-07:00` |
| **Commit Subject** | `docs: remove Sign in link (login flow retired) (#13005)` |
| **Privacy Rating** | **`D` (Aggressive / Leaky)** |

---

## 1. Default State & Opt-Out Mechanism

Continue exhibits a **split and inconsistent default state** across its components—combining stripped commercial analytics SDKs, opt-in CLI OpenTelemetry, enabled-by-default legacy IDE flags, broken UI opt-out pointers, and unconditional phone-home / proxy bypasses:

1. **CLI Version Beacon (`ON` by default, no opt-out)**:
   - On every invocation, `extensions/cli/src/index.ts:24` imports `./version.js`, which unconditionally executes `getLatestVersion()` at module import time (`extensions/cli/src/version.ts:65-75`). This sends `GET https://api.continue.dev/cn/info?id=<machineId>` (`extensions/cli/src/version.ts:40-44`) regardless of CLI flags, headless mode, `autoUpdateCli`, `CONTINUE_METRICS_ENABLED=0`, or `CONTINUE_CLI_ENABLE_TELEMETRY=0`.
   - Separate from the import-time ping, `UpdateService.doInitialize(headless)` skips `checkAndAutoUpdate()` when `headless || NODE_ENV === "test"` (`extensions/cli/src/services/UpdateService.ts:35-42`), checks `globalContext.get("autoUpdateCli") ?? true` (`extensions/cli/src/services/UpdateService.ts:46-59,148`), skips `0.0.0-dev`, and auto-runs `npm i -g @continuedev/cli` when updating (`extensions/cli/src/services/UpdateService.ts:92-99`).

2. **CLI OpenTelemetry Metrics (`OFF` by default / explicit opt-in via env vars)**:
   - In `extensions/cli/src/telemetry/telemetryService.ts:68-85`, `TelemetryService` initializes OpenTelemetry **only** when `hasOtelConfig` (`OTEL_EXPORTER_OTLP_ENDPOINT || OTEL_EXPORTER_OTLP_METRICS_ENDPOINT || OTEL_METRICS_EXPORTER`) is truthy **and** `telemetryEnabled` is true:
     - `CONTINUE_METRICS_ENABLED="0"` disables; `"1"` enables.
     - Otherwise `CONTINUE_CLI_ENABLE_TELEMETRY !== "0"` enables when an OTEL env var is set (`extensions/cli/src/telemetry/telemetryService.ts:74-84`).
   - `DO_NOT_TRACK` is **not** honored anywhere in the repository (`rg` negative match).

3. **IDE / Core `allowAnonymousTelemetry` (`ON` by default in config, UI toggle removed, controls only residual behavior)**:
   - Core defaults `config.allowAnonymousTelemetry = true` when omitted in both JSON (`core/config/load.ts:138-139`) and YAML (`core/config/yaml/loadYaml.ts:437-438`), and `extensions/vscode/config_schema.json:2737-2743` specifies `"default": true`.
   - In VS Code, `core/config/profile/doLoadConfig.ts:343-350` overrides `newConfig.allowAnonymousTelemetry = false` if `(await ide.isTelemetryEnabled()) === false`, where `VsCodeIde.isTelemetryEnabled()` returns `vscode.env.isTelemetryEnabled && (config.get("telemetryEnabled") ?? true)` (`extensions/vscode/src/VsCodeIde.ts:244-251`).
   - **Broken UI Opt-Out Pointer**: `extensions/vscode/package.json:89-95` deprecates `continue.telemetryEnabled` with the message: *"Please use the _Telemetry > Allow Anonymous Telemetry_ setting in the extension's User Settings page."* However, `gui/src/pages/config/sections/UserSettingsSection.tsx:66,86,88-294` defines `allowAnonymousTelemetry = config.allowAnonymousTelemetry ?? true` and `disableTelemetryToggle = false` as dead variables and **never renders any telemetry toggle in JSX**.
   - **Opt-Out Bypasses**:
     - `core/config/profile/doLoadConfig.ts:84,353` unconditionally calls `ide.getUniqueId()` and `IdeInfoService.setup(uniqueId, ideInfo)` regardless of `allowAnonymousTelemetry` or `isTelemetryEnabled()`.
     - `core/context/providers/WebContextProvider.ts:9,16-36` (`@web`) and `core/indexing/docs/crawlers/DefaultCrawler.ts:4,15-26` (`core/indexing/docs/crawlers/DocsCrawler.ts:51-65`) attach `getHeaders()`—including `uniqueId`—to `https://proxy-server-blue-l6vsfbzhba-uw.a.run.app` even when telemetry is disabled.
     - `DataLogger.logDevData` (`core/data/log.ts:104-117`) and `DevDataSqliteDb.logTokensGenerated` (`core/llm/index.ts:353-358`, `core/data/devdataSqlite.ts:40-51`) always log local dev data (`~/.continue/dev_data/0.2.0/<event>.jsonl` and `~/.continue/dev_data/devdata.sqlite`) regardless of `allowAnonymousTelemetry`.
     - `CONTINUE_ALLOW_ANONYMOUS_TELEMETRY` appears only in `extensions/cli/vitest.setup.ts:7` and is never read in production code.

---

## 2. External Auth / Local Non-Workspace File Reads

1. **Claude Code Config & Skills Compatibility Reads (Functional, Non-Telemetry)**:
   - `extensions/cli/src/hooks/hookConfig.ts:73-91,101-123` reads `~/.claude/settings.json`, `<cwd>/.claude/settings.json`, and `<cwd>/.claude/settings.local.json` to load `hooks` and `disableAllHooks` for Claude Code hook compatibility.
   - `extensions/cli/src/util/loadMarkdownSkills.ts:95-99` and `core/config/markdown/loadMarkdownSkills.ts:23-28` read `<cwd>/.claude/skills` alongside `.continue/skills` for markdown skill discovery.
   - **No credential/auth files** from Claude (`~/.claude/.credentials.json`), Codex (`~/.codex/auth.json`), Gemini (`~/.gemini/`), or SSH (`~/.ssh/`) are read for tracking, identity stitching, or telemetry.

2. **Cloud Provider Credential Chains (Only When Configured as Active LLM)**:
   - AWS `fromNodeProviderChain` (which searches `~/.aws/credentials`, env vars, and IMDS) is invoked strictly inside the AWS-provider LLM adapters: `core/llm/llms/Bedrock.ts:603-612`, `core/llm/llms/BedrockImport.ts:86-94`, `core/llm/llms/SageMaker.ts:133-140`, and `packages/openai-adapters/src/apis/Bedrock.ts:88-97`.

3. **OS CA Trust Store & Unused Continue Auth Path**:
   - `core/util/ca.ts:6-27` reads the OS certificate store via `mac-ca`, `win-ca`, and `system-ca` for TLS verification.
   - `extensions/cli/src/auth/authEnv.ts:5` defines `AUTH_CONFIG_PATH = path.join(env.continueHome, "auth.json")`, which is unused in the retired WorkOS auth flow (`extensions/cli/src/auth/workos.ts`).

4. **Non-Workspace Continue State (`~/.continue`)**:
   - `DataLogger.logLocalData` (`core/data/log.ts:104-117`) and `DevDataSqliteDb` (`core/data/devdataSqlite.ts:40-51`) write dev data and token metrics to `~/.continue/dev_data/`.
   - When `OTEL_LOG_USER_PROMPTS === "1"` and verbose/debug logging is active, CLI writes debug logs (including prompt text) to `~/.continue/logs/cn.log` (`extensions/cli/src/telemetry/telemetryService.ts:489-571`, `extensions/cli/src/util/logger.ts:28-30`).

---

## 3. Persistent Device / Install IDs & Hashing

| Identifier | Source / Generation | Hashing / Salting | Where Transmitted or Stored |
|---|---|---|---|
| **CLI Startup Beacon ID** (`id`) | Raw `node_machine_id.machineIdSync()` (`extensions/cli/src/version.ts:22-24`) | **None (unhashed raw library output)** | Transmitted in query string on every CLI run: `GET https://api.continue.dev/cn/info?id=<id>` (`extensions/cli/src/version.ts:40-44`). |
| **CLI `getUniqueId()`** | `process.env.CONTINUE_USER_ID \|\| node_machine_id.machineIdSync()` (`extensions/cli/src/util/uniqueId.ts:9-18`) | **None (raw unless env-overridden)** | Referenced in unused `_mergeUserAgentIntoRequestOptions` (`extensions/cli/src/config.ts:28-39`, header `x-continue-unique-id`). |
| **VS Code `uniqueId`** | `vscode.env.machineId` (`extensions/vscode/src/VsCodeIde.ts:257-259`) | VS Code-managed hash; fallback unhashed | Passed unconditionally to `IdeInfoService.setup(uniqueId, ideInfo)` (`core/config/profile/doLoadConfig.ts:84,353`), sent via `getHeaders()` to Cloud Run proxy (`core/context/providers/WebContextProvider.ts:9,16-36`, `core/indexing/docs/crawlers/DefaultCrawler.ts:4,15-26`), and logged to local `dev_data` (`core/autocomplete/CompletionProvider.ts:283`, `core/nextEdit/providers/BaseNextEditProvider.ts:360`). |
| **VS Code Webview Fallback `vscMachineId`** | `getUniqueId()` injects `window.vscMachineId`; if `vscode.env.machineId === "someValue.machineId"`, falls back to `node-machine-id` `machineIdSync()` (`extensions/vscode/src/ContinueGUIWebviewViewProvider.ts:161`, `extensions/vscode/src/util/vscode.ts:109-115`) | **None on fallback** | Exposed into GUI webview `window.vscMachineId`. |
| **IntelliJ `uniqueId`** | First non-null `NetworkInterface.getNetworkInterfaces()` hardware MAC address formatted `%02X-...` (`extensions/intellij/src/main/kotlin/com/github/continuedev/continueintellijextension/utils/Utils.kt:54-77`, `extensions/intellij/src/main/kotlin/com/github/continuedev/continueintellijextension/continue/IntelliJIde.kt:197-199`, `extensions/intellij/src/main/kotlin/com/github/continuedev/continueintellijextension/continue/IdeProtocolClient.kt:81,115-116`), or `"No MAC Address Found"` | **None (raw uppercase MAC address, unhashed, unsalted)** | Populated unconditionally in `IdeInfoService.setup(uniqueId, ideInfo)` (`core/config/profile/doLoadConfig.ts:84,353`), transmitted in `uniqueId` header to Cloud Run proxy (`core/continueServer/stubs/headers.ts:4-13`), and written to local `dev_data` (`core/autocomplete/CompletionProvider.ts:283`, `core/nextEdit/providers/BaseNextEditProvider.ts:360`). |
| **CLI OTEL Resource Host/Account IDs** | `os.hostname()` (`extensions/cli/src/telemetry/telemetryService.ts:105`); optional `ACCOUNT_UUID` / `ORGANIZATION_ID` (`extensions/cli/src/telemetry/telemetryService.ts:315-326`) | **None (plaintext)** | Emitted only when user explicitly enables OTEL via `OTEL_*` env vars. |
| **Core `LLM.uniqueId`** | Passed to `BaseLLM` (`core/llm/index.ts:225`) | N/A | **Dead field** — assigned and never read anywhere in `core/`. |

---

## 4. Exact Outbound Telemetry / Analytics / Crash / OTEL / Relay / Update Endpoints

| Endpoint / URL | Trigger & Gate | Payload / Headers Sent | Source Reference |
|---|---|---|---|
| `https://api.continue.dev/cn/info?id=<machineId>` | **Unconditional** on every CLI module load (`import { getVersion } from "./version.js"`) | `id` = raw `node_machine_id.machineIdSync()` | `extensions/cli/src/index.ts:24`, `extensions/cli/src/version.ts:22-24,38-44,65-75` |
| `https://proxy-server-blue-l6vsfbzhba-uw.a.run.app/web` | User invokes `@web` context provider (**bypasses** `allowAnonymousTelemetry: false`) | Web search query + `getHeaders()`: `key` (`"NfZFVegMpdyT3P5UmAggr7T7Hb6PlcbB"`), `timestamp`, `v: "1"`, `extensionVersion`, `os`, `uniqueId` | `core/context/providers/WebContextProvider.ts:9,16-36`, `core/continueServer/stubs/headers.ts:4-13`, `core/deploy/constants.ts:1-11` |
| `https://proxy-server-blue-l6vsfbzhba-uw.a.run.app/crawl` | Docs indexing when `useLocalCrawling` is false and host != `github.com` (**bypasses** `allowAnonymousTelemetry: false`) | Target docs URL + `getHeaders()` (same static `key`, `extensionVersion`, `os`, `uniqueId`) | `core/indexing/docs/crawlers/DefaultCrawler.ts:4,15-26`, `core/indexing/docs/crawlers/DocsCrawler.ts:51-65`, `core/continueServer/stubs/headers.ts:4-13` |
| `https://api.continue.dev/` (remote default config: `getAssistant`) | CLI launched without `--config` and without local `~/.continue/config.yaml` (`remote-default-config`) | Fetches `continuedev/default-cli-config` (`CONTINUE_API_BASE` overrides base URL) | `extensions/cli/src/env.ts:9`, `extensions/cli/src/configLoader.ts:94-100,326-355` |
| `https://api.continue.dev/` (`syncSecrets`) | CLI config references `${{ secrets.* }}` absent from `process.env` and `.env` | Requests secret lookup via `CLIPlatformClient.apiClient.syncSecrets` | `extensions/cli/src/CLIPlatformClient.ts:113-136` |
| `https://api.continue.dev/` (`reportUnhandledErrorToApi`, `postAgentMetadata`, `reportFailure`) | Only when CLI runs `cn serve --id <agentId>` (remote/CI agent mode) | Error message/stack or agent metadata (`--id` gated) | `extensions/cli/src/index.ts:99-117`, `extensions/cli/src/util/metadata.ts:125-182`, `extensions/cli/src/tools/reportFailure.ts:60` |
| `https://api.github.com/repos/.../pulls` & `https://api.continue.dev/api/checks/status` / `agents/...` | Explicit user invocation of `cn checks` subcommand | PR metadata / check status lookup | `extensions/cli/src/commands/checks.ts:65-66,128-149` |
| `npm i -g @continuedev/cli` (npm registry) | CLI interactive mode when `autoUpdateCli ?? true` and not `0.0.0-dev` (skipped in `--headless` and `test`) | Standard npm CLI package upgrade | `extensions/cli/src/services/UpdateService.ts:35-59,92-99,148` |
| User-configured OTEL (`OTEL_EXPORTER_OTLP_METRICS_ENDPOINT`, `${OTEL_EXPORTER_OTLP_ENDPOINT}/v1/metrics`, or `http://localhost:4318/v1/metrics`) | **Opt-in only**: requires `OTEL_*` env var set and `CONTINUE_METRICS_ENABLED !== "0"` / `CONTINUE_CLI_ENABLE_TELEMETRY !== "0"` | OpenTelemetry metrics (`console` or `otlp` exporter, 20s default interval) | `extensions/cli/src/telemetry/telemetryService.ts:68-85,112-155` |
| User-configured `config.data` HTTP/file destinations | Only if explicitly configured in `config.yaml` `data:` array | DevData schemas (`autocomplete`, `nextEditOutcome`, etc.) | `core/data/log.ts:162-237` |
| *Retired / Blocked Endpoints*: `StorageSyncService`, `uploadArtifact` | Hardcoded `accessToken: null` blocks execution | N/A (dead path) | `extensions/cli/src/services/StorageSyncService.ts:85`, `extensions/cli/src/commands/serve.ts:188`, `extensions/cli/src/tools/uploadArtifact.ts:64-70` |
| *Docs Website Trackers (Not In Agent)*: `https://static.reo.dev/7aa28ed11570734/reo.js`, `window.gtag` / `window.posthog` | Only when visiting Mintlify documentation site | Docs site analytics (`Reo.init`, cookie banner consent toggles) | `docs/reo-tracking.js:1-17`, `docs/c15t-cookie-banner.js:72-90` |

---

## 5. Exact Data & Properties Collected

1. **IP Address & GeoIP Handling**:
   - No client-side IP or GeoIP lookup SDK/endpoint exists in the codebase (`rg` negative match for `ipify|geoip|ip-api|checkip|ipinfo|x-forwarded-for`).
   - However, CLI sends a direct HTTPS request to `https://api.continue.dev/cn/info?id=<machineId>` (`extensions/cli/src/version.ts:40-44`), and `@web` / default docs crawling send direct HTTPS POSTs to Continue's closed-source Cloud Run proxy `https://proxy-server-blue-l6vsfbzhba-uw.a.run.app` (`core/context/providers/WebContextProvider.ts:9,16-36`, `core/indexing/docs/crawlers/DefaultCrawler.ts:4,15-26`)—exposing the client's source IP address to Continue-operated servers with **no client-side IP suppression or anonymization header**.
   - In CLI OTEL (when opted in), resource attributes include plaintext `host.name: os.hostname()`, `os.type: os.type()`, and `service.version` (`extensions/cli/src/telemetry/telemetryService.ts:101-108`).

2. **Custom / Private Model Names**:
   - **CLI OTEL (when enabled)**: `recordCost` (`extensions/cli/src/telemetry/telemetryService.ts:382`), `recordTokenUsage` (`extensions/cli/src/telemetry/telemetryService.ts:392`), and `recordResponseTime` (`extensions/cli/src/telemetry/telemetryService.ts:480`) pass the **verbatim `model` string** as an OTEL metric attribute with no scrubbing, hashing, or private-model filtering.
   - **Local DevData / SQLite**: `DevDataSqliteDb.logTokensGenerated` (`core/llm/index.ts:353-358`, `core/data/devdataSqlite.ts:40-51`) records provider and model names locally. If a user configures a `data:` HTTP destination, `packages/config-yaml/src/schemas/data/autocomplete/v0.2.0.ts:47-53` and `packages/config-yaml/src/schemas/data/nextEditOutcome/v0.2.0.ts:35-44` retain `modelProvider` and `modelName` even at the `"noCode"` privacy level.

3. **Prompts, Code Snippets, File Paths, & Stack Traces**:
   - **CLI OTEL**: `logUserPrompt`, `logToolResult`, and `logApiRequest` (`extensions/cli/src/telemetry/telemetryService.ts:489-571`) contain `// TODO: Implement OTLP logs export` and do **not** transmit prompts/tool outputs over OTLP. If `OTEL_LOG_USER_PROMPTS === "1"` and debug/verbose logging is enabled, raw prompts are written locally to `~/.continue/logs/cn.log` (`extensions/cli/src/telemetry/telemetryService.ts:489-571`, `extensions/cli/src/util/logger.ts:28-30`).
   - **Local DevData (`~/.continue/dev_data/0.2.0/*.jsonl`)**: Unconditionally logs `autocomplete` and `nextEditOutcome` events locally (`core/data/log.ts:104-117`, `core/autocomplete/CompletionProvider.ts:283`, `core/nextEdit/providers/BaseNextEditProvider.ts:360`). Even when exported to a user-configured `data:` HTTP endpoint with `level: "noCode"`, `packages/config-yaml/src/schemas/data/autocomplete/v0.2.0.ts:47-53` still retains `filepath`, `gitRepo`, `uniqueId`, `userId`, `userAgent`, `modelProvider`, and `modelName` (and `packages/config-yaml/src/schemas/data/nextEditOutcome/v0.2.0.ts:35-44` retains `gitRepo`, `uniqueId`, `userId`, `userAgent`, `modelProvider`, and `modelName`).
   - **Error / Stack Traces**: In CLI `cn serve --id <agentId>` mode only, `reportUnhandledErrorToApi` (`extensions/cli/src/index.ts:99-117`) and `reportFailure` (`extensions/cli/src/tools/reportFailure.ts:60`) can send failure messages/stacks to `api.continue.dev`. Local indexing errors (`core/indexing/CodebaseIndexer.ts:681-687`) and IntelliJ binary stderr errors (`extensions/intellij/src/main/kotlin/com/github/continuedev/continueintellijextension/continue/process/ContinueBinaryProcess.kt:41-52`) are local no-ops / `console.debug` only.

4. **Behavioral Profile Fields**:
   - **CLI OTEL (when opted in)**: `getStandardAttributes` (`extensions/cli/src/telemetry/telemetryService.ts:307-331`) attaches `terminal.type`, optional `app.version` (`OTEL_METRICS_INCLUDE_VERSION === "true"`), `organization.id` (`ORGANIZATION_ID`), and `user.account_uuid` (`ACCOUNT_UUID` when `OTEL_METRICS_INCLUDE_ACCOUNT_UUID !== "false"`). Metric attributes (`extensions/cli/src/telemetry/telemetryService.ts:335-486,585-592`) record `language`, `command`, `is_headless`, `is_github_actions`, and `is_continue_remote_agent`. (`codeEditDecisionCounter` and `authAttemptsCounter` have zero callers, and `mcpConnectionsGauge` has no callback.)
   - **Cloud Run Proxy Requests**: Send `extensionVersion`, `os`, `timestamp`, `v: "1"`, hardcoded `key`, and `uniqueId` (`core/continueServer/stubs/headers.ts:4-13`).

---

## 6. Docs / Privacy Policy vs Code Discrepancies

1. **README "Anonymous Telemetry Removed" Overstatement**:
   - `README.md:27-31` states under `## Final 2.0.0 Release`: *"This included removing anonymous telemetry, pulling out authentication, squashing bugs, and more."*
   - **Code Reality**: While PostHog, Sentry, and login flows were stripped (`TESTING.md:16`), CLI still phones home unconditionally to `https://api.continue.dev/cn/info?id=<raw machineId>` on every launch (`extensions/cli/src/version.ts:22-24,40-44,65-75`), and IDE extensions still populate `uniqueId` unconditionally (`core/config/profile/doLoadConfig.ts:84,353`)—using a raw unhashed MAC address in IntelliJ (`extensions/intellij/src/main/kotlin/com/github/continuedev/continueintellijextension/utils/Utils.kt:54-77`)—and transmit it to `https://proxy-server-blue-l6vsfbzhba-uw.a.run.app` during `@web` and docs crawling (`core/context/providers/WebContextProvider.ts:9,16-36`, `core/indexing/docs/crawlers/DefaultCrawler.ts:4,15-26`).

2. **Deleted Telemetry Documentation & Broken Redirects**:
   - `docs/docs.json:844-849` defines redirects from `/telemetry` and `/advanced/telemetry` to `/customize/telemetry`, which no longer exists in `docs/`.
   - `docs/guides/running-continue-without-internet.mdx:7` still instructs users: *"Turn off 'Allow Anonymous Telemetry' in the user settings. This will stop Continue from attempting requests to PostHog..."*, even though PostHog is removed and the UI toggle is absent.

3. **VS Code Settings Deprecation Pointer to a Non-Existent UI Toggle**:
   - `extensions/vscode/package.json:89-95` tells users `continue.telemetryEnabled` is deprecated and directs them: *"Please use the _Telemetry > Allow Anonymous Telemetry_ setting in the extension's User Settings page."*
   - `gui/src/pages/config/sections/UserSettingsSection.tsx:66,86,88-294` no longer renders any telemetry toggle at all.

4. **CLI OTEL Spec (`session.id` & Logs) vs Implementation**:
   - `extensions/cli/spec/otlp-metrics.md:37,47` documents `OTEL_METRICS_INCLUDE_SESSION_ID`, but `extensions/cli/src/telemetry/telemetryService.ts:307-331` never reads it or emits `session.id`. Conversely, OTLP log export is unimplemented (`// TODO: Implement OTLP logs export` at `extensions/cli/src/telemetry/telemetryService.ts:489-571`).

---

## 7. Zero-Telemetry / Negative-Check Verification

All claims of removed or absent telemetry channels were verified via targeted `rg` searches across code directories (excluding `node_modules`, `vendor`, `test-examples`, `CHANGELOG.md`, `package-lock.json`, and `docs/`):

1. **PostHog & Sentry**: `0` runtime SDK imports, initializations, or capture calls remain in `core/`, `extensions/`, `gui/`, `packages/`, or `binary/`. Matches were limited to `TESTING.md:16` (removal checklist), comments in `.github/actions/track-rerun/action.yml:7` and `core/tsconfig.json:27`, test/fixture strings (`sentry-mcp`, `continuedev/sentry-nextjs`), and stale lockfile entries in `extensions/cli/package-lock.json:39,90,4691,16143`. (Contrast with `docs/reo-tracking.js:1-17` and `docs/c15t-cookie-banner.js:72-90`, which only live in the Mintlify docs site.)
2. **LaunchDarkly**: Present only as an unused dependency in `core/package.json:93` (`"launchdarkly-node-client-sdk": "^3.2.0"`); `0` imports or calls across the repository.
3. **Amplitude / Mixpanel / Segment / Bugsnag / Datadog / Rollbar / Honeycomb**: `0` telemetry SDK imports or outbound usages (only an unused `analyticsSchema` definition in `packages/config-types/src/index.ts:194-206` and `extensions/vscode/config_schema.json:3321-3336`).
4. **IP / GeoIP Lookup APIs**: `0` matches for `ipify|geoip|ip-api|checkip|ipinfo|x-forwarded-for`.
5. **`DO_NOT_TRACK`**: `0` matches across the entire repository.
6. **Gutted Telemetry Stubs Verified**:
   - `core/indexing/CodebaseIndexer.ts:681-687`: `sendIndexingErrorTelemetry` only logs via `console.debug`.
   - `extensions/intellij/src/main/kotlin/com/github/continuedev/continueintellijextension/continue/process/ContinueBinaryProcess.kt:41-52`: `reportErrorTelemetry` reads stderr and discards it without network calls.
   - `core/llm/index.ts:225`: `this.uniqueId` is stored on `BaseLLM` and never read in `core/`.
   - `extensions/cli/src/services/StorageSyncService.ts:85` and `extensions/cli/src/tools/uploadArtifact.ts:64-70`: gated out by hardcoded `accessToken: null`.

---

```json
{
  "slug": "continue",
  "platform": "Continue",
  "group": "B",
  "repo": "continuedev/continue",
  "sha": "5522c6f44ca0ac3528b37244818fbfa39b5af470",
  "commit_date": "2026-07-20T21:00:09-07:00",
  "license": "Apache-2.0",
  "status": "Active",
  "default_state": "Mixed: IDE telemetry SDKs (PostHog/Sentry) removed and CLI OTEL metrics are opt-in (off unless OTEL_EXPORTER_OTLP_* or OTEL_METRICS_EXPORTER is set), but CLI unconditionally sends an unhashed machineId beacon to api.continue.dev on every import/launch, allowAnonymousTelemetry defaults to true in Core/VS Code schemas, and local dev_data logs unconditionally.",
  "opt_out_controls": [
    "VS Code telemetry: continue.telemetryEnabled (default true) AND vscode.env.isTelemetryEnabled",
    "Core config: allowAnonymousTelemetry in config.json / config.yaml (default true)",
    "CLI OTEL metrics: CONTINUE_METRICS_ENABLED=0 or CONTINUE_CLI_ENABLE_TELEMETRY=0 (when OTEL_* env vars are set)",
    "CLI auto-update: autoUpdateCli in globalContext (no effect on import-time api.continue.dev/cn/info ping)",
    "Broken UI control: gui/src/pages/config/sections/UserSettingsSection.tsx renders no telemetry toggle despite VS Code setting deprecation message"
  ],
  "do_not_track": "Not respected (0 matches for DO_NOT_TRACK in repository)",
  "opt_out_bypasses": [
    "extensions/cli/src/version.ts:65 unconditionally calls getLatestVersion() on import of version.js, sending raw node_machine_id.machineIdSync() to api.continue.dev/cn/info even in --headless mode or with telemetry/autoUpdateCli disabled",
    "core/config/profile/doLoadConfig.ts:84,353 sets IdeInfoService.uniqueId unconditionally; core/context/providers/WebContextProvider.ts (@web) and core/indexing/docs/crawlers/DefaultCrawler.ts (docs crawling) send uniqueId in headers to proxy-server-blue-l6vsfbzhba-uw.a.run.app even when allowAnonymousTelemetry is false",
    "core/data/log.ts (DataLogger.logDevData) and core/data/devdataSqlite.ts (DevDataSqliteDb) always write ~/.continue/dev_data/ regardless of allowAnonymousTelemetry"
  ],
  "persistent_ids": [
    "CLI version check id & getUniqueId(): node_machine_id.machineIdSync() (overridable only for getUniqueId via CONTINUE_USER_ID)",
    "VS Code uniqueId: vscode.env.machineId (webview fallback uses raw node_machine_id.machineIdSync() when machineId is 'someValue.machineId')",
    "IntelliJ uniqueId: first non-null NetworkInterface hardware MAC address formatted %02X-...",
    "CLI OTEL (opt-in): os.hostname(), optional ACCOUNT_UUID and ORGANIZATION_ID"
  ],
  "hashing": "None for CLI node_machine_id.machineIdSync(), VS Code webview fallback machineIdSync(), and IntelliJ MAC address (raw uppercase %02X-...); only VS Code's built-in vscode.env.machineId is VS Code-hashed.",
  "external_auth_reads": "Reads ~/.claude/settings.json and <cwd>/.claude/settings(.local).json for Claude Code hook compatibility (extensions/cli/src/hooks/hookConfig.ts), <cwd>/.claude/skills for skills (extensions/cli/src/util/loadMarkdownSkills.ts), OS CA store (core/util/ca.ts), and AWS fromNodeProviderChain (~/.aws/credentials) only when Bedrock/SageMaker is configured; zero reads of other tools' credentials for tracking.",
  "telemetry_endpoints": [
    "https://api.continue.dev/cn/info?id=<machineId> (unconditional CLI startup version ping)",
    "https://proxy-server-blue-l6vsfbzhba-uw.a.run.app/web (Cloud Run proxy for @web)",
    "https://proxy-server-blue-l6vsfbzhba-uw.a.run.app/crawl (Cloud Run proxy for docs crawling)",
    "https://api.continue.dev/ (CLI remote default config, syncSecrets fallback, cn checks, and --id remote agent errors/metadata)",
    "User-configured OTEL_EXPORTER_OTLP_METRICS_ENDPOINT / OTEL_EXPORTER_OTLP_ENDPOINT/v1/metrics / localhost:4318/v1/metrics (opt-in CLI OTEL only)",
    "User-configured config.data HTTP destinations (opt-in only)"
  ],
  "ip_geo_handling": "No client-side IP/GeoIP API calls, but direct HTTPS calls to api.continue.dev/cn/info and closed Cloud Run proxy proxy-server-blue-l6vsfbzhba-uw.a.run.app expose client IP without client-side suppression; CLI OTEL exports plaintext os.hostname().",
  "custom_model_handling": "CLI OTEL (when opted in) exports verbatim model names on cost, token usage, and response time metrics without hashing or filtering; local DevData/SQLite and opt-in user data: noCode payloads retain modelProvider and modelName.",
  "docs_discrepancies": [
    "README.md:31 claims anonymous telemetry was removed in 2.0.0 despite unconditional CLI machineId ping to api.continue.dev/cn/info and unconditional uniqueId headers to Cloud Run proxy",
    "extensions/vscode/package.json:89-95 points users to a 'Telemetry > Allow Anonymous Telemetry' UI toggle removed from gui/src/pages/config/sections/UserSettingsSection.tsx",
    "docs/docs.json:844-849 redirects /telemetry to deleted /customize/telemetry; docs/guides/running-continue-without-internet.mdx:7 still references turning off telemetry to stop PostHog",
    "extensions/cli/spec/otlp-metrics.md documents OTEL_METRICS_INCLUDE_SESSION_ID which extensions/cli/src/telemetry/telemetryService.ts never emits"
  ],
  "privacy_rating": "D",
  "one_line_verdict": "PostHog and Sentry were stripped for 2.0.0 and CLI OTEL is strictly opt-in, but CLI unconditionally phones home to api.continue.dev with a raw machineId on every launch, IntelliJ/VS Code leak persistent MAC/machineId headers to Continue's Cloud Run proxy even when telemetry is disabled, DO_NOT_TRACK is ignored, and the UI opt-out toggle referenced by VS Code settings and docs was deleted."
}
```
