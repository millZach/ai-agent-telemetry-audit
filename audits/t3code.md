# Forensic Telemetry & Privacy Audit: T3 Code (`pingdotgg/t3code`)

**Baseline vs. Live `HEAD` Delta Verification**

- **Slug:** `t3code`
- **Platform:** T3 Code (`pingdotgg/t3code`)
- **Group:** `A`
- **Repository:** `https://github.com/pingdotgg/t3code`
- **License:** `MIT`
- **Status:** Active (`main`)
- **Baseline Audited SHA:** `4c2e2b373433b729b4080f8bab42aa0754b63304` (`2026-11-24T08:45:32-08:00`, `fix(web,mobile): a 59.5-second workflow step reads 1m 0s, not 60s (#42248)`)
- **Live `HEAD` SHA:** `2b84f4862ce7320bdff28cef88d3ed8d0f23797e` (`2026-11-24T20:51:11-08:00`, `fix(pi): reject generated text from failed responses (#42602)`)
- **Commits Between Baseline & Live `HEAD`:** `152` commits
- **Privacy Rating (Baseline `4c2e2b37`):** **`F`** (Severe Red Flags)
- **Privacy Rating (Live `HEAD` `2b84f486`):** **`D`** (Aggressive / Leaky — significant privacy remediations landed across 12+ PRs, removing external provider auth harvesting and verbatim custom model leakage, but default-on analytics without UI/config/CLI opt-out, unconditional `anonymous-id` creation when disabled, unsalted SHA-256 hashing, missing PostHog IP/GeoIP suppression behind a closed-source ingest proxy, remaining Desktop OTEL/hosted-web opt-out gaps, and Relay Axiom geo/Clerk `user.id` logging keep it from `C`)

---

## Baseline (`4c2e2b37`) vs. Live `HEAD` (`2b84f486`) Delta Table

| Area / Finding | Baseline (`4c2e2b37`) | Live `HEAD` (`2b84f486`) | Delta Verdict & Evidence |
|---|---|---|---|
| **External CLI auth reads for telemetry (`~/.codex/auth.json`, `~/.claude.json`)** | Read twice on every startup (even when disabled); `openai` & `claude` IDs stitched into `distinct_id` and `install.owner_observed` | **Removed from telemetry.** `Identify.ts` never opens Codex/Claude files; `distinct_id` = `installId`; `CodexChatGptAuth.ts` no longer calls `recordOwners` | **FIXED (`#42299`)** — `apps/server/src/telemetry/Identify.ts:18, 103-145`; `apps/server/src/provider/CodexChatGptAuth.ts:123-166` |
| **Unconditional local state/ID generation when opted out** | Created/read `anonymous-id`, read Codex/Claude files, and called `getServerOwners` even when disabled | `getTelemetryIdentity` (`upsertAnonymousId`) **still runs unconditionally** in `AnalyticsService.make` before checking `telemetryEnabled`; `getServerOwners` still eagerly reads/hashes `CLOUD_LINKED_USER_ID` | **PARTIAL (`#42299`):** Auth files gone, but `AnalyticsService.ts:152` & `serverRuntimeStartup.ts:178` still run unconditionally when disabled |
| **Custom / private model names** | Unmatched/Ollama/GGUF/Azure/custom-endpoint model IDs sent **verbatim** via `modelLabel` | Known bundled manifest / non-custom vendor catalog slugs mapped; **all other/custom/local models masked as `custom:<driver>`** | **FIXED (`#42465`)** — `apps/server/src/telemetry/ProviderDimensions.ts:43-92` |
| **Desktop relay tracing cold-start race** | `relayTracing.ts` captured `process.env` at import time before login-shell discovery finished | `allowedFor` returns `false` while `T3CODE_DESKTOP_SHELL_DISCOVERY_PENDING === "1"` and re-evaluates dynamically per span/flush; Desktop IPC returns `null` until settled | **FIXED (`#42464`)** — `apps/server/src/cloud/relayTracing.ts:21-45`; `packages/shared/src/relayTracing.ts:143-210`; `apps/desktop/src/ipc/methods/window.ts:103-125` |
| **Windows Desktop user env opt-out** | `T3CODE_TELEMETRY_ENABLED` ignored when launched without inherited env | PowerShell probe now checks `WINDOWS_PROFILE_ENV_NAMES` including `T3CODE_TELEMETRY_ENABLED` across Process -> User -> Machine scopes | **FIXED (`#42376`)** — `apps/desktop/src/shell/DesktopShellEnvironment.ts:132-137, 462-464`; `packages/shared/src/shell.ts:305-323` |
| **Desktop shell/Windows `OTEL_SDK_DISABLED` forwarding** | `LOGIN_SHELL_ENV_NAMES` omitted `T3CODE_OTEL_SDK_DISABLED` & `OTEL_SDK_DISABLED` | **Still omitted** in both `LOGIN_SHELL_ENV_NAMES` and `WINDOWS_PROFILE_ENV_NAMES` | **UNFIXED** — `apps/desktop/src/shell/DesktopShellEnvironment.ts:102-137, 549-571, 660-674` |
| **Browser & hosted web relay tracing** | Local browser & hosted `app.t3.codes` unconditionally returned `true` | Local-server browsers now stay off until `serverConfig.observability.relayTracingAllowed` arrives and honor server opt-out; **hosted web `app.t3.codes` still has NO opt-out** | **PARTIAL (`#42356`):** `apps/web/src/cloud/publicConfig.ts:69-80`; `apps/web/src/lib/runtime.ts:25-35`; `apps/web/src/lib/runtime.test.ts:43-65` |
| **Machine details on relay spans** | `OTEL_RESOURCE_ATTRIBUTES` / request address details could leak | `nonInterferingTracer` strips `url.full`/`url.query`/`server.address`/`server.port`, drops span events/stacks; `ConfigProvider` ignores operator `OTEL_RESOURCE_ATTRIBUTES` | **FIXED (`#42356`)** — `packages/shared/src/relayTracing.ts:73-140, 196`; `apps/server/src/cloud/relayTracing.ts:41-42` |
| **Relay-delivered webhook spans** | Webhook runs could export local spans (with workspace paths) to Relay Axiom | `runWebhookDelivery` wrapped in `withLocalTracing`, keeping run spans local | **FIXED (`#42450`)** — `apps/server/src/scheduledTasks/ScheduledTaskService.ts:2554-2561` |
| **Mobile connection traces & device ID** | No switch for mobile relay tracing; analytics reused `agent-awareness.device-id` | Dedicated **`connectionTracesEnabled`** switch (still **ON by default**, `!== false`) in Diagnostics & Connect onboarding; separate `t3code.analytics.device-id` created | **IMPROVED (`#42415`, `#42394`)** — `apps/mobile/src/features/observability/diagnosticsSetting.ts:1-19`; `ConnectionTracesSwitch.tsx:13-63`; `mobile-storage.ts:11-12, 194` |
| **Agent feedback install linkage** | `agent.feedback` included `installId` and host properties in shared batches | Sent with `{ anonymous: true }` in its own isolated single-event `/batch/` POST using `event.uuid` as `distinct_id` and omitting `installId`/OS/host metadata | **FIXED (`#42398`, `#42466`)** — `apps/server/src/feedback/FeedbackStore.ts:208-220`; `apps/server/src/telemetry/AnalyticsService.ts:215-222, 251-258` |
| **MCP tool analytics strings** | `handoffSettings` / `toolOutcome` could risk non-closed fields | Strictly closed enum allowlists (`oneOf`) + `ERROR_CODE_SHAPE` (`^[A-Za-z][A-Za-z0-9_]{0,63}$`) | **HARDENED (`#42397`)** — `apps/server/src/telemetry/ProviderDimensions.ts:147-222` |
| **Relay Worker IP, UA, geo, Clerk `user.id`** | Logged `remoteAddress`, `user-agent`, `relay.edge.colo`/`country`/`continent`, and raw Clerk `user.id` | Strips `remoteAddress` and `user-agent` from HTTP trace span request copy; **still logs `relay.edge.colo`/`country`/`continent` and raw Clerk `user.id` / `relay.auth.subject`** | **PARTIAL (`#42380`)** — `infra/relay/src/http/Api.ts:213-265, 328-339, 394-406`; `infra/relay/src/observability.ts:25` |
| **Android push `user_id` leak to FCM & Clerk deletion** | Raw Clerk `user_id` sent in every FCM push; no Clerk deletion webhook cleanup | Android `>= 2.4.0` receives `hashTelemetryId(\`${userId}:${deviceId}\`)` (`user_hash`); Clerk `user.deleted` webhook purges relay DB rows & tunnels | **IMPROVED (`#42459`, `#42285`, `#42430`)** — `infra/relay/src/agentActivity/fcmPayloads.ts:14-24`; `infra/relay/src/accounts/AccountDeletion.ts:106-260` |
| **In-app UI / config / CLI opt-out & DNT** | No UI toggle, no settings key, no CLI flag, `DO_NOT_TRACK` ignored | **Still NO UI toggle (`shareUsageData`), NO settings key, NO CLI flag, `DO_NOT_TRACK` ignored** | **UNCHANGED** — `packages/contracts/src/settings.ts:1842-2199`; `WelcomePage.tsx:346-383`; `SettingsPanels.tsx:2584-2601`; `cli/config.ts:33-248` |
| **Unsalted hashing & PostHog `$geoip_disable`/`$ip`** | Unsalted SHA-256; omits `$geoip_disable`/`$ip`; closed-source `ingest.t3.codes` | **Unchanged** (unsalted SHA-256; no `$geoip_disable`/`$ip`; closed-source `ingest.t3.codes`) | **UNCHANGED** — `packages/shared/src/telemetryId.ts:8-12`; `AnalyticsService.ts:70-75, 205-249` |

