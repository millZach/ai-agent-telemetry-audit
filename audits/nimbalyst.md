# Forensic Privacy & Telemetry Audit: Nimbalyst (`nimbalyst/nimbalyst`)

- **Slug**: `nimbalyst`
- **Platform**: `Nimbalyst`
- **Group**: `A`
- **Repository**: `https://github.com/nimbalyst/nimbalyst` (`nimbalyst/nimbalyst`)
- **Branch**: `main` (`iOS Release v1.3.1 (build 5)`)
- **Audited Commit SHA**: `044ff88a1ceb5b2d73686f777f8740ed0539bf1f`
- **Commit Date**: `2026-11-23T15:34:13-05:00`
- **License**: `MIT` (`LICENSING.md:3`, `package.json`)
- **Status**: **Active** — Official open-source successor to `stravu/crystal`; however, `LICENSING.md:5-8` states that the Nimbalyst collaboration sync server (the Cloudflare Worker powering `wss://sync.nimbalyst.com`) is a separate, non-included project.
- **Privacy Rating**: **`D` (Aggressive / Leaky)**

---

## 1. Default State & Opt-Out Mechanism

### Default State
Unlike Crystal's dual-client `minimalClient` design (which is completely absent in Nimbalyst—verified 0 `minimalClient` matches via `rg`), Nimbalyst replaces it with full PostHog instrumentation **enabled (opt-out) by default** across all three client apps (Electron Desktop, iOS, and Android):

1. **Desktop (Electron Main & Renderer)**:
   - `packages/electron/src/main/utils/store.ts:2193-2196` defaults `analyticsEnabled` to `true` (`getAppStore().get('analyticsEnabled', true)`), and `packages/electron/src/main/services/analytics/AnalyticsService.ts:408-415` creates an `<userData>/analytics-settings.json` store (`0o600` via `privateSettingsStore.ts:84-91`) defaulting `analyticsEnabled: true` and `analyticsId: 'nimbalyst_${ulid()}'`.
   - At module load/singleton instantiation (`AnalyticsService.ts:59-67, 78-82`), `AnalyticsService` unconditionally instantiates two `posthog-node` clients (`this.postHogClient` and `this.sessionTracker`) pointing to PostHog project `phc_s3lQIILexwlGHvxrMBqti355xUgkRocjMXW4LjV0ATw` (`AnalyticsService.ts:12, 418-431`).
   - In the renderer (`packages/electron/src/renderer/index.tsx:258-329`), `posthog-js` initializes unconditionally with `posthog.init(...)`, though outbound captures are gated fail-closed by `before_send` checking `isAnalyticsConsentGranted()` (`packages/electron/src/renderer/utils/analyticsConsent.ts:22, 26-28`, `renderer/index.tsx:301-302`) until IPC resolves the main setting (`renderer/index.tsx:321-329`).
2. **iOS**: `packages/ios/NimbalystNative/Sources/Analytics/AnalyticsManager.swift:22, 40` initializes `isEnabled = UserDefaults.standard.object(forKey: Self.analyticsEnabledKey) as? Bool ?? true` (ON by default) and immediately initializes `PostHogSDK.shared.setup(config)` on launch (`AnalyticsManager.swift:27-50`).
3. **Android**: `packages/android/app/src/main/java/com/nimbalyst/app/analytics/AnalyticsManager.kt:31, 63` initializes `isEnabled = preferences.getBoolean(KEY_ANALYTICS_ENABLED, true)` (ON by default) and immediately initializes `PostHogAndroid.setup(appContext, config)` in `NimbalystApplication.onCreate()` (`AnalyticsManager.kt:36, 72`).

### Opt-Out Controls
- **Desktop UI**: **Settings -> Advanced -> "Send Anonymous Usage Data"** (`packages/electron/src/renderer/components/GlobalSettings/panels/AdvancedPanel.tsx:467-472`) triggers IPC `analytics:set-enabled` (`packages/electron/src/main/ipc/SettingsHandlers.ts:781-783`) -> `applyAnalyticsEnabled()` (`packages/electron/src/main/services/analytics/applyAnalyticsEnabled.ts:17-28`), which calls `analytics.optIn()`/`optOut()`, persists `analyticsEnabled` in the main app store (`AnalyticsService.ts:155, 171`), and broadcasts `analytics:enabled-changed` to every open `BrowserWindow` (`applyAnalyticsEnabled.ts:34-42`, `renderer/index.tsx:331-340`).
- **Desktop MCP Tool**: `analytics_set_enabled` (`packages/electron/src/main/mcp/settingsServer.ts:137-142, 322-323`) invokes `SettingsControlService.setAnalytics()`, which also routes through `applyAnalyticsEnabled()`.
- **Desktop Environment Suppression**: `isAnalyticsSuppressedByEnv()` (`packages/electron/src/main/services/analytics/analyticsSuppression.ts:11-15`) suppresses both main (`AnalyticsService.ts:286, 428`) and renderer (via IPC `analytics:allowed`) when `NIMBALYST_DISABLE_ANALYTICS=1`, `PLAYWRIGHT=1`, or truthy `PLAYWRIGHT_TEST` is set.
- **iOS UI**: **Settings -> Privacy -> "Usage Analytics"** toggle (`packages/ios/NimbalystNative/Sources/Views/SettingsView.swift:292-299`) invokes `AnalyticsManager.shared.optIn()` / `optOut()`.
- **Android UI**: **Settings -> "Send anonymous usage data"** switch (`packages/android/app/src/main/java/com/nimbalyst/app/ui/SettingsScreen.kt:262-268`) invokes `AnalyticsManager.optIn()` / `optOut()`.
- **CLI Update Checks Only**: `NIMBALYST_WIKI_NO_UPDATE_CHECK` or `CI` disables `packages/cli` update checks (`packages/cli/src/mcp/updateCheck.ts:22, 236-239`).
- **Do Not Track (`DO_NOT_TRACK` / `DNT`)**: **Not honored** anywhere in Desktop, Mobile, or CLI (0 matches across `packages/`).

