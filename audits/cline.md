# Privacy & Telemetry Forensic Audit: Cline (`cline/cline`)

## Header Metadata

- **Slug:** `cline`
- **Platform:** `Cline`
- **Group:** `B`
- **Repository:** `cline/cline` (`https://github.com/cline/cline`)
- **License:** `Apache-2.0`
- **Status:** `Active`
- **Branch:** `main`
- **Commit SHA:** `34c104e6fc49781cda4adcd73ffdccd8f029fef9`
- **Commit Date:** `2026-11-23T13:06:22+01:00`
- **Commit Subject:** `Move the third party notices to the root on the CLI install (#15836)`

---

## 1. Default State & Opt-Out Mechanism

### Default State

Telemetry is **enabled by default (Opt-Out)** across all shipped Cline distribution surfaces — VS Code extension (including the A/B rollout wrapper `apps/vscode-rollout`), Cline CLI + Hub daemon (`apps/cli`, `sdk/packages/core`), and Cline Desktop (`apps/examples/desktop-app`):

1. **Shared Global Settings (`sdk/packages/core/src/services/global-settings.ts:61-65,240-242`):**
   - `GlobalSettingsSchema` defines `telemetryOptOut: z.boolean().default(false).catch(false)` (`sdk/packages/core/src/services/global-settings.ts:63`) and `autoUpdateEnabled: z.boolean().default(true).catch(true)` (`sdk/packages/core/src/services/global-settings.ts:64`).
   - `isTelemetryOptedOutGlobally()` returns `readGlobalSettings().telemetryOptOut`, so telemetry starts ON unless explicitly set to `true`.
2. **VS Code Extension (`apps/vscode`):**
   - `StateManager` defaults `telemetrySetting` to `"unset"` (`apps/vscode/src/shared/storage/state-keys.ts:277`). On `Controller` startup, `syncTelemetrySettingFromSharedGlobalSettings` maps `telemetryOptOut: false` to `"enabled"` when no remote-config override exists (`apps/vscode/src/sdk/telemetry-settings-sync.ts:13-15,45-51`, `apps/vscode/src/sdk/SdkController.ts:322`).
   - The General Settings checkbox renders `checked={telemetrySetting !== "disabled"}` (`apps/vscode/webview-ui/src/components/settings/sections/GeneralSettingsSection.tsx:28-36`), treating both `"unset"` and `"enabled"` as opted in.
   - Official release workflows bake PostHog and OpenTelemetry (OTLP logs, metrics, traces, and `CLINE_TRACE_RECORD_CONTENT: "true"`) into production, beta, nightly, and A/B-rollout VSIX builds via build-time defines (`.github/workflows/ext-vscode-publish.yml:213-224`, `.github/workflows/ext-vscode-publish-stable.yml:268-279`, `.github/workflows/ext-vscode-ab-package.yml:276-291`, `apps/vscode/esbuild.mjs:153`, `apps/vscode/webview-ui/vite.config.ts:145-156`, `apps/vscode/src/shared/services/config/posthog-config.ts:40-46`, `apps/vscode/src/shared/services/config/otel-config.ts:131-145`).
   - In the classic `TelemetryProviderFactory.getDefaultConfigs()`, build-time OTLP is active by default unless `ClineEndpoint.isSelfHosted()` is true (`apps/vscode/src/services/telemetry/TelemetryProviderFactory.ts:110-130`); error tracking defaults to `"no-op"` (`apps/vscode/src/services/error/ErrorProviderFactory.ts:54-59`).
   - `SdkController` instantiates `createVscodeSdkTelemetryHandle()` (`apps/vscode/src/sdk/SdkController.ts:323`), which wraps `@cline/core`'s `createConfiguredTelemetryHandle(...)` in `VscodeTelemetryPolicyService` (`apps/vscode/src/sdk/sdk-telemetry.ts:32-66,68-77`).
3. **Cline CLI, Hub Daemon & Desktop App:**
   - Both CLI release binaries (`.github/workflows/cli-publish.yml:141-166`, `apps/cli/bun.mts:88-120`) and the Desktop Tauri sidecar (`.github/workflows/desktop-publish.yml:309-355`, `apps/examples/desktop-app/scripts/telemetry-define-args.ts:13-66`) inline `TELEMETRY_SERVICE_API_KEY`, `ERROR_SERVICE_API_KEY`, `OTEL_TELEMETRY_ENABLED`, `OTEL_LOGS_EXPORTER=otlp`, `OTEL_METRICS_EXPORTER=otlp`, `OTEL_TRACES_EXPORTER=otlp`, `CLINE_TRACE_RECORD_CONTENT="true"`, `OTEL_EXPORTER_OTLP_PROTOCOL`, `OTEL_EXPORTER_OTLP_ENDPOINT`, and `OTEL_EXPORTER_OTLP_HEADERS`.
   - Desktop initializes `createDesktopObservability()` at sidecar startup (`apps/examples/desktop-app/sidecar/index.ts:71`, `apps/examples/desktop-app/sidecar/observability.ts:28-55`), and its webview displays the **Telemetry** switch checked when `!telemetryOptOut` (`apps/examples/desktop-app/webview/components/views/settings/settings-view.tsx:1024-1029`).
   - **Self-Hosted Exception:** In VS Code, creating `~/.cline/endpoints.json` or bundling `endpoints.json` sets `Environment.selfHosted` (`apps/vscode/src/config.ts:85-92,130,296-305`), disabling `PostHogClientProvider` (`apps/vscode/src/services/telemetry/providers/posthog/PostHogClientProvider.ts:25-28`), `CustomPostHogProvider` (`apps/vscode/webview-ui/src/CustomPostHogProvider.tsx:11,17`), and build-time OTEL (`apps/vscode/src/services/telemetry/TelemetryProviderFactory.ts:113`), though **not** `apps/vscode-rollout`.

### Opt-Out Controls