---

## 1. Default State & Opt-Out Mechanism

### Default State: **ON by default** across all first-party telemetry channels
- **Server/Desktop Product Analytics (`ingest.t3.codes` / PostHog):** Defaults to `true` via `Config.Boolean("T3CODE_TELEMETRY_ENABLED").pipe(Config.withDefault(true))` (`apps/server/src/telemetry/AnalyticsService.ts:76`). Official releases set `T3CODE_TELEMETRY_CHANNEL` to `stable`, `nightly`, or `preview`, while local/fork builds default to `"dev"` (`.github/workflows/release.yml:601`; `apps/server/src/config.ts:27-28`).
- **Relay Connection Tracing (Axiom OTLP):** Enabled by default when built with relay tracing credentials (`isRelayTracingEnabled` returns `true` unless opted out, `packages/shared/src/otelEnvironment.ts:321-332`).
- **Mobile Crash Reporting (`expo-observe`):** Defaults to **ON** (`preferences.crashReportsEnabled !== false` in `apps/mobile/src/features/observability/observe.ts:56`, `apps/mobile/src/features/diagnostics/SettingsDiagnosticsRouteScreen.tsx:170`, `apps/mobile/src/persistence/mobile-preferences.ts:157-158`).
- **Mobile Relay Connection Traces:** Defaults to **ON** (`preferences.connectionTracesEnabled !== false` in `apps/mobile/src/features/observability/diagnosticsSetting.ts:15`, `apps/mobile/src/features/observability/ConnectionTracesSwitch.tsx:44`, `apps/mobile/src/persistence/mobile-preferences.ts:159-160`).
- **Opt-In Channels (OFF by default):**
  - **Agent Feedback (`agent.feedback`):** `agentFeedback.mode` defaults to `"off"` (`docs/user/telemetry.md:23`; `apps/server/src/feedback/FeedbackStore.ts:94-98`).
  - **Publish Agent Activity (Live Activities / Push):** Disabled until explicitly enabled (`apps/server/src/relay/AgentAwarenessRelay.ts:196-204`).
  - **Custom OTLP Diagnostics Export:** Unset by default (`apps/server/src/cli/config.ts:458-491`; `packages/shared/src/otelEnvironment.ts:344-379`), though **local NDJSON trace files** (`server.trace.ndjson` and `desktop.trace.ndjson`) are always written locally (`apps/server/src/config.ts:172`; `apps/desktop/src/app/DesktopObservability.ts:652-688`).

### Available Opt-Out Controls
1. **No In-App UI Toggle, Config Setting, or CLI Flag for Product Analytics:**
   - `packages/contracts/src/settings.ts:1842-2199` (`ServerSettingsPatch` and `ClientSettingsPatch`) contains **zero** telemetry opt-out keys (e.g., no `shareUsageData` setting; only `enableProviderUpdateChecks` at `:1895` and `agentFeedback` at `:1894`).
   - `apps/web/src/components/onboarding/WelcomePage.tsx:346-383` (`TelemetryNotice`) and `apps/web/src/components/settings/SettingsPanels.tsx:2584-2601` (`AboutSettingsPanel`) render only a text notice linking out to `https://t3.codes/privacy-policy` (`PRIVACY_POLICY_URL`) with no toggle and without naming `T3CODE_TELEMETRY_ENABLED=false`.
   - `apps/server/src/cli/config.ts:33-248` defines no `--no-telemetry` or `--disable-telemetry` CLI flag.