### Opt-Out Bypasses, Ordering Flaws & Fail-Open Paths
1. **iOS & Android Cold-Launch `identify()` + `$set` Before `optOut()` (Bypass)**:
   - On iOS (`AnalyticsManager.swift:52-64`), `initialize()` calls `PostHogSDK.shared.identify(id)` (`lines 53-55`) and, in `#if DEBUG`, `PostHogSDK.shared.capture("$set", properties: ["$set_once": ["is_dev_user": true]])` (`lines 58-60`) **before** checking `if !isEnabled { PostHogSDK.shared.optOut() }` (`lines 62-64`).
   - On Android (`AnalyticsManager.kt:74-87`), `initialize()` identically calls `PostHog.identify(distinctId)` (`line 75`) and, when `isDebugBuild()`, `PostHog.capture("\$set", ...)` (`lines 78-83`) **before** calling `if (!isEnabled) { PostHog.optOut() }` (`lines 85-87`).
   - Consequently, opted-out mobile installs invoke PostHog `$identify` (and debug `$set`) on every cold start prior to opting out the SDK.
2. **Mobile `setDistinctIdFromPairing()` and `setEmail()` Lack `isEnabled` Checks**:
   - `AnalyticsManager.swift:75-94` and `AnalyticsManager.kt:97-120` guard `alias()`, `identify()`, and `capture("$set", { email })` only on `initialized`, not `isEnabled` (relying solely on `PostHog.optOut()` state once initialization finishes).
3. **Final Telemetry Event Sent Upon Opting Out (Desktop, iOS, Android)**:
   - Desktop `AnalyticsService.optOut()` (`AnalyticsService.ts:166-169`) dispatches `captureImmediate({ distinctId, event: 'analytics_opt_out' })` immediately *before* `postHogClient.optOut()`.
   - iOS `optOut()` (`AnalyticsManager.swift:103-108`) sends `"mobile_analytics_opt_out"` *before* `PostHogSDK.shared.optOut()`.
   - Android `optOut()` (`AnalyticsManager.kt:133-139`) sends `"mobile_analytics_opt_out"` *before* `PostHog.optOut()`.
4. **Desktop Store Read Fails Open**:
   - Both `isAnalyticsEnabled()` (`store.ts:2197-2202`) and `AnalyticsService.allowedToSendAnalytics()` (`AnalyticsService.ts:291-296`) explicitly catch exceptions and return `true` (*"Fail open - if we can't read the setting, allow analytics"*). By contrast, `canEvaluateRemoteConfig()` (`AnalyticsService.ts:139-147`) and renderer `consentGranted` (`analyticsConsent.ts:22`) fail closed.
   - Note: `AnalyticsService.sessionTracker` (`AnalyticsService.ts:80`) never has `.optOut()` called on it during `optOut()`, though its sole call site `setSessionId()` (`AnalyticsService.ts:182-185`) checks `this.allowedToSendAnalytics()` and returns early (and its `before_send` checks `isAnalyticsSuppressedByEnv()`).
5. **Unconfigurable Desktop Auto-Updater Checks & Auto-Download**:
   - `AutoUpdaterService` (`packages/electron/src/main/services/autoUpdater.ts:73-74, 101, 437, 597, 602`) configures `autoDownload = true`, `autoInstallOnAppQuit = true`, and polls GitHub Releases (`nimbalyst/nimbalyst`) automatically with no opt-out toggle in `store.ts` or `AdvancedPanel.tsx`.

---

## 2. External Auth / Local Non-Workspace File Reads

Nimbalyst reads several external AI CLI credential and configuration stores outside workspace directories. Static inspection confirms these reads are used **solely for local CLI orchestration, quota meters, and user-initiated Cloudflare Sandbox provisioning**—they are **not** transmitted to PostHog or Nimbalyst's telemetry/sync endpoints:

1. **Claude Code OAuth Credentials & Settings**:
   - `packages/electron/src/main/services/ClaudeUsageService.ts:241-303`: On macOS, executes `security find-generic-password -s "<serviceName>" -w` (`lines 294-296`, matching `Claude Code-credentials` and config-dir-scoped keychain entries via `resolveClaudeKeychainServiceNames`) and falls back to reading `<CLAUDE_CONFIG_DIR or ~/.claude>/.credentials.json` (`lines 267-288`). Also reads `~/.claude/settings.json` via `ClaudeSettingsManager.getInstance().getUserLevelEnv()` (`ClaudeUsageService.ts:233`, `ClaudeSettingsManager.ts:7, 113, 475-483`). Extracted `claudeAiOauth.accessToken` is sent only to `https://api.anthropic.com/api/oauth/usage` with header `'anthropic-beta': 'oauth-2025-04-20'` (`ClaudeUsageService.ts:55, 332`).
   - `packages/electron/src/main/services/cloudflareSandbox/claudeCredentialExport.ts:53-55, 105-108`: Exports `claudeAiOauth.accessToken` into `/home/nimbalyst/.claude/.credentials.json` inside the user's own Cloudflare Sandbox container when provisioning sandbox execution.
2. **OpenAI Codex Auth & Local Session History**:
   - `packages/electron/src/main/services/CodexAuthService.ts:11, 109-117`: Interacts with the local `codex` binary (`~/.codex/auth.json` for ChatGPT OAuth / API key login state).
   - `packages/electron/src/main/services/CodexUsageService.ts:118`: Reads `$CODEX_HOME/sessions` or `~/.codex/sessions` locally to aggregate usage metrics.
3. **OpenCode Configuration**:
   - `packages/electron/src/main/services/OpenCodeConfigService.ts:8, 17-22, 43-45`: Reads/merges `$XDG_CONFIG_HOME/opencode/opencode.json`, `%APPDATA%/opencode/opencode.json` (Windows), or `~/.config/opencode/opencode.json` to populate local OpenCode / LM Studio models in the picker.
4. **Nimbalyst CLI Cloud Credentials**:
   - `packages/cli/src/cloud/credentials.ts:25-53`: Reads owner-scoped `<configDir>/credentials.json` for `nim login` OAuth tokens against Nimbalyst sync servers (`NIM_SERVER`, default `https://sync.nimbalyst.com` per `packages/cli/src/index.ts:43`).

---

## 3. Persistent Device / Install IDs & Hashing

| Platform | Identifier | Storage Location | Persistence & Scope | Linkages / Stitching |
|---|---|---|---|---|
| **Desktop (PostHog)** | `nimbalyst_${ulid()}` | `<userData>/analytics-settings.json` (`0o600`) (`AnalyticsService.ts:409-415`) | Persists across restarts until `userData` removed | **Shared into QR pairing payload** (`CredentialService.ts:229, 236`) and **linked to `email`** (`onboardingAnalytics.ts:48-50, 101`) |
| **Desktop (Launch/Session Counters)** | `firstLaunchAt`, `launchCount`, `sessionsCreated`, `lastDailyActiveDate` | `<userData>/analytics-settings.json` (`AnalyticsService.ts:24-37`) | Install-scoped local counters | Bucketed into `launch_number`, `days_since_install`, `local_date` (`AnalyticsService.ts:192-199`, `dailyActiveHeartbeat.ts:146-156`) |
| **Desktop (Sync `deviceId`)** | `sha256(JSON.stringify(["computer-v1", seed, String(dev), String(ino), personalMemberId])).slice(0, 32)` | `<userData>/computer-identity` (`0o600` UUIDv4 seed) + filesystem `dev`/`ino` (`directoryDeviceIdentity.ts:23-84`) | Directory-owned & account-scoped SHA-256 | Sent to `wss://sync.nimbalyst.com` with cleaned `os.hostname()` as `name` (`SyncManager.ts:235-245`) |
| **iOS (PostHog)** | `nimbalyst_mobile_<uuid>` | iOS Keychain (`AnalyticsManager.swift:32-37`) | Survives app restart until unpair (`reset()` at `:119-123`) | **Replaced/merged via `alias(analyticsId)` + `identify(analyticsId)` on QR pair** (`AppState.swift:187`, `AnalyticsManager.swift:75-87`) & **linked to Stytch `email`** (`AppState.swift:640-642`, `AnalyticsManager.swift:91-94`) |
| **iOS (Sync `deviceId`)** | `UIDevice.current.identifierForVendor?.uuidString ?? UUID().uuidString` | `UserDefaults` (`nimbalyst_device_id`) (`WebSocketClient.swift:533-547`) | Vendor/install-scoped | Sent in `deviceAnnounce` alongside raw `UIDevice.current.name` (e.g., `"<User>'s iPhone"`) (`WebSocketClient.swift:508-518, 549-555`) |
| **Android (PostHog)** | `nimbalyst_mobile_<uuid>` | `EncryptedSharedPreferences` (`nimbalyst_analytics`, AES256-GCM) (`AnalyticsManager.kt:44-60`) | Survives app restart until unpair (`reset()` at `:156-162`) | **Merged via `PostHog.alias(analyticsId)` + `identify(analyticsId)` on QR pair** (`AnalyticsManager.kt:97-109`, `SettingsScreen.kt:449, 475`) & **linked to Stytch `email`** (`MainActivity.kt:162`, `AnalyticsManager.kt:114-120`) |
| **Android (Sync `deviceId`)** | Random `UUID.randomUUID()` | `SharedPreferences` (`nimbalyst_device`) | Install-scoped | Sent in `deviceAnnounce` with `Build.MANUFACTURER` + `Build.MODEL` |