- **VS Code UI:** Settings -> General -> uncheck **Allow error and usage reporting** (`apps/vscode/webview-ui/src/components/settings/sections/GeneralSettingsSection.tsx:28-36`), invoking `updateTelemetrySetting` (`apps/vscode/src/core/controller/state/updateTelemetrySetting.ts:12-15`) which writes `telemetryOptOut: true` to shared `~/.cline/data/settings/global-settings.json` and sets `telemetrySetting: "disabled"` in `StateManager` (`apps/vscode/src/sdk/SdkController.ts:2225-2230`).
- **VS Code Host Setting:** VS Code `telemetry.telemetryLevel` (`vscode.env.isTelemetryEnabled` via `HostProvider.env.getTelemetrySettings`) is checked for ordinary classic PostHog/OTEL events and ordinary `VscodeTelemetryPolicyService` events (`apps/vscode/src/services/telemetry/providers/posthog/PostHogTelemetryProvider.ts:61-64`, `apps/vscode/src/services/telemetry/providers/opentelemetry/OpenTelemetryTelemetryProvider.ts:197-202`, `apps/vscode/src/sdk/sdk-telemetry.ts:234-240`).
- **Desktop UI:** Settings -> General -> **Telemetry** switch invokes sidecar command `set_telemetry_opt_out` (`apps/examples/desktop-app/webview/components/views/settings/settings-view.tsx:768-771,1024-1029`, `apps/examples/desktop-app/sidecar/commands.ts:3340-3346`), writing `telemetryOptOut` via `setTelemetryOptOutGlobally(...)`.
- **CLI (No Interactive/Command Opt-Out):** `apps/cli/src/commands/` and `apps/cli/src/tui/` expose **zero** CLI flags, subcommands, or TUI config toggles for `telemetryOptOut` (only `autoUpdateEnabled` appears in `apps/cli/src/tui/views/config-view.tsx:638-640,853-854`). Users must manually edit `~/.cline/data/settings/global-settings.json` to set `"telemetryOptOut": true` (`docs/enterprise-solutions/monitoring/telemetry.mdx:46-54`).
- **Standard Environment Variables Not Honored:**
  - `DO_NOT_TRACK`: **Not supported** (`0` matches in repository).
  - `CLINE_TELEMETRY_DISABLED`: **Not read by any production code** (`0` production matches; ironically, `apps/cli/src/tests/helpers/env.ts:93`, `apps/cli/src/cli.e2e.test.ts:223`, and `apps/cli/src/cli.tuistory.e2e.test.ts:55` set `CLINE_TELEMETRY_DISABLED: "1"`, which has no effect).
  - `OTEL_SDK_DISABLED`: **Not supported** (`0` matches in `sdk/` or `apps/`).
- **CLI Auto-Update Opt-Out (Separate):** `CLINE_NO_AUTO_UPDATE=1` or `"autoUpdateEnabled": false` in `global-settings.json` / CLI TUI disables CLI npm update checks (`apps/cli/src/commands/update.ts:311-314`, `apps/cli/src/tui/views/config-view.tsx:638-640`).

### Opt-Out Bypasses & Gaps (Verified in Code)

1. **`captureRequired` Bypasses User Opt-Out:**
   - Classic `TelemetryService.captureRequired` (`apps/vscode/src/services/telemetry/TelemetryService.ts:476-483`) routes to `provider.logRequired(...)`. Both `OpenTelemetryTelemetryProvider.logRequired` (`apps/vscode/src/services/telemetry/providers/opentelemetry/OpenTelemetryTelemetryProvider.ts:129-143`, comment: `"Required events always go through regardless of settings"`) and `PostHogTelemetryProvider.logRequired` (`apps/vscode/src/services/telemetry/providers/posthog/PostHogTelemetryProvider.ts:93-102`) emit unconditionally without checking `telemetrySetting` or `hostEnabled`.
   - SDK `VscodeTelemetryPolicyService.captureRequired` checks only `this.hostTelemetryEnabled`, **ignoring Cline's own `telemetrySetting === "disabled"`** (`apps/vscode/src/sdk/sdk-telemetry.ts:110-115,242-248`). When `telemetryOptOut` flips from `false` to `true`, `writeGlobalSettings` calls `captureTelemetryOptOut(options.telemetry)` which invokes `captureRequired("user.opt_out", ...)` (`sdk/packages/core/src/services/global-settings.ts:233-235`, `sdk/packages/core/src/services/telemetry/core-events.ts:418-426`). Sandboxed plugins with `record.required === true` also emit via `captureRequired` (`sdk/packages/core/src/runtime/host/local/agent-event-bridge.ts:283-287,303-328`).
2. **VS Code Host Telemetry Off (`vscode.env.isTelemetryEnabled === false`) Fails to Block Startup Event & Prompt Traces:**
   - `createVscodeSdkTelemetryHandle()` calls `createConfiguredTelemetryHandle(...)` BEFORE checking host telemetry state (`apps/vscode/src/sdk/sdk-telemetry.ts:32-60`). Inside `OpenTelemetryProvider`'s constructor (`sdk/packages/core/src/services/telemetry/OpenTelemetryProvider.ts:183-193`), if `telemetryOptOut` in `global-settings.json` is `false`, `this.tracerProvider.register()` registers the global OTLP `NodeTracerProvider` immediately.
   - `VscodeTelemetryPolicyService.initializeHostTelemetryState()` (`apps/vscode/src/sdk/sdk-telemetry.ts:176-189`) calls `this.emitProviderCreatedOnce()` -> `this.handle.emitProviderCreated?.()` (which bypasses `VscodeTelemetryPolicyService` and calls `telemetry.captureRequired("telemetry.provider_created", ...)` in `sdk/packages/core/src/services/telemetry/OpenTelemetryProvider.ts:381-399`) **before** `HostProvider.env.getTelemetrySettings({})` resolves.
   - `resolveAiSdkTelemetry` (`sdk/packages/llms/src/services/langfuse-telemetry.ts:217-265`) checks `getHostOtlpTracer()` and `isTelemetryOptedOutGlobally()`, **never `vscode.env.isTelemetryEnabled`**. If VS Code telemetry is OFF while `telemetryOptOut` is `false`, `cline`/`cline-pass` requests still export full AI SDK/Langfuse trace spans (including prompts and outputs when `CLINE_TRACE_RECORD_CONTENT="true"`).
3. **`apps/vscode-rollout` Wrapper Ignores All Opt-Outs & Self-Hosted Mode:**
   - On every VS Code activation, `refreshCohort()` resolves the persistent machine/generated ID (`cline.generatedMachineId` -> `node-machine-id` `machineId()` -> `vscode.env.machineId`) and `POST`s `{ api_key, distinct_id }` to `https://data.cline.bot/decide?v=3` **without checking `telemetrySetting`, `telemetryOptOut`, `vscode.env.isTelemetryEnabled`, or `endpoints.json` self-hosted mode** (`apps/vscode-rollout/src/rollout.ts:19-45,82-116,139-148`).