2. **`DO_NOT_TRACK` Ignored:** Zero occurrences of `DO_NOT_TRACK` exist anywhere in `apps/`, `packages/`, or `infra/`. `apps/marketing/src/pages/privacy-policy.astro:435` explicitly states: *"We currently do not respond to browser 'Do Not Track' signals."*
3. **Environment Variables (Server & Desktop):**
   - `T3CODE_TELEMETRY_ENABLED=false` disables both `AnalyticsService` (`apps/server/src/telemetry/AnalyticsService.ts:76, 134, 138-149`) and server/desktop/local-browser relay tracing (`packages/shared/src/otelEnvironment.ts:321-332`; `apps/server/src/cloud/relayTracing.ts:21-27`; `apps/desktop/src/ipc/methods/window.ts:103-125`; `apps/server/src/ws.ts:1952-1955`).
   - `T3CODE_OTEL_SDK_DISABLED=true` / `OTEL_SDK_DISABLED=true` stops OpenTelemetry export and relay tracing (`packages/shared/src/otelEnvironment.ts:254-269, 285, 327`), **but does NOT disable `AnalyticsService` (`ingest.t3.codes`)** because `AnalyticsService.ts` never reads OTEL disable flags.
   - Desktop shell discovery (`#42376`, `#42464`) now reads `T3CODE_TELEMETRY_ENABLED` from macOS/Linux login shells (`LOGIN_SHELL_ENV_NAMES`, `apps/desktop/src/shell/DesktopShellEnvironment.ts:125, 564`) and Windows PowerShell/User/Machine env (`WINDOWS_PROFILE_ENV_NAMES`, `:136, 462-464`; `packages/shared/src/shell.ts:305-323`). `AnalyticsService` (`AnalyticsService.ts:128-149`) and `relayTracing.ts` (`apps/server/src/cloud/relayTracing.ts:26, 43`; `apps/desktop/src/ipc/methods/window.ts:117-121`) both wait until `T3CODE_DESKTOP_SHELL_DISCOVERY_PENDING` clears before enabling export.
4. **Enterprise Admin Managed Policy (`limits.telemetry: false`):**
   - Supported via OS admin paths (`/Library/Application Support/T3Code/policy.json`, `/etc/t3code/policy.json`, `C:\Program Files\T3Code\policy.json` — `packages/shared/src/managedPolicyFile.ts:36-47`). Disables server analytics (`AnalyticsService.ts:130-131`), server relay tracing (`apps/server/src/cloud/relayTracing.ts:25, 40`), Desktop IPC relay tracing (`apps/desktop/src/ipc/methods/window.ts:107-110`), and local-server browser relay tracing (`apps/server/src/ws.ts:1952-1955`). Does not control hosted `app.t3.codes` or mobile (`docs/user/managed-policy.md:265-267`).
5. **Mobile UI Switches:**
   - **Send crash reports** (`crashReportsEnabled`) in **Settings > About T3 Code > Diagnostics** (`apps/mobile/src/features/diagnostics/SettingsDiagnosticsRouteScreen.tsx:162-211`).
   - **Send connection traces** (`connectionTracesEnabled`) in **Settings > About T3 Code > Diagnostics** (`SettingsDiagnosticsRouteScreen.tsx:213-228`) and the **T3 Connect setup sheet** (`apps/mobile/src/features/cloud/ConnectOnboardingRouteScreen.tsx:106-117`) via `ConnectionTracesSwitch.tsx:36-63`.

### Remaining Opt-Out Bypasses & Gaps
1. **Desktop shell/Windows scraping drops `T3CODE_OTEL_SDK_DISABLED` and `OTEL_SDK_DISABLED`:** `apps/desktop/src/shell/DesktopShellEnvironment.ts:102-137, 549-571, 660-674` omits both OTEL opt-out env vars from `LOGIN_SHELL_ENV_NAMES`, `WINDOWS_PROFILE_ENV_NAMES`, and `CACHED_ENV_NAMES`. Users setting `OTEL_SDK_DISABLED=true` or `T3CODE_OTEL_SDK_DISABLED=true` in `~/.zshrc` or Windows user env still export Desktop relay traces when launched from the OS GUI.
2. **Hosted web app (`app.t3.codes`) relay tracing has NO user opt-out:** `apps/web/src/cloud/publicConfig.ts:73-80` falls back to `(isHostedStaticApp() ? true : null)` when `serverRelayTelemetry` is `null` (confirmed in `apps/web/src/lib/runtime.test.ts:60-65` and `docs/user/telemetry.md:86`). There is no UI switch, storage flag, or DNT check on `app.t3.codes`.
3. **Unconditional `anonymous-id` creation & `getServerOwners` read when disabled:** See Sections 2 & 3 below.

---

## 2. External Auth / Local Non-Workspace File Reads

### Telemetry Identity Delta (`#42299`)
- **Removed:** In Baseline (`4c2e2b37`), `apps/server/src/telemetry/Identify.ts` read `~/.codex/auth.json` (extracting `tokens.account_id` and decoding `tokens.id_token` JWT `user_id`) and `~/.claude.json` (extracting `userID` and `oauthAccount.accountUuid`) twice on every server startup, and `apps/server/src/provider/CodexChatGptAuth.ts` eagerly recorded hashed OpenAI `user_id` owners. In Live `HEAD`, **all Codex/Claude file reads and `openai`/`claude` owner sources have been completely removed from `Identify.ts` and `CodexChatGptAuth.ts`** (`apps/server/src/telemetry/Identify.ts:18, 135-164`; `apps/server/src/provider/CodexChatGptAuth.ts:123-166`).
- **Remaining Unconditional Server Secret Read When Telemetry Is Disabled:**
  - `apps/server/src/serverRuntimeStartup.ts:178` executes `yield* analytics.recordOwners(yield* getServerOwners, "server");` unconditionally at boot. Because `yield* getServerOwners` evaluates before `analytics.recordOwners` / `analytics.record`, `Identify.ts:151-164` STILL reads `CLOUD_LINKED_USER_ID` from `ServerSecretStore` and computes `hashTelemetryId(connectUser.value)` on every boot even when telemetry is disabled (after which `AnalyticsService.record` at `AnalyticsService.ts:331` drops the event).