### Cross-Device Identity Stitching Assessment
Unlike Crystal (which avoided user PII stitching in PostHog), Nimbalyst actively performs **two forms of identity stitching** inside PostHog:
1. **QR Code `alias()` + `identify()`**: When Desktop generates a sync QR payload (`CredentialService.ts:208-240`), it embeds its `analyticsId` (`version: 5`). On scan, iOS and Android call `PostHog.alias(analyticsId)` followed by `PostHog.identify(analyticsId)` (`AnalyticsManager.swift:80, 85`, `AnalyticsManager.kt:101, 107`), permanently merging the mobile and desktop person profiles inside PostHog.
2. **Email Person Property (`$set: { email }`)**: Desktop onboarding calls `posthog.people.set({ email })` (`onboardingAnalytics.ts:48-50, 101`) when an email is entered, and both iOS and Android automatically call `setEmail(email)` upon Stytch authentication (`AppState.swift:640-642`, `MainActivity.kt:162`). The PostHog server-side allow-list explicitly preserves `email` in `KEPT_PERSON_PROPERTIES` (`packages/electron/src/shared/analytics/posthogIngestAllowList.ts:150-171`).
3. **Hardware Fingerprint Negative Check**: No OS machine-ID libraries (`getMachineId`, `node-machine-id`, Electron `crashReporter`) are used anywhere (`rg` verified 0 matches). Desktop `directoryDeviceIdentity.ts` hashes a local random UUIDv4 seed + filesystem `(dev, ino)` + `personalMemberId` (excludes hostname, MAC, and OS serial).

---

## 4. Exact Outbound Telemetry / Analytics / Crash / OTEL / Relay / Update Endpoints

| Endpoint | Purpose / Component | Trigger & Opt-Out Status | Source Citation |
|---|---|---|---|
| `https://us.i.posthog.com` | **Product & Usage Telemetry** (`posthog-node`, `posthog-js`, `posthog-ios`, `posthog-android`), project `phc_s3lQIILexwlGHvxrMBqti355xUgkRocjMXW4LjV0ATw` | **ON by default** on Desktop, iOS, Android; Desktop renderer gates via `before_send` + `opt_out_capturing()`; mobile calls `identify()` *before* `optOut()` on opted-out launch; all 3 platforms emit an opt-out event right before `optOut()` | `AnalyticsService.ts:12, 167, 419-430`; `renderer/index.tsx:258-329`; `AnalyticsManager.swift:15-16, 53-64, 103-108`; `AnalyticsManager.kt:22-23, 75-87, 133-139` |
| `wss://sync.nimbalyst.com` / `https://sync.nimbalyst.com` (and fallback `https://collabv3.nimbalyst.workers.dev`) | **Closed-Source Collaboration Sync & Mobile Relay** (Cloudflare Worker per `LICENSING.md:7-8`); auth token refresh, device presence, push requests, E2EE session/file/tracker sync | User-enabled Sync / Teams / Mobile pairing; WebSocket URL includes `?token=<jwt>` plus platform/version telemetry query params (`SyncManager.ts:264`); `deviceAnnounce`, `requestMobilePush` (title/body/sessionId), and `fleetActivityUpdate` travel **in plaintext** alongside E2EE payloads | `LICENSING.md:7-8`; `SyncManager.ts:264, 278`; `StytchAuthService.ts:1515, 1739, 2036`; `ShareHandlers.ts:23`; `collabV3WireTypes.ts:303, 313-322`; `packages/ios/NimbalystNative/Sources/Sync/WebSocketClient.swift:139, 507-529` |
| `https://api.stytch.com/v1` / `https://test.stytch.com/v1` | **Authentication** (Stytch B2B/Consumer OAuth for Sync/Teams) | Triggered when user logs into Sync / Teams / Console | `StytchAuthService.ts:157`; `packages/ios/README.md:201` |
| `https://console.nimbalyst.com` | **Teams / Device Console Web Origin** | User-initiated web console / device-code / mobile pages auth | `packages/ios/NimbalystNative/Sources/Pages/ConsoleRoute.swift:7`; `packages/android/app/src/main/java/com/nimbalyst/app/pages/ConsoleRoute.kt:42` |
| `https://github.com/nimbalyst/nimbalyst` (GitHub Releases via `electron-updater`) | **Desktop Auto-Updater** (`latest`/`alpha` feeds, `autoDownload: true`) | Automatic on launch + periodic poll + manual Help menu check; **no Settings UI toggle to disable automatic update checks** | `packages/electron/package.json` (`publish`); `autoUpdater.ts:37-41, 73-74, 92-102, 437, 597, 602` |
| `https://extensions.nimbalyst.com/registry` & `https://cdn.extensions.nimbalyst.com` (& `api.github.com` / `github.com` clones) | **Extension Marketplace Registry & CDN** | Opening Marketplace settings / checking extension updates / installing marketplace or GitHub extensions | `ExtensionMarketplaceHandlers.ts:43, 236, 561`; `packages/marketplace/src/index.ts:20, 29-30` |
| `https://registry.npmjs.org/@nimbalyst/cli/latest` & `https://raw.githubusercontent.com/nimbalyst/nimbalyst-wiki/main/.claude-plugin/plugin.json` (+ `.codex-plugin/plugin.json`) | **CLI & Wiki Plugin Daily Update Notice** | At most once per 24h in CLI/MCP server; suppressed by `NIMBALYST_WIKI_NO_UPDATE_CHECK` or `CI` | `packages/cli/src/mcp/updateCheck.ts:22-28, 36-39, 236-239` |
| `api.push.apple.com` (APNs, via `sync.nimbalyst.com`) & Firebase Cloud Messaging (FCM) | **Mobile Push Notifications & Live Activities** | iOS registers APNs/ActivityKit tokens (`environment: "production"`); Android fetches FCM token only if push is enabled AND `app/google-services.json` is compiled into build (not present in OSS repo) | `packages/ios/NimbalystNative/Sources/Notifications/NotificationManager.swift:183-206`; `packages/ios/README.md:165-167`; `packages/android/app/src/main/java/com/nimbalyst/app/notifications/NotificationManager.kt:103-131`; `packages/android/README.md:13, 75` |
| `https://api.anthropic.com/api/oauth/usage` | **Local Claude Usage Meter** | Activity-aware 30-min polling (when Claude credentials exist locally) | `ClaudeUsageService.ts:55-56, 332` |
| `https://nimbalyst.com/privacy-policy` & `https://github.com/nimbalyst/nimbalyst/issues` | **Static Privacy Policy & User Feedback Links** | External browser navigation only when clicked by user | `SettingsView.swift:301`; `SettingsScreen.kt:502`; `FeedbackIntakeDialog.tsx:24-25` |