4. **PostHog Feature Flags & Webview Identification Ignore Telemetry Opt-Out:**
   - `FeatureFlagsService` explicitly documents `"works independently of telemetry settings"` (`apps/vscode/src/services/feature-flags/FeatureFlagsService.ts:15-20`) and polls `https://data.cline.bot` using `getDistinctId()` via `PostHogFeatureFlagsProvider` (`apps/vscode/src/services/feature-flags/providers/PostHogFeatureFlagsProvider.ts:28-31,45-59`).
   - Webview `CustomPostHogProvider` (`apps/vscode/webview-ui/src/CustomPostHogProvider.tsx:22-56`) initializes `posthog-js` unconditionally (unless `environment === "selfHosted"`) with `advanced_disable_decide: false` and `before_send: () => null`, and calls `posthog.identify(distinctId, { email: userInfo?.email, name: userInfo?.displayName })` even when `!isTelemetryEnabled` before calling `posthog.opt_out_capturing()`.
   - CLI (`apps/cli/src/utils/feature-flags.ts:26-78`), Desktop sidecar (`apps/examples/desktop-app/sidecar/feature-flags.ts:117-162`), and detached Hub daemon (`sdk/packages/core/src/services/feature-flags/cline-account-feature-flags.ts:28-70`) all poll `https://data.cline.bot` regardless of `telemetryOptOut`, sending `resolveCoreDistinctId()` when signed out or `accountId` (and `email` in CLI/Desktop context; Desktop also writes `userId`/`email` to `~/.cline/data/cache/feature-flags-account.cline-code.json`).
5. **Runtime & Env Overrides (`bypassUserSettings: true`):**
   - Setting `CLINE_OTEL_TELEMETRY_ENABLED="true"` (`apps/vscode/src/services/telemetry/TelemetryProviderFactory.ts:121-130`, `apps/vscode/src/shared/services/config/otel-config.ts:180`) or receiving organization Remote Config OTEL settings (`apps/vscode/src/core/storage/remote-config/utils.ts:251-257`) instantiates `OpenTelemetryTelemetryProvider` with `bypassUserSettings: true` (`apps/vscode/src/services/telemetry/providers/opentelemetry/OpenTelemetryTelemetryProvider.ts:197-202`), overriding both Cline and VS Code opt-outs.
   - Setting `remoteConfig.telemetryEnabled` in enterprise Remote Config locks the VS Code checkbox (`GeneralSettingsSection.tsx:30,37-39`, `remote-config/utils.ts:48-50`).
6. **CLI / Desktop / Hub Daemon Restart Lag:**
   - In `sdk/packages/core/src/services/telemetry/OpenTelemetryProvider.ts:417-432` and `OpenTelemetryAdapter.ts:56,63-68,177-179`, `isTelemetryOptedOutGlobally()` is only checked at service construction (`enabled` is a static boolean on `OpenTelemetryAdapter`). Toggling `telemetryOptOut: true` stops Langfuse AI SDK trace recording on the next turn (`langfuse-telemetry.ts:235-237`), but OTLP logs and metrics in the running CLI, Desktop sidecar, and detached `cline hub` daemon continue emitting until those processes are restarted (confirmed in `docs/enterprise-solutions/monitoring/telemetry.mdx:58`).
7. **Banner Dismissal Tracking Unchecked:**
   - `BannerService.sendBannerEvent()` `POST`s `{ banner_id, instance_id: this.hostInfo.distinctId, surface, event_type }` to `${apiBaseUrl}/banners/v2/messages` without checking telemetry opt-out (`apps/vscode/src/services/banner/BannerService.ts:252-275`).

---

## 2. External Auth / Local Non-Workspace File Reads

- **Zero External Auth Harvesting for Telemetry:** Cline does **not** read SSH keys (`~/.ssh`), browser cookies, GitHub/npm/Docker tokens, or other coding agents' credential files (`~/.claude/.credentials.json`, `~/.codex/auth.json`) for telemetry or tracking.
  - Verified by `rg` across all non-test source files (`apps/vscode/src`, `sdk/`, `apps/cli/src`, `apps/examples/desktop-app/sidecar`) for `.codex`, `.claude`, `.opencode`, `.cursor`, `.windsurf`, `auth.json`, and `homedir`/`os.homedir`.
- **Conditional / User-Configured Non-Workspace Local Reads (Not Telemetry-Harvested):**
  1. **Session Import Adapters (`sdk/packages/core/src/services/session-import/`):**
     - `ClaudeCodeImportAdapter` reads `$CLAUDE_CONFIG_DIR/projects` or `~/.claude/projects/*.jsonl` (`claude-code.ts:220-251`, `paths.ts:42`).
     - `CodexImportAdapter` reads `$CODEX_HOME` or `~/.codex/sessions` and `session_index.jsonl` (`codex.ts:148-189`, `paths.ts:50`).
     - `OpencodeImportAdapter` snapshots `$XDG_DATA_HOME/opencode/opencode.db` (`~/.local/share/opencode/opencode.db`) (`opencode.ts:47-87`).
     - Verified callers: ONLY Desktop sidecar commands `list_importable_sessions` and `import_sessions` (`apps/examples/desktop-app/sidecar/commands.ts:2673-2720`, invoked from Desktop UI/onboarding import components). An `rg` over `sdk/packages/core/src/services/session-import/` confirms zero telemetry calls or auth/secret reads in the importer (matches are strictly local token-count fields and comments).
  2. **Skills, Rules & Optional Provider Configs:**
     - `getSkillsDirectoriesForScan` reads project/global `.clinerules/skills`, `.cline/skills`, `.claude/skills`, `~/.cline/skills`, `~/.agents/skills` (`apps/vscode/src/core/storage/skill-directories.ts:4-40`).
     - `community.ts` passes `settingSources = ["user", "project"]` to `claude-code` provider when the user explicitly selects Claude Code (`sdk/packages/llms/src/providers/vendors/community.ts:163-168`).
     - `aws-region.ts` reads `$AWS_CONFIG_FILE` or `~/.aws/config` solely for `[profile]` `region` resolution when Bedrock is used (`apps/cli/src/utils/aws-region.ts:36-46`).
     - `apps/vscode/src/services/auth/oca/utils/constants.ts:4` defines `~/.oca/config.json` for user-supplied Oracle Code Assist OAuth config when using OCA.
     - `apps/examples/desktop-app/sidecar/local-runtimes/lmstudio.ts:103,110` reads `~/.lmstudio-home-pointer` / `~/.lmstudio` for local LLM discovery.
  3. **Own Storage Paths:** `~/.cline/` (`~/.cline/endpoints.json`, `~/.cline/data/settings/global-settings.json`, `~/.cline/data/globalState.json`, `~/.cline/data/sessions/machine-id`, `~/.cline/data/cache/*`) and `~/Documents/Cline/{Rules,Workflows,MCP,Hooks}` (`apps/vscode/src/config.ts:130`, `apps/vscode/src/core/storage/disk.ts:48-107`).
  4. **Workspace-Scoped Git Metadata (When `cline`/`cline-pass` Active & Telemetry Enabled):** `VscodeGitTelemetry` runs `git status --porcelain=v2 --branch` and `git remote -v` in the active task workspace `cwd` after model responses (`apps/vscode/src/sdk/git-telemetry.ts:35-93,106`).