### Non-Telemetry External Auth & Host Reads (Functional Features)
- **`~/.claude/.credentials.json` & `~/.claude.json` (Usage Reset Credits):** `apps/server/src/provider/claudeResetCredits.ts:194-201, 243-249` reads `claudeAiOauth.accessToken` from `.credentials.json` (on non-macOS hosts) and `oauthAccount.organizationUuid` from `.claude.json` solely for querying/claiming Claude quota reset credits (`https://api.anthropic.com/api/oauth/usage`, `:215-218`), not for telemetry.
- **`~/.cursor/sandbox.json`:** `packages/provider-cursor/src/server/textGeneration.ts:239` checks existence before running Cursor text generation.
- **Desktop Browser Cookie Databases & OS Keychains:** On explicit user action in Settings -> Integrations (unless blocked by Managed Policy `limits.browserCookieImport: false`), `apps/desktop/src/preview/BrowserImport/Sources.ts:83-210` and `BrowserImport.ts:233-295` read SQLite/cookie stores from Chrome, Chromium, Edge, Brave, Vivaldi, Opera, Arc, Helium, Safari, Firefox, and Zen, and decrypt them via macOS Keychain, Linux `libsecret` (`apps/desktop/src/preview/BrowserImport/LinuxBrowserSecret.ts:13-38`, `t3-browser-secret`), or Windows DPAPI.
- **Desktop Shell Profile & Power/Process Inspection:** `apps/desktop/src/shell/DesktopShellEnvironment.ts:363-428` spawns interactive login shells (`-ilc`) or PowerShell at startup to read allowlisted env vars; `apps/desktop/src/telemetry/DesktopTelemetryPublisher.ts:29-55` monitors local lock/idle/suspend/battery/thermal state (local IPC only); `apps/server/src/resourceTelemetry/ProcessGroupMetrics.ts:14-146` samples process groups every minute only when user-configured `otlpMetricsUrl` is set (`:103`).

---

## 3. Persistent Device / Install IDs & Hashing

### Persistent Identifiers Generated
1. **Server / Desktop Install ID (`<stateDir>/anonymous-id`):**
   - Path: `anonymousIdPath: join(stateDir, "anonymous-id")` (`~/.t3/userdata/anonymous-id` or `<baseDir>/userdata/anonymous-id`, `apps/server/src/config.ts:176`).
   - Creation: `upsertAnonymousId` (`apps/server/src/telemetry/Identify.ts:103-133`) generates a random UUIDv4 and persists it atomically.
   - **Created & Read Even When Telemetry Is Disabled:** `AnalyticsService.make` (`apps/server/src/telemetry/AnalyticsService.ts:152`) calls `const { distinctId, installId } = yield* getTelemetryIdentity;` **unconditionally at layer initialization**, before `telemetryEnabled` (`:133-149`) is checked inside `sendBatch` (`:208`) and `record` (`:331`). Disabling `T3CODE_TELEMETRY_ENABLED=false` or setting `limits.telemetry: false` never prevents creation of or deletes `<stateDir>/anonymous-id`.
   - Usage: `distinctId` and `installId` both carry `hashTelemetryId(anonymousId)` (`Identify.ts:140-144`; `AnalyticsService.ts:215, 225`), attached to every non-anonymous product event.
2. **Web Browser Device ID (`localStorage`):**
   - `apps/web/src/connection/platform.ts:352-365` persists a random UUID under `localStorage` key `"t3code:analytics-device-id"`.
   - `packages/client-runtime/src/authorization/remote.ts:55-63, 108-114` hashes it (`hashTelemetryId`) and appends `?clientDevice=<sha256>` (alongside `?clientOwner=<sha256>` if signed into Connect) to WebSocket upgrades, which `apps/server/src/ws.ts:885-901, 3855` turns into `install.owner_observed` (`ownerSource: "device"` or `"connect"`).
3. **Mobile Analytics Device ID vs. Relay Device ID (`MobileSecureStorage`) — Delta `#42394`:**
   - `apps/mobile/src/persistence/mobile-storage.ts:11-12, 146, 193-194` now maintains **two separate persistent UUIDs** in secure storage: `"t3code.analytics.device-id"` (`loadOrCreateAnalyticsDeviceId`) and `"t3code.agent-awareness.device-id"` (`loadOrCreateAgentAwarenessDeviceId`).
   - `apps/mobile/src/connection/platform.ts:193-214` sends the hashed `analytics.device-id` as `AnalyticsDevice` (`clientDevice`) so analytics cannot correlate with the relay's push `RelayDeviceIdentity`.
   - Separately, `expo-observe` sends its own random install ID (`docs/user/telemetry.md:92`), and mobile relay tracing uses `Constants.sessionId` as `service.instance.id` (`apps/mobile/src/features/observability/tracing.ts:56`).
4. **Browser Page-Load Instance ID:** `apps/web/src/observability/clientTracer.ts:7` generates an ephemeral per-page-load `randomUUID()` for `service.instance.id`.

### Hashing (`packages/shared/src/telemetryId.ts`)
- **`hashTelemetryId`** (`packages/shared/src/telemetryId.ts:8-9`):
  ```ts
  export const hashTelemetryId = (value: string): string =>
    Hex.encode(sha256(new TextEncoder().encode(value)));
  ```
- **Still Unsalted SHA-256:** Zero per-install salt, secret pepper, or domain prefix. Anyone possessing a raw Clerk Connect `userId`, `anonymous-id`, or device UUID can precompute `SHA-256(raw_id)` and match it against `distinct_id` / `installId` / `ownerId`.
- **Android Push Recipient Hash (`#42459`):** `infra/relay/src/agentActivity/fcmPayloads.ts:14-24` computes `user_hash: hashTelemetryId(\`${userId}:${deviceId}\`)` (unsalted SHA-256 of `${userId}:${deviceId}`) for Android versions `>= 2.4.0` instead of sending raw `user_id` to FCM (builds `< 2.4.0` still receive raw `user_id`).

---

## 4. Exact Outbound Telemetry / Analytics / Crash / OTEL / Relay / Update Endpoints