*Retired endpoint note*: Live PostHog feature-flag polling for SQLite migration (`sqlite-rollout-alpha` / `sqlite-rollout-stable`) was explicitly removed (`migrationFlag.ts:8-22`, `renderer/index.tsx:278` `advanced_disable_flags: true`, `AnalyticsManager.swift:48` `preloadFeatureFlags = false`, `AnalyticsManager.kt:70` `preloadFeatureFlags = false`), using a local built-in config (`builtInRolloutConfigSource`) instead.

---

## 5. Exact Data & Properties Collected

### IP Addresses & GeoIP Handling (`$geoip_disable` / `$ip` / `sendDefaultPii`)
- **Desktop Main (`AnalyticsService.ts:422-427`)**: Initializes `new PostHog(key, { privacyMode: true, disableGeoip: false, enableExceptionAutocapture: false })`. **Setting `disableGeoip: false` explicitly preserves GeoIP enrichment from the sender's IP address.** While `privacyMode: true` in `posthog-node` omits default host device fields, `disableGeoip: false` leaves client IP resolution enabled in PostHog ingestion.
- **Desktop Renderer (`renderer/index.tsx:258-315`)**: Configures `autocapture: false`, `capture_heatmaps: false`, `disable_session_recording: true`, `capture_exceptions: false`, `capture_pageview: false`, `capture_pageleave: false`, `advanced_disable_flags: true`, **without** `ip: false`, `$geoip_disable`, or `property_blacklist: ['$ip']`. PostHog ingests client IP and performs GeoIP lookup on HTTP requests.
- **iOS (`AnalyticsManager.swift:44-49`) & Android (`AnalyticsManager.kt:66-71`)**: Disable `captureScreenViews`, `captureApplicationLifecycleEvents`, and `preloadFeatureFlags`, with **no IP/GeoIP suppression**.

### Custom / Private Model Names
Several analytics events transmit **verbatim, unsanitized model identifiers** (including private, Bedrock/Azure deployment, OpenCode, or local LM Studio model names) directly to PostHog:
- **`mobile_session_created` (100% ingested via `INGESTED_ALWAYS` at `posthogIngestAllowList.ts:50`)**: Sends `"model": selectedModelId ?? "default"` (`packages/ios/NimbalystNative/Sources/Views/SessionListView.swift:788-790`, `packages/android/app/src/main/java/com/nimbalyst/app/ui/sessionlist/SessionListActions.kt:86-88`).
- **`mobile_meta_agent_created`**: Sends `"model": selectedModelId ?? "default"` (`SessionListView.swift:832-834`, `SessionListActions.kt:85, 88`).
- **`mobile_action_prompt_launched_new_session`**: Sends `"model": action.model ?? "inherit"` (`packages/ios/NimbalystNative/Sources/Views/SessionDetailView.swift:1005-1007`, `packages/android/app/src/main/java/com/nimbalyst/app/ui/sessiondetail/SessionDetailViewModel.kt:171-172`).
- **`codex_session_started` (12.5% distinct-id sampled via `INGESTED_SAMPLED` at `posthogIngestAllowList.ts:112`)**: Sends verbatim `model: initData.model` (`packages/electron/src/main/services/ai/MessageStreamingHandler.ts:2600-2601`).
- **`ai_model_selected`** (`packages/electron/src/renderer/components/Settings/SettingsView.tsx:713-717`) and **`ai_thinking_mode_changed`** (`packages/electron/src/renderer/components/UnifiedAI/SessionTranscript.tsx:1647-1651`): Emit verbatim `modelName` (suffix after `:`) and `model: currentModel` from the client, though both event names are currently filtered into `INTENTIONALLY_DROPPED` server-side by PostHog transformation `Cost control allow-list` (`posthogIngestAllowList.ts:4-8`).