---

## 3. Persistent Device / Install IDs & Hashing

### Persistent Identifiers Generated & Transmitted

1. **VS Code Hardware Machine ID / Fallback UUID (`apps/vscode/src/services/logging/distinctId.ts:10-96`):**
   - `initializeDistinctId` is called **unconditionally during `StateManager.initialize()`** at extension startup (`apps/vscode/src/core/storage/StateManager.ts:100`), even when telemetry is disabled.
   - Resolution order: `globalState["cline.generatedMachineId"]` -> `node-machine-id`'s `machineId()` (OS hardware/registry/machine-id fingerprint) -> `"cl-" + uuidv4()` persisted to `globalState["cline.generatedMachineId"]` (`distinctId.ts:25-39`).
   - Stored as both `_deviceId` (never cleared on sign-in or sign-out, attached as `device_id` on every event in `TelemetryService.ts:466,515-518`) and initial `_distinctId` (updated to `userInfo.id` upon account authentication via `setDistinctId`).
2. **A/B Rollout Loader ID (`apps/vscode-rollout/src/rollout.ts:31-45`):**
   - Independently derives `distinctId` in order: `~/.cline/data/globalState.json["cline.generatedMachineId"]` -> `node-machine-id`'s `machineId()` -> `vscode.env.machineId`, transmitted to `https://data.cline.bot/decide?v=3` on every activation.
3. **Core SDK / CLI / Desktop / Hub Daemon Hardware ID (`sdk/packages/core/src/services/telemetry/distinct-id.ts:12-83`):**
   - `resolveCoreDeviceId()` and `resolveCoreDistinctId()` invoke `node-machine-id`'s `machineIdSync()`, falling back to `"cl-" + nanoid()` persisted to `~/.cline/data/sessions/machine-id` (`distinct-id.ts:44-82`).
   - Core `TelemetryService` unconditionally resolves `this.deviceId = options.deviceId ?? resolveCoreDeviceId()` in its constructor and attaches `device_id` to every telemetry event (`sdk/packages/core/src/services/telemetry/TelemetryService.ts:38,152`).
   - CLI and Desktop feature-flag services also attach `resolveCoreDistinctId()` when unauthenticated (`apps/cli/src/utils/feature-flags.ts:31-33`, `apps/examples/desktop-app/sidecar/feature-flags.ts:122-124`).
4. **Desktop Boot ID & Ephemeral Window IDs:**
   - Desktop sidecar generates `bootId: randomUUID()` per process (`apps/examples/desktop-app/sidecar/context.ts:669`); `VscodeGitTelemetry` generates `windowId = randomUUID()` (`apps/vscode/src/sdk/git-telemetry.ts:145`).
5. **Cross-Provider Account Stitching (`sdk/packages/core/src/auth/*.ts`, `sdk/packages/core/src/services/telemetry/core-events.ts:428-451`):**
   - Authenticating with **Cline**, **OpenAI Codex** (`openai-codex` ChatGPT Subscription OAuth, `sdk/packages/core/src/auth/codex.ts:380-384`), or **Oracle Code Assist** (`oca` OAuth, `sdk/packages/core/src/auth/oca.ts:427-431`) calls `identifyAccount(options.telemetry, { id: credentials.accountId, email: credentials.email, provider })`, overwriting `distinctId` with the provider's account ID and persisting `user_id`, `account_id`, `account_email`, and `provider` in telemetry `commonProperties` alongside the permanent hardware `device_id`.

### Hashing Practices

- **`workspace_id` in `task.git_snapshot` (`apps/vscode/src/sdk/git-telemetry.ts:13,181-183`):** HMAC-SHA256 using an ephemeral in-memory 32-byte random key (`randomBytes(32)`) rotated on host restart, hashing `${sessionId}\0${resolve(cwd)}`.
- **Langfuse Sampling (`sdk/packages/llms/src/services/langfuse-telemetry.ts:249`):** Non-cryptographic `fnv1a32(samplingKey)` modulo 100 to decide trace sampling per task/session.
- **Persistent Device / Account IDs:** Hardware `machineId()` and fallback `cl-*` UUIDs/nanoids, as well as `user_id`, `account_email`, `user_name`, and `organization_name`, are transmitted **unhashed / verbatim** in telemetry, OTLP attributes, and PostHog requests.

---

## 4. Exact Outbound Telemetry / Analytics / Crash / OTEL / Relay / Update Endpoints

1. **PostHog Analytics & Feature Flags Reverse Proxy:**
   - `https://data.cline.bot` (`POST /batch/`, `/i/v0/e/`, `/decide?v=3`, flags endpoints) via `posthog-node` and `posthog-js`:
     - VS Code backend PostHog client & feature flags (`apps/vscode/src/shared/services/config/posthog-config.ts:43`, `apps/vscode/src/services/telemetry/providers/posthog/PostHogClientProvider.ts:32-34`, `apps/vscode/src/services/feature-flags/providers/PostHogFeatureFlagsProvider.ts:28-31`).
     - VS Code webview `posthog-js` (`apps/vscode/webview-ui/src/CustomPostHogProvider.tsx:22-34`, plus `uiHost: https://us.posthog.com` or `https://us.i.posthog.com` in `posthog-config.ts:44`).
     - VS Code A/B rollout wrapper `POST https://data.cline.bot/decide?v=3` (`apps/vscode-rollout/src/rollout.ts:19,88-91`).
     - Core SDK / CLI / Desktop / Hub daemon feature flags (`sdk/packages/core/src/services/feature-flags/posthog.ts:23-26`, `apps/cli/src/utils/feature-flags.ts:52`, `apps/examples/desktop-app/sidecar/feature-flags.ts:144`, `sdk/packages/core/src/services/feature-flags/cline-account-feature-flags.ts:58`).