| Category | Exact URL / Host | Trigger & Opt-Out Behavior | Verified Source (`file:line`) |
|---|---|---|---|
| **Product Usage Analytics Proxy** | `https://ingest.t3.codes/batch/` | Default `T3CODE_POSTHOG_HOST` + `/batch/`, using key `phc_XOWci4oZP4VvLiEyrFqkFjP4CZn55mjYYBMREK5Wd6m`. Closed-source server (archives batches, forwards subset to PostHog). Retries up to `MAX_BATCH_RETRY_MS = 30 * 60_000` (30m). Stopped by `T3CODE_TELEMETRY_ENABLED=false` or Managed Policy `limits.telemetry: false`. | `apps/server/src/telemetry/AnalyticsService.ts:53, 70-76, 211, 243` |
| **Relay Client OTLP Tracing** | `https://api.axiom.co/v1/traces` (mobile default) & CI-injected `T3CODE_RELAY_CLIENT_OTLP_TRACES_URL` / `VITE_RELAY_OTLP_TRACES_URL` (Axiom dataset `t3-code-relay-traces-*`) | Server, Desktop, Web, and Mobile `RelayTracing` export `relay.connection.attempt` / relay client spans. Server/Desktop/local-browser stopped by `T3CODE_TELEMETRY_ENABLED=false`, `T3CODE_OTEL_SDK_DISABLED=true`, `OTEL_SDK_DISABLED=true`, or `limits.telemetry: false` (subject to Desktop shell forwarding gap for `OTEL_SDK_DISABLED`). Mobile stopped by `Send connection traces` switch (`#42415`). Hosted `app.t3.codes` has **no opt-out**. | `apps/mobile/app.config.ts:487-491`; `apps/server/src/cloud/relayTracing.ts:15, 21-46`; `apps/web/src/cloud/publicConfig.ts:54-95`; `.github/workflows/release.yml:460-462, 1271-1273`; `infra/relay/src/observability.ts:33-63` |
| **Mobile Crash Reporting (`expo-observe`)** | `<T3CODE_RELAY_URL>/v1/mobile/observe` (default `https://relay.t3.codes/v1/mobile/observe`, fallback `https://observe.invalid`) -> Axiom (`t3-code-mobile-logs`, `t3-code-mobile-metrics`) | Enabled by default (`crashReportsEnabled !== false`); forwards crashes, JS errors, stack traces (symbolicated against R2 source maps), and startup timings to Axiom (30-day retention). Stopped via **Send crash reports** switch in Mobile Diagnostics. | `apps/mobile/app.config.ts:24-26, 475-477, 494`; `apps/mobile/src/features/observability/observe.ts:8, 30-62`; `infra/relay/src/mobileTelemetry/MobileTelemetry.ts:231-324`; `infra/relay/src/observability.ts:65-88` |
| **T3 Connect Relay / Hosted Web** | `https://relay.t3.codes`, `https://app.t3.codes`, `https://latest.app.t3.codes`, `https://nightly.app.t3.codes`, `https://t3.codes` | T3 Connect tunnel/auth, Clerk authentication, hosted static web app, agent activity notifications (opt-in **Publish agent activity** -> Apple APNs / Google FCM `fcm.googleapis.com`), install scripts. | `packages/shared/src/connectAuth.ts:10`; `apps/web/vercel.ts:3-6`; `apps/server/src/cli/install.ts:57-58`; `apps/server/src/relay/AgentAwarenessRelay.ts:269-291`; `infra/relay/src/agentActivity/FcmClient.ts:125` |
| **User-Configured OTLP Export** | User-configured `T3CODE_OTLP_*_URL`, `OTEL_EXPORTER_OTLP_*_ENDPOINT`, or Settings -> Maintenance -> Diagnostics | Disabled by default unless user/admin configures an endpoint. Includes `host.name` resource attribute (`ServerConfig.otlpResource` / `DesktopObservability`). | `apps/server/src/cli/config.ts:458-491`; `apps/server/src/config.ts:125-140`; `apps/desktop/src/app/DesktopObservability.ts:610-648` |
| **Passive Third-Party Favicon Fetch** | `https://www.google.com/s2/favicons?domain=<host>&sz=<size>` | Automatically queried by browser/Desktop UI whenever public external hostnames render in markdown links or preview tabs (excludes RFC6762 private hosts/IPs, sends all public domains). Ignores all telemetry flags. | `packages/shared/src/favicon.ts:74-84`; `packages/shared/src/hostClassification.ts:121-152`; `apps/web/src/components/chat/MarkdownExternalLink.tsx:25`; `apps/web/src/components/RightPanelTabs.tsx:648` |
| **Model Manifest Auto-Fetch** | `https://raw.githubusercontent.com/pingdotgg/t3code/main/apps/server/src/provider/model-manifest.json` | Refreshed on demand / 1h TTL (`5m` retry). Disabled ONLY by `enableProviderUpdateChecks: false` (ignores `T3CODE_TELEMETRY_ENABLED`). | `apps/server/src/provider/ModelManifest.ts:28-38, 251-263` |
| **LiteLLM Pricing Auto-Fetch** | `https://raw.githubusercontent.com/BerriAI/litellm/main/model_prices_and_context_window.json` (fallback `https://api.github.com/repos/BerriAI/litellm/contents/model_prices_and_context_window.json?ref=main`) | Fetched when opening/refreshing Usage with 24h TTL. Ignores `enableProviderUpdateChecks` and `T3CODE_TELEMETRY_ENABLED`. | `apps/server/src/usage/UsageService.ts:72-77, 300-325` |
| **Desktop & CLI Auto-Updates** | `https://api.github.com/repos/pingdotgg/t3code/releases`, `https://github.com/pingdotgg/t3code/releases/download` (or Managed Policy `updates.feedUrl` / `T3CODE_RELEASE_BASE_URL`) | Desktop `electron-updater` polls every `4 hours` (`AUTO_UPDATE_POLL_INTERVAL`), disabled only by Managed Policy `updates.mode: "off"`. | `apps/desktop/src/updates/DesktopUpdates.ts:65-67, 1345-1350`; `apps/server/src/cli/update.ts:75-108`; `packages/shared/src/cliRelease.ts:7-45, 135-136`; `docs/user/managed-policy.md:289-290` |
| **Cloudflare Tunnel Reachability Probes** | `https://1.1.1.1/dns-query`, `region1.v2.argotunnel.com:7844`, `region2.v2.argotunnel.com:7844` | T3 Connect reachability check every 5 minutes when managed tunnel is configured. | `apps/server/src/cloud/ConnectReachability.ts:30-42` |