### Prompts, Code, File Paths, Stack Traces & Error Telemetry
- **Prompts / Chat Content / File Contents**: Not collected by PostHog events (`ErrorNotificationService.ts:296-300` intentionally strips error messages/stacks from `uncaught_error`, sending only `errorType` and `errorCategory`). `DatabaseErrorTelemetry.ts:1-9, 59-75` likewise classifies local SQLite/PGLite failures into bounded categories, codes, and engine `sqlState` (`SQLITE_BUSY`, `42P01`) over allow-listed Nimbalyst tables rather than sending raw SQL queries or paths.
- **Exception — Raw SQLite Error Text in `mobile_session_load_error` (100% ingested via `posthogIngestAllowList.ts:54`)**: On iOS (`packages/ios/NimbalystNative/Sources/Views/SessionListLoadFailure.swift:24-39`), `DatabaseError` failures attach `sqliteResultCode` and `detail: "SQLite \(dbError.resultCode.rawValue): \(dbError.message ?? \"no message\")"`, transmitting the raw SQLite engine error message string to PostHog (whereas `SessionDetailView.swift:42-46` explicitly avoids sending detail strings).
- **Dead `workspace_path` IPC Call**: `packages/electron/src/renderer/components/AgenticCoding/WorkstreamGroup.tsx:246` invokes `window.electronAPI.invoke('analytics:track', { event: 'session_reparented', properties: { workspace_path: projectPath } })`, which would pass a raw local directory path—however, `AnalyticsHandlers.ts` registers no `'analytics:track'` handler, so this call fails locally without sending anything.
- **Plaintext Sync Relay Data (`wss://sync.nimbalyst.com`)**: While transcripts, session histories, settings, and file indexes use AES-256-GCM E2EE (`packages/ios/README.md:173`, `CredentialService.ts:188`), the closed-source Cloudflare Worker receives plaintext `deviceAnnounce` metadata (`os.hostname()`, `UIDevice.current.name`, platform, version, focus/idle status), `requestMobilePush` notification `title`, `body`, `sessionId`, and `reason` (`collabV3WireTypes.ts:313-322`, `mobilePushRequest.ts:30-50`), and `fleetActivityUpdate` snapshots (`collabV3WireTypes.ts:322`).

### Behavioral Profile, Identity & Conversion Fields
- **Identity / Demographic Person Properties (`$set` preserved server-side via `KEPT_PERSON_PROPERTIES` at `posthogIngestAllowList.ts:165-171`)**:
  - `email` (Desktop onboarding at `onboardingAnalytics.ts:48-50, 101` + Mobile Stytch OAuth at `AnalyticsManager.swift:91-94`, `AnalyticsManager.kt:114-120`)
  - `user_role`, `custom_role_text` (free-text role), `referral_source`, `referral_search_detail` (free-text search terms), `referral_ai_detail`, `referral_other_detail`, `referral_social_detail`, `developer_mode` (`onboardingAnalytics.ts:45-87`, `docs/POSTHOG_EVENTS.md:535-543`)
- **Daily Active & Device Profile (`daily_active`, 100% ingested + `nimbalyst_session_start`, 12.5% sampled)**: `nimbalyst_version`, `platform`, `cpu_arch`, `has_git_installed`, `release_channel`, `build_type`, `days_since_install` (bucketed), `launch_number` (bucketed), `last_session_at` (ISO timestamp), `has_nimbalyst_session`, `$set_once: { is_dev_user: true }` for non-`OFFICIAL_BUILD` builds (`AnalyticsService.ts:192-213`, `dailyActiveHeartbeat.ts:107-156`, `renderer/index.tsx:287, 306-310`).
- **Marketplace & Push Telemetry**: `extension_marketplace_installed` (100% ingested) and `extension_marketplace_updated`/`uninstalled` capture `extensionId`, `version`, and `source` (`marketplace` / `github-url`) (`posthogIngestAllowList.ts:60`, `ExtensionMarketplacePanel.tsx:234, 277, 316, 372`); `mobile_push_requested` captures `reason`, `forced`, `accepted`, `attemptedCount`, `deliveredCount`, and `rejection` (`mobilePushRequest.ts:42-49`).

---

## 6. Docs / Privacy Policy vs Code Discrepancies