2. **OpenTelemetry Collector Endpoints (Production Build-Time OTLP + Enterprise/Runtime Overrides):**
   - Configured at release build time via CI secrets `OTEL_EXPORTER_OTLP_ENDPOINT` / `OTEL_EXPORTER_OTLP_HEADERS` / `OTEL_EXPORTER_OTLP_PROTOCOL` (`.github/workflows/ext-vscode-publish.yml:215-224`, `ext-vscode-publish-stable.yml:272-279`, `ext-vscode-ab-package.yml:280-291`, `cli-publish.yml:143-166`, `desktop-publish.yml:312-355`). The collector URL and auth headers are injected from GitHub Secrets at CI time (not hardcoded in plaintext in source).
   - Exported paths appended by OTLP HTTP/Proto exporters: `<otlpEndpoint>/v1/logs`, `<otlpEndpoint>/v1/metrics`, `<otlpEndpoint>/v1/traces` (`apps/vscode/src/services/telemetry/providers/opentelemetry/OpenTelemetryExporterFactory.ts:51-53,98-99`, `sdk/packages/core/src/services/telemetry/OpenTelemetryProvider.ts:297-350`), or gRPC/runtime overrides via `CLINE_OTEL_EXPORTER_OTLP_*` and enterprise Remote Config (`apps/vscode/src/shared/services/config/otel-config.ts:178-207`, `apps/vscode/src/core/storage/remote-config/utils.ts:106-129,243-257`).
   - Desktop webview errors forward via local loopback `POST /telemetry/error` on the Desktop Bun sidecar (`apps/examples/desktop-app/sidecar/server.ts:265-326`) into `captureSdkError(...)` -> OTLP logs/events.
3. **Direct Langfuse Endpoint (Env-Var Configured Only, Not Default):**
   - When operator/user sets `LANGFUSE_PUBLIC_KEY`, `LANGFUSE_SECRET_KEY`, and `LANGFUSE_BASE_URL` (with `CLINE_OTEL_TRACE_ALL_PROVIDERS` for non-Cline providers), `langfuse-telemetry.ts` exports spans directly to Langfuse (`sdk/packages/llms/src/services/langfuse-telemetry.ts:4-10,202-233`).
4. **Cline API, App, Banners & Remote Config Endpoints:**
   - Production: `https://api.cline.bot`, `https://app.cline.bot`, `https://api.cline.bot/v1/mcp` (`apps/vscode/src/config.ts:337-343`); Staging: `https://staging-app.cline.bot`, `https://core-api.staging.int.cline.bot` (`config.ts:310-315`).
   - Banner event tracking: `POST https://api.cline.bot/banners/v2/messages` (`apps/vscode/src/services/banner/BannerService.ts:254-274`).
   - Account/remote config endpoints: `/api/v1/users/me/remote-config`, `/api/v1/organizations/{id}/remote-config`, `/api/v1/users/me`, etc. (`apps/vscode/src/shared/cline/api.ts:6-14`).
5. **Enterprise Prompt Storage (Remote-Config Driven):**
   - User/admin-configured AWS S3, Cloudflare R2, or Azure Blob buckets uploading conversation history from local queue (`apps/vscode/src/core/storage/remote-config/utils.ts:228-237`, `apps/vscode/src/shared/services/worker/worker.ts:231`, `docs/enterprise-solutions/monitoring/prompt-storage.mdx:6-11`).
6. **Auto-Update Endpoints (Non-Telemetry):**
   - **CLI:** Startup version check `GET https://registry.npmjs.org/cline/<tag>` (and `"kanban"`) (`apps/cli/src/commands/update.ts:189,200,311-323`).
   - **Desktop (Tauri Updater):** Release channel `https://github.com/cline/cline/releases/download/desktop-latest/latest.json` (`apps/examples/desktop-app/src-tauri/tauri.conf.json:13-17`); Beta channel `https://github.com/cline/cline/releases/download/desktop-beta/latest.json` (`tauri.beta.conf.json:20-23`); Nightly `endpoints: []` (`tauri.nightly.conf.json:20-22`).

---

## 5. Exact Data & Properties Collected

1. **IP & Geolocation Handling:**
   - **Zero IP/GeoIP suppression:** An `rg` across the entire repository for `disable_geoip`, `disableGeoip`, and `disableGeoIP` returned `0` matches. Neither `posthog-node` (`PostHogClientProvider.ts:32-37`, `sdk/packages/core/src/services/feature-flags/posthog.ts:23-26`), `posthog-js` (`CustomPostHogProvider.tsx:22-34`), `apps/vscode-rollout` (`rollout.ts:88-91`), nor OTLP exporters disable GeoIP enrichment or strip client IP at the closed `https://data.cline.bot` reverse proxy / OTLP collector.
2. **Custom / Private Model Names & Provider IDs:**
   - **Transmitted verbatim (unhashed, unredacted):** `captureTaskCreated`, `captureConversationTurnEvent`, `captureTokenUsage`, `captureToolUsage`, `captureTaskLifecycleEvent`, and `captureAgentUnexpectedReasoningTokens` attach raw `provider`/`providerId` and `model`/`modelId` directly from session config (`sdk/packages/core/src/services/agent-events.ts:224-281`, `sdk/packages/core/src/runtime/host/local-runtime-host.ts:1436-1437,2198-2199`, `sdk/packages/core/src/services/telemetry/core-events.ts:477-585`, `sdk/packages/shared/src/services/telemetry.ts:259-307`). Custom Ollama, LM Studio, LiteLLM, or private OpenAI-compatible model names are emitted as-is.
3. **Prompts, Code, Paths, URLs & Stack Traces:**
   - **Full Prompt & Output Capture for `cline` / `cline-pass`:** Release CI workflows set `OTEL_TRACES_EXPORTER: otlp` and `CLINE_TRACE_RECORD_CONTENT: "true"` (`.github/workflows/ext-vscode-publish-stable.yml:275-276`, `cli-publish.yml:162-163`, `desktop-publish.yml:315-316,351-352`). In `sdk/packages/llms/src/services/langfuse-telemetry.ts:258-265`, for `cline`/`cline-pass` requests when `telemetryOptOut` is `false`, `resolveAiSdkTelemetry` attaches `LangfuseStreamTelemetry` with `recordInputs: true` and `recordOutputs: true`, exporting full prompts, system instructions, tool outputs, and model responses in OTLP trace spans. (Third-party BYOK providers do **not** emit AI SDK trace spans unless `CLINE_OTEL_TRACE_ALL_PROVIDERS` and `LANGFUSE_*` env vars are explicitly set, `langfuse-telemetry.ts:202-215`).
   - **Git Metadata (`task.git_snapshot`, `cline`/`cline-pass` only):** Emits raw `git.branch`, raw `git.head_sha`, `git.dirty`/`staged`/`unstaged`/`untracked` booleans, and sanitized `git.remote_url` (stripping credentials/query/hash while preserving host and repo path), alongside HMAC-hashed `workspace_id` (`apps/vscode/src/sdk/git-telemetry.ts:18-93,181-183,214-229`).
   - **Error Messages, Paths & Browser URLs:**
     - `sanitizeTelemetryErrorMessage` (`sdk/packages/shared/src/services/telemetry.ts:426-437`) redacts `Bearer` tokens, `api_key`/`password`/`secret` query pairs, and user home directory segments (`/Users/[redacted]`, `/home/[redacted]`, `C:\Users\[redacted]`), truncating `error_message` to 500 chars; however, non-home file paths, hostnames, error types, status codes, and messages outside those regexes are sent verbatim.
     - Desktop webview error bridge accepts `sourceUrl`, `lineno`, `colno`, `stack` (truncated to 500 chars) and forwards them via `captureSdkError` (`apps/examples/desktop-app/sidecar/server.ts:301-325`).
     - Classic `TelemetryService.captureBrowserError` forwards raw `errorMessage` and `context` (`action`, `url`, `remoteBrowserHost`, `endpoint`) (`apps/vscode/src/services/telemetry/TelemetryService.ts:859-884`).
     - `apps/vscode-rollout/src/extension.ts:187,201-205` captures `error.message + "\n" + error.stack` (up to 2000 chars) on fallback bundle activation failure.