---

## 5. Exact Data & Properties Collected

### A. Product Usage Analytics (`https://ingest.t3.codes/batch/`)
**Common Envelope (`AnalyticsService.ts:210-241`):**
- Standard events: `api_key`, `uuid` (UUIDv7), `distinct_id` (= `installId` = `SHA-256(anonymous-id)`), `installId`, `$process_person_profile: false`, `timestamp`, `platform`, `wsl`, `arch`, `t3CodeVersion`, `telemetryChannel`, `clientType` (`"desktop-app"` or `"cli-web-client"`, `:167`), `serverOs`, `serverArch`, `serverWslDistro`, `serverAppVersion`, `serverMode` (`"desktop"` or `"web"`).
- Anonymous events (`anonymous: true`, isolated in single-event requests by `takeBatch` at `:251-258`): `distinct_id` = `event.uuid`, omits `installId` and all host/OS/arch/mode metadata (retains only `t3CodeVersion`, `telemetryChannel`, `$process_person_profile: false`).
- **IP & GeoIP Suppression:** Client **never** sets `$geoip_disable: true` or `$ip: null` (confirmed negative `rg`). Closed-source `ingest.t3.codes` handles raw requests before forwarding to PostHog.

**Event Catalog:**
1. `server.boot.heartbeat` (`serverRuntimeStartup.ts:174-177`): `threadCount`, `projectCount`.
2. `install.owner_observed` (`AnalyticsService.ts:353-374`; callers `serverRuntimeStartup.ts:178`, `ws.ts:3855`): `ownerSource` (`"connect" | "device"` — `openai` and `claude` removed in `#42299`), `ownerId` (unsalted SHA-256), `observedBy` (`"server" | "client"`).
3. `client.thread.started` (`ws.ts:837-879, 2306-2310`): `surface`, `appVersion`/`clientAppVersion`, `clientOs`, `os`, `clientDeviceType`, `webDeployment`, `clientBrowser`, `osMajorVersion`/`clientOsMajorVersion`, `deviceModel`/`clientDeviceModel`, `connectionMethod`, `workspaceStrategy`.
4. `client.turn.requested` (`ws.ts:1496, 2315`): client origin properties (`readClientAnalyticsProps`).
5. `feature.used` (`FeatureUsage.ts:20-91`; `ws.ts:1500-1503, 3870-3875`): 18 orchestration commands + 17 WebSocket RPCs + guest share actions, with closed enum `variant` and `surface`.
6. `provider.turn.completed` & `provider.request.completed` (`ProviderUsageAnalytics.ts:174-206, 336-374, 415-427`): `reportVersion` (`2`), `provider` (driver kind), `model` & `observedModel` (known catalog slug or `custom:<driver>` per `#42465`), `mixedModels`, `effort`, `serviceTier`, `runtimeMode` (permission mode), `interactionMode`, `createdBy`, `creationSource`, `scheduled`, `subagent`, `terminalStatus`, `usageStatus`, `usageScope`, `hasSubagents`, `inputTokens`, `cachedInputTokens`, `cacheCreationTokens`, `outputTokens`, `reasoningTokens`, `otherInputTokens`, `otherOutputTokens`, `nativeChildTurns`, `childUsageStatus`, `childInputTokens`, `childOutputTokens`, `childrenRunning`, `providerTurns`, `continuations`, `steeringRestarts`, `durationMs`.
7. `mcp.tool.invoked` (`McpHttpServer.ts:1165-1189`; `ProviderDimensions.ts:147-281`): `tool`, `callerProvider`, `callerModel` (masked `custom:<driver>` if custom), `callerOrigin`, `callerDepth` (max 5), `durationMs`, `outcome` (`"ok" | "error"`), `errorCode` (closed regex/tag per `#42397`), closed `handoffSettings`, `targetProvider`, `targetModel`, `targetRuntimeMode`, `targetInteractionMode`, `crossProvider`.
8. `mcp.delegated_task.finished` (`DelegatedTaskAnalytics.ts:62-71`): `status`, `callerProvider`, `callerModel`, `targetProvider`, `targetModel`, `crossProvider`, `completionWake`, `durationSeconds`.
9. `chatgpt.auth.started` / `completed` & `chatgpt.transfer.started` / `completed` (`CodexChatGptAuth.ts:135-162`): `flow`, `intent`, `callbackMode`, `outcome`, `durationMs`, `failureStage`, `connectedAccountCount`, `connectedConnectionCount`, `savedConnectionCount`.
10. `agent.feedback` (`FeedbackStore.ts:208-220`, **opt-in `off` by default**): sent `{ anonymous: true }` in its own request without `installId`; `kind`, `subject`, `expected`/`actual`/`workaround` (only when `userRequested`), `trace` (scrubbed turn outline + optional user-approved conversation), `userRequested`, `source`, `provider`, `model`.

### B. Relay Client & Relay Worker Telemetry (Axiom, 30-day retention)
- **Client Relay Spans (`packages/shared/src/relayTracing.ts`, `supervisor.ts:567-595`):** `relay.connection.attempt` sends `service.name`, `service.version`, `service.runtime`, random `service.instance.id`, `t3code.client.surface`, `environment.id`, `environment.target.kind`, attempt/retry counters, timings, error names (`traceSafeError` strips messages & stack frames; `nonInterferingTracer` strips `url.full`/`url.query`/`server.address`/`server.port`; operator `OTEL_RESOURCE_ATTRIBUTES` stripped).
- **Relay Worker Spans (`infra/relay/src/http/Api.ts:213-265, 328-339, 394-406`):** Strips `remoteAddress` and `user-agent` from span copies (`#42380`), **but still records** Cloudflare `relay.edge.colo`, `relay.edge.country`, `relay.edge.continent` (`:218-221`), `relay.environment_id` (`:369`), `relay.auth.mode`, **`relay.auth.subject`** (raw Clerk user ID), and **`user.id`** (raw Clerk user ID, `:334, 399`, queried in `RelayRecentSpansView`, `infra/relay/src/observability.ts:25`).
- **Mobile `expo-observe` (`infra/relay/src/mobileTelemetry/MobileTelemetry.ts:252-324`):** Rate-limited on `cf-connecting-ip` (`:266`, not exported to spans); forwards JS errors/native crashes, stack traces (symbolicated on Relay from R2), startup timings, device model, OS version, app version, and random install ID to Axiom (`t3-code-mobile-logs`, `t3-code-mobile-metrics`).