1. **"Never collect emails or IP addresses (no PII)" — Directly Contradicted in Code**:
   - `README.md:118-120` promises: *"We never collect: Usernames, emails, or IP addresses (no PII)"*.
   - `packages/electron/src/renderer/components/GlobalSettings/panels/AdvancedPanel.tsx:471` displays: *"Help improve Nimbalyst by sending anonymous usage data. No prompts or personal info collected."*
   - `docs/ANALYTICS_GUIDE.md:22-24, 444` and `docs/POSTHOG_EVENTS.md:567` state events must never include *"Usernames or email addresses"* or *"IP addresses"*.
   - **Reality**: Plaintext `email` is explicitly transmitted via PostHog `$set` on Desktop onboarding (`onboardingAnalytics.ts:48-50, 101`) and Mobile Stytch login (`AnalyticsManager.swift:91-94`, `AnalyticsManager.kt:114-120`), explicitly documented in the same file (`docs/POSTHOG_EVENTS.md:535`), and explicitly whitelisted on PostHog's ingestion transformation (`posthogIngestAllowList.ts:165-171`). Meanwhile, `AnalyticsService.ts:426` sets `disableGeoip: false` and none of the PostHog clients suppress client IP ingestion.
2. **Outdated Docs Claim Opted-Out Retention Ping (`nimbalyst_session_start`) Still Fires**:
   - `docs/ANALYTICS_GUIDE.md:29, 427, 445` and a stale comment at `AnalyticsService.ts:187-189` assert that even opted-out users send a `nimbalyst_session_start` event on startup as a "retention ping".
   - **Reality**: In current code, `AnalyticsService.setSessionId()` (`AnalyticsService.ts:182-185`) checks `if (!this.allowedToSendAnalytics()) return;` and skips sending `nimbalyst_session_start` when opted out.
3. **Mobile Privacy Manifest (`PrivacyInfo.xcprivacy`) Accurate About Identity Linkage, But Initialization Bugs Violate Opt-Out**:
   - `packages/ios/NimbalystApp/Resources/PrivacyInfo.xcprivacy:13-15` accurately discloses `NSPrivacyCollectedDataTypeProductInteraction` with `NSPrivacyCollectedDataTypeLinked = true` (matching `packages/ios/README.md:186` *"PostHog analytics, linked to identity"*), unlike desktop `README.md` which claims purely "anonymous" telemetry with no PII. However, iOS `SettingsView.swift:312` tells users *"Anonymous usage analytics..."* even after identity linking via QR `alias()` and Stytch `email`.

---

## 7. Zero-Telemetry / Negative-Check Verification

Word-bounded `rg` verification across `packages/` and `.github/` (excluding `node_modules`, `.git`, `CHANGELOG*`, `THIRD_PARTY*`, `__tests__`, `*.test.*`, `docs/`, `design/`):

- **Non-PostHog Analytics / Telemetry / Crash / OTEL SDKs**:
  - Zero occurrences of `mixpanel`, `bugsnag`, `rollbar`, `datadog`, `opentelemetry`, `otel`, `minimalClient`, `SEGMENT_WRITE_KEY`, `plausible`, `umami`, `logrocket`, `fullstory`, `matomo`, `countly`, `appcenter`, `crashlytics`, `instabug`, `statsig`, `launchdarkly`, or `split.io`.
  - Word-bounded matches for `sentry` appeared **only** in static MCP/plugin preset catalogs (`MCPServersPanel.tsx:122, 202, 453-460`, `ClaudeCodePluginsPanel.tsx:54`) and marketing sample data (`marketing/utils/sessionData.ts:331`); zero Sentry SDK dependencies or DSNs exist in runtime code (`package.json`, `packages/electron/package.json`).
  - Word-bounded matches for `amplitude` appeared **only** as audio waveform amplitude in voice/color utilities (`VoiceModeService.ts:915-921`, `AudioPipeline.swift:461, 472`, `projectAppearance.ts:40-43`).
  - **Electron `crashReporter`**: Zero matches across `packages/`.
- **Do Not Track / Hardware IDs**:
  - Zero occurrences of `DO_NOT_TRACK`, `doNotTrack`, `DNT`, `getMachineId`, or `machineId` across `packages/`.
- **Dead `analytics:track` IPC**:
  - `analytics:track` appears **only** at `WorkstreamGroup.tsx:246` (unregistered in `AnalyticsHandlers.ts`).
- **Pre-Consent / Opt-Out Cleanliness Check**: **Failed** due to opt-out default (`analyticsEnabled: true`), fail-open desktop settings reads (`store.ts:2197-2202`, `AnalyticsService.ts:291-296`), opt-out ping emission (`analytics_opt_out` / `mobile_analytics_opt_out`), and iOS/Android `PostHog.identify()` + DEBUG `$set` executing before `PostHog.optOut()` on every opted-out launch.

---