4. **Behavioral Profile & Identity Fields:**
   - Persistent `device_id` (hardware `machineId` or generated `cl-*` UUID/nanoid) and `distinct_id` on every event (`TelemetryService.ts:38,152`, `apps/vscode/src/services/telemetry/TelemetryService.ts:466,515-518`).
   - User/Org identity attributes: `user_id`, `account_id`, `account_email`, `user_name` (`displayName`), `email`, `name`, `provider`, `organization_id`, `organization_name`, `member_id`, `member_role`, `alias` (`OpenTelemetryTelemetryProvider.ts:180-195`, `core-events.ts:443-450`, `CustomPostHogProvider.tsx:45-49`).
   - Task & execution metrics: `task.created`/`restarted`/`completed` (`durationMs`, `source`), `task.conversation_turn` (`source`, `mode`, `provider`, `model`), `task.tokens` (`tokensIn`, `tokensOut`, `cacheWriteTokens`, `cacheReadTokens`, `reasoningTokenCount`, `totalCost`), `task.mode`, `task.tool_used` (`tool`, `success`, `autoApproved`), `task.skill_used`, `task.mention_used`, `task.compaction_*`, `task.provider_request_started`/`stream_started`/`first_chunk_received`/`stream_failed`, `sdk.error`, `user.extension_activated`, `user.auth_*`, `user.provider_configured`, `user.opt_out`, `remote_config.refresh`/`session_gate`, and host/OS metadata (`extension_version`, `cline_type`, `platform`, `platform_version`, `os_type`, `os_version`, rollout bundle metadata) (`sdk/packages/core/src/services/telemetry/core-events.ts:44-120`, `sdk/packages/shared/src/services/telemetry-config.ts:36-46`, `apps/vscode/src/services/telemetry/TelemetryService.ts:617-653`).

---

## 6. Docs / Privacy Policy vs Code Discrepancies

No `PRIVACY*` file exists in the repository (`0` matches); `README.md`, `apps/cli/README.md`, and `apps/vscode/README.marketplace.md` contain zero telemetry disclosures (`apps/vscode/README.md` is empty). Privacy/telemetry behavior is documented in `docs/enterprise-solutions/monitoring/*.mdx`, `GeneralSettingsSection.tsx:44-61`, and `apps/vscode-rollout/README.md`, yielding these verified discrepancies:

1. **VS Code Host Telemetry Claim (`docs/enterprise-solutions/monitoring/telemetry.mdx:39`) vs Code:**
   - **Docs claim:** *"In VS Code, Cline also stops sending usage events while the editor's own telemetry is off."*
   - **Code reality:** When VS Code's `telemetry.telemetryLevel` is off (and Cline's `telemetryOptOut` is `false`), `OpenTelemetryProvider` registers its global OTLP `NodeTracerProvider` immediately at construction (`OpenTelemetryProvider.ts:183-193`), `VscodeTelemetryPolicyService.initializeHostTelemetryState()` fires `telemetry.provider_created` unconditionally before checking VS Code settings (`sdk-telemetry.ts:176-189`), `langfuse-telemetry.ts:217-265` ignores `vscode.env.isTelemetryEnabled` (so `cline`/`cline-pass` prompt/response traces still export), classic `logRequired` bypasses `hostEnabled` (`OpenTelemetryTelemetryProvider.ts:129-143`, `PostHogTelemetryProvider.ts:93-102`), and `apps/vscode-rollout` sends hardware `machineId` to `https://data.cline.bot/decide?v=3` (`rollout.ts:82-148`).
2. **Anonymized User IDs & Hashed Branches (`docs/enterprise-solutions/monitoring/opentelemetry-events.mdx:362,367-378`) vs Code:**
   - **Docs claim:** *"`user_id`: Anonymized user identifier"*, *"User identifiers: Anonymized tokens"*, *"Branch names: Hashed in worktree events"*, and *"File paths, command arguments, and code content are never included in raw form."*
   - **Code reality:** Telemetry transmits raw `user_id`, `account_id`, `account_email`, `user_name`, `organization_id`, `organization_name` (`OpenTelemetryTelemetryProvider.ts:184-192`, `core-events.ts:443-450`), and webview `email`/`name` (`CustomPostHogProvider.tsx:45-49`); `VscodeGitTelemetry` sends **raw** `git.branch`, `git.head_sha`, and sanitized `git.remote_url` (`apps/vscode/src/sdk/git-telemetry.ts:61,67,72,214-229`); and release builds enable `CLINE_TRACE_RECORD_CONTENT: "true"` exporting full prompts/code/responses for `cline`/`cline-pass` (`langfuse-telemetry.ts:259-265`). (By contrast, `telemetry.mdx:23-25` and `GeneralSettingsSection.tsx:45-46` do disclose the `cline`/`cline-pass` prompt exception.)
3. **Cline Account Only Claim (`docs/enterprise-solutions/monitoring/telemetry.mdx:7,19`) vs Third-Party OAuth Identification:**
   - **Docs claim:** *"When you're signed in to a Cline account, events also identify your account and organization."*
   - **Code reality:** Signing into third-party **OpenAI Codex** (`openai-codex` ChatGPT Subscription) or **Oracle Code Assist** (`oca`) via OAuth also invokes `identifyAccount(telemetry, { id: credentials.accountId, email: credentials.email, provider })` (`sdk/packages/core/src/auth/codex.ts:380-384`, `sdk/packages/core/src/auth/oca.ts:427-431`), attaching external account IDs and emails to `device_id`.