---

## 6. Docs / Privacy Policy vs Code Discrepancies

Note: `docs/user/telemetry.md` and `docs/internals/08_product-analytics.md` were substantially updated alongside `#42299`, `#42465`, `#42398`/`#42466`, `#42356`, and `#42415` and now accurately describe `ingest.t3.codes`, `custom:<driver>` model masking, removal of Codex/Claude account IDs, isolated anonymous `agent.feedback`, and mobile's separate switches. **However, material discrepancies remain:**

1. **`"anonymous"` framing in UI & Privacy Policy vs. identity/device linking & raw Clerk IDs:**
   - `WelcomePage.tsx:353`, `SettingsPanels.tsx:2589`, and `apps/marketing/src/pages/privacy-policy.astro:85` still describe product telemetry as *"anonymous usage data"*. In reality, non-anonymous events attach persistent `installId` and `install.owner_observed` links installs to unsalted SHA-256 hashes of T3 Connect accounts (`connect`) and web/mobile devices (`device`), while Relay Worker Axiom spans record unhashed Clerk `user.id` (`infra/relay/src/http/Api.ts:334, 399`) and Cloudflare `relay.edge.colo`/`country`/`continent` (`:218-221`).
2. **Undisclosed unconditional `anonymous-id` creation & `getServerOwners` execution when opted out:**
   - Neither `docs/user/telemetry.md` nor `privacy-policy.astro` discloses that setting `T3CODE_TELEMETRY_ENABLED=false` or `limits.telemetry: false` still generates/persists `<stateDir>/anonymous-id` (`AnalyticsService.ts:152`) and still reads/hashes `CLOUD_LINKED_USER_ID` at startup (`serverRuntimeStartup.ts:178`).
3. **Desktop `T3CODE_OTEL_SDK_DISABLED` / `OTEL_SDK_DISABLED` doc vs. implementation gap:**
   - `docs/user/telemetry.md:75-76` states relay tracing stops when `T3CODE_OTEL_SDK_DISABLED` or `OTEL_SDK_DISABLED` turns OTEL export off, without noting that Desktop GUI launches ignore both variables if placed in shell profiles (`~/.zshrc`) or Windows user env vars (`apps/desktop/src/shell/DesktopShellEnvironment.ts:102-137`).
4. **Outdated `privacy-policy.astro` (`effectiveDate="2026-10-06"`) omits passive/background third-party fetches:**
   - Does not mention passive `www.google.com/s2/favicons` lookups on public rendered domains (`packages/shared/src/favicon.ts:81`), background `raw.githubusercontent.com` fetches for `model-manifest.json` (`ModelManifest.ts:28-38`) and LiteLLM pricing (`UsageService.ts:72-77`, ignoring both `enableProviderUpdateChecks` and `T3CODE_TELEMETRY_ENABLED`), or Cloudflare edge colo/country logging on Axiom spans.
5. **Onboarding UI omits opt-out mechanism:**
   - `WelcomePage.tsx:353` tells users *"To opt out, see the privacy policy"* rather than providing a toggle or naming `T3CODE_TELEMETRY_ENABLED=false`.

---

## 7. Zero-Telemetry / Negative-Check Verification

To verify the absence of hidden third-party analytics/crash SDKs and verify opt-out flag coverage across Live `HEAD`, read-only `rg -n -w` checks were executed across `apps/`, `packages/shared/`, `packages/contracts/`, `packages/client-runtime/`, and `infra/`:

- **Browser/Third-Party Analytics & Crash SDKs (`posthog-js`, `@posthog`, `sentry`, `amplitude`, `mixpanel`, `bugsnag`, `rollbar`, `datadog`, `gtag`, `plausible`, `fathom`, `umami`):** **0 telemetry SDK hits** across first-party source code (only false-positive string matches in `packages/issue-tracker-linear/schema/linear.graphql` GraphQL schema fields, CSS `scrollbar`, audio `amplitude`, and a Claude test transcript fixture). Product analytics uses a custom `HttpClient` POST to `https://ingest.t3.codes/batch/` (`AnalyticsService.ts:74, 243`), and mobile crash reporting uses `expo-observe` routed through `<T3CODE_RELAY_URL>/v1/mobile/observe` (`apps/mobile/app.config.ts:24-26`).
- **Privacy Suppression Signals (`DO_NOT_TRACK`, `$geoip_disable`, `$ip`):** **0 hits** across all of `apps/`, `packages/`, and `infra/`. Confirmed T3 Code does not honor `DO_NOT_TRACK` and does not include PostHog GeoIP/IP suppression fields.
- **External Auth Telemetry Reads (`.codex/auth.json`, `tokens.account_id`, `openAiUserId`):** **0 hits** in `apps/server/src/telemetry/` or `apps/server/src/provider/CodexChatGptAuth.ts` (remaining `.credentials.json`/`.claude.json` reads in `apps/server/src/provider/claudeResetCredits.ts:194-250` are strictly for Anthropic usage-limit reset credits).
- **Clean/Positive Baseline Controls Verified:** Local-server browser relay tracing respects server opt-out (`apps/web/src/lib/runtime.test.ts:43-58`), Desktop cold-start race waits on shell discovery (`apps/server/src/cloud/relayTracing.ts:26`), custom/unknown models mask to `custom:<driver>` (`ProviderDimensions.ts:71`), `agent.feedback` sends anonymously in isolated requests (`FeedbackStore.ts:219`; `AnalyticsService.ts:251-258`), and Clerk deletion webhooks clean up relay account state (`AccountDeletion.ts:106-260`).

---