```json
{
  "slug": "nimbalyst",
  "platform": "Nimbalyst",
  "group": "A",
  "repo": "nimbalyst/nimbalyst",
  "sha": "044ff88a1ceb5b2d73686f777f8740ed0539bf1f",
  "commit_date": "2026-11-23T15:34:13-05:00",
  "license": "MIT",
  "status": "Active (Successor to Crystal; closed-source wss://sync.nimbalyst.com Worker per LICENSING.md)",
  "default_state": "ON by default (opt-out) across Electron Desktop (analyticsEnabled: true), iOS (AnalyticsManager.isEnabled = true), and Android (KEY_ANALYTICS_ENABLED = true); fails open to enabled on Desktop settings read errors",
  "opt_out_controls": "Desktop Settings -> Advanced ('Send Anonymous Usage Data'), MCP tool analytics_set_enabled, env vars NIMBALYST_DISABLE_ANALYTICS=1 / PLAYWRIGHT=1 / PLAYWRIGHT_TEST; iOS Settings -> Privacy ('Usage Analytics'); Android Settings ('Send anonymous usage data'); CLI update-check opt-out via NIMBALYST_WIKI_NO_UPDATE_CHECK / CI",
  "do_not_track": false,
  "opt_out_bypasses": "iOS and Android AnalyticsManager.initialize() call PostHog.identify(distinctId) (and DEBUG $set is_dev_user) BEFORE calling PostHog.optOut() on opted-out launches; Desktop isAnalyticsEnabled() and allowedToSendAnalytics() fail open (return true) on store errors; Desktop/iOS/Android all fire an opt-out event right before sdk.optOut(); Desktop AutoUpdaterService polls GitHub Releases with autoDownload=true and no UI opt-out",
  "persistent_ids": "Desktop PostHog: nimbalyst_<ulid> in analytics-settings.json; Mobile PostHog: nimbalyst_mobile_<uuid> in iOS Keychain / Android EncryptedSharedPreferences, stitched to desktop distinctId via QR pairing alias()+identify() and Stytch/onboarding email; Sync deviceId: Desktop sha256(computer-v1 + uuid seed + fs dev/ino + personalMemberId), iOS identifierForVendor + UIDevice.current.name, Android random UUID + Build.MODEL",
  "hashing": "SHA-256 over random UUID seed + directory (dev, ino) + personalMemberId (32 hex chars) for desktop sync deviceId; PostHog sampling panel hashes distinct_id via sha256Hex server-side; NO unsalted external-account credential hashing",
  "external_auth_reads": "Reads ~/.claude/.credentials.json, macOS Keychain ('Claude Code-credentials'), ~/.claude/settings.json, ~/.codex/auth.json + ~/.codex/sessions, and opencode.json strictly for local Claude usage meter (api.anthropic.com), local CLI orchestration, and user-initiated Cloudflare Sandbox provisioning — NOT transmitted to PostHog or telemetry endpoints",
  "telemetry_endpoints": [
    "https://us.i.posthog.com (PostHog SDKs across Electron main/renderer, iOS, Android; project phc_s3lQIILexwlGHvxrMBqti355xUgkRocjMXW4LjV0ATw)",
    "wss://sync.nimbalyst.com / https://sync.nimbalyst.com & fallback https://collabv3.nimbalyst.workers.dev (closed-source Cloudflare Worker sync/relay; plaintext JWT, platform/version, device names, push payloads, fleet activity + E2EE transcripts)",
    "https://api.stytch.com/v1 & https://console.nimbalyst.com (Stytch auth & console web flows)",
    "https://github.com/nimbalyst/nimbalyst (electron-updater auto-updates)",
    "https://extensions.nimbalyst.com/registry & https://cdn.extensions.nimbalyst.com (extension marketplace)",
    "https://registry.npmjs.org/@nimbalyst/cli/latest & https://raw.githubusercontent.com/nimbalyst/nimbalyst-wiki/main/ (CLI/plugin daily update check)",
    "APNs (api.push.apple.com via sync worker) & optional FCM (Android if google-services.json present)",
    "https://api.anthropic.com/api/oauth/usage (local Claude quota meter)"
  ],
  "ip_geo_handling": "No IP suppression on any client: Electron main explicitly configures privacyMode: true with disableGeoip: false; renderer posthog.init, posthog-ios, and posthog-android omit ip=false / GeoIP disable options, allowing PostHog IP/GeoIP enrichment",
  "custom_model_handling": "Verbatim model identifiers emitted on mobile_session_created (100% ingested), mobile_meta_agent_created, mobile_action_prompt_launched_new_session, codex_session_started (12.5% sampled), ai_model_selected (dropped server-side), and ai_thinking_mode_changed (dropped server-side)",
  "docs_discrepancies": "README.md, AdvancedPanel.tsx, ANALYTICS_GUIDE.md, and POSTHOG_EVENTS.md claim Nimbalyst never collects emails, IP addresses, or PII, whereas onboardingAnalytics.ts and iOS/Android AnalyticsManager explicitly send plaintext email via PostHog $set (kept by server-side KEPT_PERSON_PROPERTIES) and AnalyticsService sets disableGeoip: false; ANALYTICS_GUIDE.md also claims opted-out users still emit nimbalyst_session_start retention pings, which current AnalyticsService.setSessionId() actually skips",
  "privacy_rating": "D",
  "one_line_verdict": "Removes Crystal's minimalClient bypass and gates desktop events via fail-closed renderer consent, but defaults telemetry ON across desktop/mobile, sends plaintext email PII + cross-device alias stitching + GeoIP in contradiction of README/UI no-PII claims, emits verbatim custom model names, fails open on desktop store errors, and runs PostHog identify() before optOut() on opted-out mobile launches."
}
```