4. **Prompt Storage S3 Key Path (`docs/enterprise-solutions/monitoring/prompt-storage.mdx:74`) vs Code:**
   - **Docs claim:** `s3://your-bucket/tasks/{taskId}/api_conversation_history.json`
   - **Code reality:** `SyncWorker` actually uploads to `tasks/${this.options.userDistinctId}/${item.taskId}/${item.key}` (`apps/vscode/src/shared/services/worker/worker.ts:231`), embedding the user/machine `distinctId` in every blob path.
5. **Misleading `CLINE_TELEMETRY_DISABLED` Variable in Own Tests (`apps/cli/src/tests/helpers/env.ts:93`):**
   - Cline's own CLI test harnesses set `CLINE_TELEMETRY_DISABLED: "1"`, even though zero production code reads `CLINE_TELEMETRY_DISABLED`.

---

## 7. Zero-Telemetry / Negative-Check Verification

This repository is **NOT** zero-telemetry. Documented negative `rg` checks across `repos/cline`:

- **Third-Party Telemetry SDKs Absent (`0` matches):** Case-insensitive repo-wide `rg` for `@sentry`, `sentry.io`, `Sentry.init`, `bugsnag`, `analytics-node`, `segment.io`, `mixpanel`, `amplitude`, `telemetrydeck`, `plausible`, and `matomo` returned **0 matches**.
- **`ErrorProviderFactory` Default (`apps/vscode/src/services/error/ErrorProviderFactory.ts:54-59`):** Defaults to `type: "no-op"` (meaning the separate PostHog error provider is inactive unless explicitly switched; errors instead flow through normal telemetry / SDK OTLP `sdk.error`).
- **Missing Privacy/Opt-Out Primitives (`0` production matches):**
  - `DO_NOT_TRACK`: **0 matches** across entire repo.
  - `CLINE_TELEMETRY_DISABLED`: **0 production matches** (matched only in 3 CLI test files setting `CLINE_TELEMETRY_DISABLED: "1"`).
  - `disable_geoip` / `disableGeoip` / `disableGeoIP`: **0 matches** across entire repo.
  - `PRIVACY*` files: **0 files** found in repo.

---