```json
{
  "slug": "t3code",
  "platform": "T3 Code",
  "group": "A",
  "repo": "pingdotgg/t3code",
  "sha": "2b84f4862ce7320bdff28cef88d3ed8d0f23797e",
  "commit_date": "2026-11-24T20:51:11-08:00",
  "license": "MIT",
  "status": "Active",
  "default_state": "ON by default for server/desktop product usage analytics (ingest.t3.codes), relay connection tracing (Axiom), mobile crash reporting (expo-observe -> Relay -> Axiom), and mobile connection tracing; OFF by default for agent.feedback, Publish agent activity, and user OTLP export",
  "opt_out_controls": [
    "Env var T3CODE_TELEMETRY_ENABLED=false (disables server/desktop/local-browser product analytics & relay tracing; read from macOS/Linux login shell and Windows User/Machine scopes)",
    "Env vars T3CODE_OTEL_SDK_DISABLED=true / OTEL_SDK_DISABLED=true (disables OTLP/relay tracing only; NOT forwarded by Desktop shell/Windows probes; does NOT stop ingest.t3.codes)",
    "Admin Managed Policy limits.telemetry: false (/etc/t3code/policy.json, /Library/Application Support/T3Code/policy.json, C:\\Program Files\\T3Code\\policy.json)",
    "Mobile UI switches (post-install opt-out): Send crash reports and Send connection traces in Settings > About T3 Code > Diagnostics (and Connect setup sheet for traces)",
    "NO in-app UI toggle, NO settings JSON key, and NO CLI flag for product analytics"
  ],
  "do_not_track": false,
  "opt_out_bypasses": [
    "AnalyticsService.make (AnalyticsService.ts:152) unconditionally executes getTelemetryIdentity -> upsertAnonymousId at startup even when T3CODE_TELEMETRY_ENABLED=false or limits.telemetry: false, creating/reading <stateDir>/anonymous-id",
    "serverRuntimeStartup.ts:178 eagerly evaluates getServerOwners at startup even when telemetry is disabled, reading/hashing CLOUD_LINKED_USER_ID",
    "DesktopShellEnvironment.ts:102-137 omits T3CODE_OTEL_SDK_DISABLED and OTEL_SDK_DISABLED from LOGIN_SHELL_ENV_NAMES, WINDOWS_PROFILE_ENV_NAMES, and CACHED_ENV_NAMES",
    "Hosted web app app.t3.codes relay tracing (apps/web/src/cloud/publicConfig.ts:79) unconditionally returns true when no primary server overrides it, with no user opt-out",
    "LiteLLM pricing fetch (UsageService.ts:300-325) and passive Google Favicon fetches (favicon.ts:81) ignore T3CODE_TELEMETRY_ENABLED and enableProviderUpdateChecks"
  ],
  "persistent_ids": [
    "<stateDir>/anonymous-id (UUIDv4, created/read unconditionally even when disabled, hashed as distinct_id & installId)",
    "Browser localStorage key t3code:analytics-device-id (UUIDv4, hashed as clientDevice)",
    "Mobile secure storage t3code.analytics.device-id (UUIDv4, hashed as clientDevice, separated from t3code.agent-awareness.device-id)",
    "expo-observe random install ID (mobile crashes/metrics)",
    "Browser per-page-load service.instance.id UUID (clientTracer.ts:7)"
  ],
  "hashing": "Unsalted SHA-256 hex (packages/shared/src/telemetryId.ts:8-9) for installId/distinct_id, connect/device ownerId, and Android FCM user_hash (sha256(userId:deviceId))",
  "external_auth_reads": "REMOVED from telemetry in #42299 (Identify.ts and CodexChatGptAuth.ts no longer read ~/.codex/auth.json or ~/.claude.json). Non-telemetry reads remain: ~/.claude/.credentials.json & ~/.claude.json in claudeResetCredits.ts (Anthropic usage reset credits), ~/.cursor/sandbox.json check in Cursor textGeneration.ts, and user-initiated BrowserImport cookie DBs/keychains",
  "telemetry_endpoints": [
    "https://ingest.t3.codes/batch/ (closed-source analytics proxy -> PostHog)",
    "https://api.axiom.co/v1/traces & build-injected Axiom OTLP URLs (relay tracing)",
    "https://relay.t3.codes/v1/mobile/observe (mobile expo-observe -> Axiom)",
    "https://www.google.com/s2/favicons?domain=... (passive public-domain favicon leak)",
    "https://raw.githubusercontent.com/pingdotgg/t3code/main/apps/server/src/provider/model-manifest.json (model catalog auto-fetch)",
    "https://raw.githubusercontent.com/BerriAI/litellm/main/model_prices_and_context_window.json & GitHub API fallback (LiteLLM pricing)",
    "https://api.github.com/repos/pingdotgg/t3code/releases (desktop/CLI auto-updates)"
  ],
  "ip_geo_handling": "AnalyticsService.ts omits PostHog $geoip_disable and $ip:null behind closed-source ingest.t3.codes; privacy-policy.astro:98-103 discloses IP logging. Relay Worker (infra/relay/src/http/Api.ts:213-265) now strips remoteAddress and user-agent from trace span requests (#42380) but STILL annotates Axiom spans with Cloudflare relay.edge.colo, relay.edge.country, relay.edge.continent, and raw Clerk user.id/relay.auth.subject; MobileTelemetry.ts:266 reads cf-connecting-ip for rate limiting",
  "custom_model_handling": "FIXED in #42465: modelLabel (ProviderDimensions.ts:66-72) maps only known bundled manifest or non-custom vendor catalog entries to catalog slugs and masks all other/custom/local/custom-endpoint/Bedrock models as custom:<driver>",
  "docs_discrepancies": [
    "WelcomePage.tsx:353, SettingsPanels.tsx:2589, and privacy-policy.astro:85 still label product telemetry 'anonymous' despite installId + Connect/device owner linking and raw Clerk user.id + Cloudflare edge geo on Relay Axiom spans",
    "Docs and privacy-policy.astro do not disclose unconditional <stateDir>/anonymous-id generation or getServerOwners reading when telemetry is disabled",
    "docs/user/telemetry.md:75-76 states T3CODE_OTEL_SDK_DISABLED/OTEL_SDK_DISABLED stop relay tracing without noting Desktop shell/Windows env discovery ignores both",
    "privacy-policy.astro (2026-10-06) omits Google favicon leaks, automatic GitHub model-manifest/LiteLLM fetches, and Cloudflare edge colo/country logging",
    "WelcomePage.tsx onboarding notice says 'To opt out, see the privacy policy' with no UI switch or inline T3CODE_TELEMETRY_ENABLED=false instruction"
  ],
  "privacy_rating": "D",
  "one_line_verdict": "Upgraded from Baseline F to D after #42299/#42465/#42464/#42376/#42415 removed Codex/Claude auth harvesting, masked custom models to custom:<driver>, fixed Desktop/Windows/local-browser relay opt-out races, and added mobile trace controls—yet remains D due to default-on tracking with no UI/config/CLI opt-out, unconditional anonymous-id creation when disabled, unsalted SHA-256 hashing, missing PostHog IP suppression behind closed-source ingest.t3.codes, hosted-web/Desktop-OTEL opt-out gaps, and Relay Axiom geo + raw Clerk user.id logging."
}
```