```json
{
  "slug": "cline",
  "platform": "Cline",
  "group": "B",
  "repo": "cline/cline",
  "sha": "34c104e6fc49781cda4adcd73ffdccd8f029fef9",
  "commit_date": "2026-11-23T13:06:22+01:00",
  "license": "Apache-2.0",
  "status": "Active",
  "default_state": "Opt-Out (ON by default across VS Code extension, A/B rollout wrapper, CLI, Hub daemon, and Desktop Tauri app via telemetryOptOut=false in global-settings.ts:63 and telemetrySetting!='disabled'; release CI workflows bake in PostHog keys plus OTLP logs/metrics/traces with CLINE_TRACE_RECORD_CONTENT='true')",
  "opt_out_controls": [
    "VS Code UI checkbox 'Allow error and usage reporting' (GeneralSettingsSection.tsx:28-36) sets telemetryOptOut=true in ~/.cline/data/settings/global-settings.json and telemetrySetting='disabled'",
    "VS Code host telemetry.telemetryLevel / vscode.env.isTelemetryEnabled gates ordinary VS Code events (sdk-telemetry.ts:238-240), though with startup/trace bypasses",
    "Desktop UI Settings -> Telemetry switch invokes sidecar command set_telemetry_opt_out (commands.ts:3340-3346, settings-view.tsx:1024-1029)",
    "CLI has NO flag, subcommand, or TUI toggle for telemetryOptOut; requires manual JSON edit to ~/.cline/data/settings/global-settings.json (telemetry.mdx:46-54) plus restarting CLI and 'cline hub stop'",
    "Self-hosted ~/.cline/endpoints.json or bundled endpoints.json disables main VS Code PostHog and build-time OTEL (config.ts:85-92,130, PostHogClientProvider.ts:25-28, TelemetryProviderFactory.ts:113), but NOT apps/vscode-rollout",
    "CLI/Desktop auto-update can be disabled via CLINE_NO_AUTO_UPDATE=1 or autoUpdateEnabled=false (update.ts:313-314)"
  ],
  "do_not_track": false,
  "opt_out_bypasses": [
    "Classic TelemetryService.captureRequired -> OpenTelemetryTelemetryProvider.logRequired and PostHogTelemetryProvider.logRequired emit regardless of telemetrySetting or VS Code hostEnabled (OpenTelemetryTelemetryProvider.ts:129-143, PostHogTelemetryProvider.ts:93-102)",
    "VS Code VscodeTelemetryPolicyService.captureRequired checks only hostTelemetryEnabled and ignores Cline's telemetrySetting='disabled' (sdk-telemetry.ts:110-115,242-248); opting out fires required event 'user.opt_out' (global-settings.ts:233-235, core-events.ts:418-426), and plugins with required=true bypass opt-out (agent-event-bridge.ts:283-287)",
    "When VS Code telemetry is OFF but global-settings telemetryOptOut=false, OpenTelemetryProvider registers the global OTLP NodeTracerProvider at construction (OpenTelemetryProvider.ts:183-193), VscodeTelemetryPolicyService fires telemetry.provider_created before HostProvider.env.getTelemetrySettings resolves (sdk-telemetry.ts:176-189), and langfuse-telemetry.ts:217-265 ignores vscode.env.isTelemetryEnabled so cline/cline-pass prompt traces still export",
    "apps/vscode-rollout POSTs hardware/generated machineId to https://data.cline.bot/decide?v=3 on every VS Code startup ignoring telemetrySetting, vscode.env.isTelemetryEnabled, and self-hosted endpoints.json (rollout.ts:31-45,82-148)",
    "PostHog feature-flag polling (FeatureFlagsService in VS Code, CLI, Desktop, and Hub daemon) explicitly ignores telemetry opt-out and sends distinctId (machineId or accountId; Desktop caches userId/email to disk) to https://data.cline.bot (FeatureFlagsService.ts:15-20, cli/feature-flags.ts:26-78, desktop/feature-flags.ts:117-162)",
    "Webview CustomPostHogProvider initializes posthog-js (unless selfHosted) with advanced_disable_decide=false and calls posthog.identify(distinctId, { email, name }) even when telemetrySetting=='disabled' before calling opt_out_capturing() (CustomPostHogProvider.tsx:22-56)",
    "CLI/Desktop/Hub daemon OpenTelemetryAdapter caches enabled status at startup (OpenTelemetryProvider.ts:417-432, OpenTelemetryAdapter.ts:56,177-179); setting telemetryOptOut=true does not stop OTLP events/metrics until process and detached 'cline hub' daemon restart",
    "CLINE_OTEL_TELEMETRY_ENABLED=true and org Remote Config OTEL set bypassUserSettings=true (TelemetryProviderFactory.ts:121-130, remote-config/utils.ts:251-257), and BannerService.sendBannerEvent POSTs instance_id without opt-out checks (BannerService.ts:252-275)",
    "CLINE_TELEMETRY_DISABLED and OTEL_SDK_DISABLED are ignored by all production code (even though Cline's own CLI tests set CLINE_TELEMETRY_DISABLED='1')"
  ],
  "persistent_ids": [
    "node-machine-id hardware machineId() / machineIdSync() initialized unconditionally at startup even when opted out (distinctId.ts:25-58, StateManager.ts:100, distinct-id.ts:12-55, TelemetryService.ts:38,152)",
    "Persistent fallback UUID/nanoid: VS Code globalState['cline.generatedMachineId'] ('cl-' + uuidv4) and Core ~/.cline/data/sessions/machine-id ('cl-' + nanoid) (distinctId.ts:37-38, distinct-id.ts:57-82)",
    "VS Code A/B rollout wrapper reads cline.generatedMachineId -> node-machine-id -> vscode.env.machineId (rollout.ts:31-45)",
    "Authenticated account/org identifiers stitched onto device_id: user_id, account_id, account_email, user_name, organization_id, organization_name, member_id across Cline, openai-codex, and oca OAuth flows (core-events.ts:428-450, codex.ts:380-384, oca.ts:427-431)"
  ],
  "hashing": "None for persistent device_id, distinct_id, user_id, account_email, user_name, or git branch/SHA/remote_url (sent unhashed/verbatim). Ephemeral per-process HMAC-SHA256 (randomBytes(32)) hashes workspace_id in task.git_snapshot (git-telemetry.ts:13,181-183); non-cryptographic fnv1a32(samplingKey) is used for trace sampling (langfuse-telemetry.ts:249).",
  "external_auth_reads": "None for telemetry or tracking (0 reads of ~/.ssh, ~/.claude/.credentials.json, ~/.codex/auth.json, or browser tokens). Non-workspace reads are strictly feature-gated/local: ~/.claude/projects, ~/.codex/sessions, and ~/.local/share/opencode/opencode.db are read only via Desktop SessionImportService (session-import/paths.ts, commands.ts:2673-2720) with zero telemetry export; ~/.aws/config read only for Bedrock region resolution (aws-region.ts:36-46); ~/.oca/config.json for OCA OAuth config; ~/.claude/skills and ~/.agents/skills for user skills.",
  "telemetry_endpoints": [
    "https://data.cline.bot (closed PostHog ingest & feature-flag /decide?v=3 reverse proxy; us.posthog.com / us.i.posthog.com uiHost)",
    "Production CI-inlined OTLP collector (<OTEL_EXPORTER_OTLP_ENDPOINT>/v1/logs, /v1/metrics, /v1/traces via HTTP/Proto/gRPC from GitHub Secrets) + optional CLINE_OTEL_* / enterprise Remote Config OTLP / LANGFUSE_* collectors",
    "https://api.cline.bot/banners/v2/messages (banner event POSTs) and https://api.cline.bot/api/v1/* / https://app.cline.bot (auth, remote config)",
    "Enterprise Remote Config user/admin-configured S3 / Cloudflare R2 / Azure Blob buckets (Prompt Storage worker.ts:231)",
    "Auto-updaters (non-telemetry): https://registry.npmjs.org/cline/<tag> (CLI) and https://github.com/cline/cline/releases/download/desktop-{latest,beta}/latest.json (Desktop)"
  ],
  "ip_geo_handling": "No IP or GeoIP suppression anywhere (0 occurrences of disable_geoip/disableGeoip); client IP is visible to the closed https://data.cline.bot PostHog proxy, CI-configured OTLP collector, api.cline.bot, and npm/GitHub updaters.",
  "custom_model_handling": "Verbatim / Unsanitized: custom, local, LiteLLM, Ollama, LM Studio, and OpenAI-compatible providerId and model/modelId strings are emitted directly in task.created, task.conversation_turn, task.tokens, task.tool_used, and task lifecycle events (agent-events.ts:224-281, core-events.ts:477-585).",
  "docs_discrepancies": [
    "docs/enterprise-solutions/monitoring/telemetry.mdx:39 claims VS Code stops usage events when editor telemetry is off, but VscodeTelemetryPolicyService fires telemetry.provider_created and registers NodeTracerProvider before checking host telemetry, langfuse-telemetry.ts ignores vscode.env.isTelemetryEnabled, classic logRequired ignores hostEnabled, and apps/vscode-rollout ignores all opt-outs",
    "docs/enterprise-solutions/monitoring/opentelemetry-events.mdx:362,367-378 claims anonymized user tokens, hashed branch names, and no raw code/paths, whereas code sends raw user_name/organization_name/account_email/email, raw git.branch/head_sha/remote_url in task.git_snapshot, and full prompts/responses for cline/cline-pass when CLINE_TRACE_RECORD_CONTENT='true'",
    "docs/enterprise-solutions/monitoring/telemetry.mdx:7,19 implies account identification is for Cline accounts only, whereas openai-codex and oca OAuth logins also invoke identifyAccount with third-party accountId and email (codex.ts:380-384, oca.ts:427-431)",
    "docs/enterprise-solutions/monitoring/prompt-storage.mdx:74 omits userDistinctId from the S3 key path documented as tasks/{taskId}/api_conversation_history.json (worker.ts:231 uploads tasks/{userDistinctId}/{taskId}/{key})",
    "No PRIVACY* file exists in repo, README.md/apps/cli/README.md/apps/vscode/README.marketplace.md omit telemetry disclosures, and CLI tests set CLINE_TELEMETRY_DISABLED='1' which no production code reads"
  ],
  "privacy_rating": "F",
  "one_line_verdict": "Opt-out by default meriting F due to active opt-out bypasses across rollout, feature-flag polling, captureRequired, and VS Code host-telemetry-off trace/event paths, baked-in PostHog (https://data.cline.bot) and OTLP logs/metrics/full-content cline-provider traces, hardware machineId initialization even when disabled, no DO_NOT_TRACK/CLINE_TELEMETRY_DISABLED/CLI-toggle support, no IP suppression, and verbatim custom models."
}
```
