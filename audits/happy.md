# Forensic Telemetry & Privacy Audit: Happy (`slopus/happy`)

- **Slug:** `happy`
- **Platform:** Happy (`slopus/happy`)
- **Group:** `A`
- **Repository:** `https://github.com/slopus/happy`
- **License:** `MIT` (`README.md:38,74`)
- **Status:** Active (`main`)
- **Audited SHA:** `cd91953de600de2db4b67d3726459bae5d925a5b`
- **Commit Date:** `2026-11-24T19:46:30-08:00` (`iOS sidebar: a hold lifts a bot or task at once, without its preview`)
- **Repository Scope at `HEAD`:** Monorepo containing `packages/happy-ios` (native Swift iOS app + `HappyMacApp`), `packages/happy-android` (native Kotlin/Compose Android app), `packages/happy-cli` (original `happy` npm CLI/daemon), `packages/happy-server` (Node/Fastify relay), `packages/happy-server-rs` (Rust relay), `packages/happy-server-self-host`, `packages/happy-push-server` (Cloudflare Worker push relay), `packages/happy-wire`, and `packages/happy-i18n` (`README.md:45-72`). As documented in `AGENTS.md:7-8`, the former Expo/React Native `packages/happy-app` client and OTA update scripts have been removed from this repository, while `happy-desktop` and `happy-agent` live in separate `slopus/*` repositories (`README.md:46-47`).
- **Privacy Rating:** **`C`** (Moderate / Mixed — **strong privacy engineering** inside the native mobile analytics pipeline—zero third-party analytics/crash SDKs, custom model masking to `'other'`, `/batch/` `$ip: null` + `$geoip_disable: true`, ephemeral `install_id` deleted on opt-out, and zero telemetry in CLI/Mac/servers—held below **`B`** by default-on mobile analytics without `DO_NOT_TRACK` support, omission of `$ip: null` / `$geoip_disable: true` on direct PostHog `/flags?v=2` requests, unauthenticated `/v1/version` update pings on launch/foreground, and **material `PRIVACY.md` contradictions** including claiming *"We never send ... stack traces"* while sending `crash_frames`, claiming push notifications never pass through Happy's backend while `happy-cli` posts plaintext `title`/`body` to `/push-event`, and omitting server-custodied `ServiceAccountToken` / plaintext `UsageReport` / `happy-push-server` disclosures)

---

## 1. Default State & Opt-Out Mechanism

### Default State by Component
- **Native iOS App (`packages/happy-ios`, `HappyAnalytics`):** **ON by default** in release builds carrying a valid `HappyPostHogKey` in `Info.plist` (`App/Info-Production.plist:38-39`; `Sources/HappyAnalytics/Analytics.swift:108-120,272-287`; `Sources/HappyAnalytics/MigrationFlag.swift:70-82`), with two gating protections:
  1. `Analytics.destination(...)` returns `nil` (disabling both `/batch/` and `/flags`) during `XCTest` / `-screen` automation (`Analytics.swift:273,284-287`), in `debug` builds unless loopback `HAPPY_ANALYTICS_CAPTURE` (`127.0.0.1`, `::1`, `[::1]`, or `localhost`) is set (`Analytics.swift:274-279`), or when `HappyPostHogKey` is empty or unexpanded `$(...)` (`MigrationFlag.swift:74-82`).
  2. Recording stays inactive (`recording == nil`) until `AccountsBridge` finishes reading every signed-in account's `analyticsOptOut` setting (`Analytics.swift:199,210,318,324-333`; `AccountsBridge.swift:39`).
  3. `AnalyticsConsent.deviceOptOut` defaults to `false` (`storage.string(forKey: "analytics.deviceOptOut") != nil` in `UserDefaultsStore(suiteName: "happy-analytics")`, `Analytics.swift:110,298,312`), and account `AccountSettings.defaults["analyticsOptOut"]` defaults to `.bool(false)` (`AccountSettings.swift:23,94`).
- **Native Android App (`packages/happy-android`, `:analytics` module):** **ON by default** in release/preview builds carrying a non-empty `BuildConfig.POSTHOG_KEY` (`app/build.gradle.kts:169`; `Analytics.kt:173-185`). `AnalyticsTarget.of` returns `null` for Robolectric, AndroidX `InstrumentationRegistry`, `bench` builds, `isRunningInTestHarness`, `isUserAMonkey`, `firebase.test.lab == "true"`, or `debug` builds without loopback `debug.happy.analytics_host` (`127.0.0.1` / `localhost`, `Analytics.kt:174-196`). `AnalyticsSwitch` reads `SharedPreferences("analytics")` key `"device_on"` defaulting to `true` (`Analytics.kt:274-283`), and `Settings.analyticsOptOut` defaults to `false` (`Settings.kt:65,146`).
- **macOS App (`HappyMacApp`):** **OFF / Not linked.** `Package.swift:40,124` links `HappyClientKit`, which excludes `HappyAnalytics`.
- **Happy CLI (`packages/happy-cli`), Relays (`happy-server`, `happy-server-rs`, `happy-server-self-host`), and Push Worker (`happy-push-server`):** **OFF / Zero product analytics.** `happy-cli` has no analytics SDKs, sets `DISABLE_AUTOUPDATER = '1'` for local Claude (`scripts/claude_local_launcher.cjs:3-4`), and injects `disableAnalytics: true` when serving local web UI (`commands/server.ts:116-119`). Note that `happy-server` exposes Prometheus metrics (`/metrics`, `api/api.ts:166`; `monitoring/metrics2.ts:37-95`), and `happy-cli` emits plaintext token/cost `usage-report` Socket.IO payloads (`apiSession.ts:901-926`) and plaintext push notification content (`pushNotifications.ts:254-299`).

### Available Opt-Out Controls
1. **iOS In-App UI Toggles (Dual Device + Per-Account):**
   - **Device-wide switch** in Settings (`SettingsViewController.swift:773-774` -> `Analytics.shared.setDeviceOptOut(!on)` writing `"1"` / `nil` to `"analytics.deviceOptOut"`, `Analytics.swift:338-344`).
   - **Per-account synced switch** in Account Settings (`SettingsAccountPages.swift:69-72` -> `session.updateSettings(["analyticsOptOut": .bool(!on)])`), synced via encrypted account settings; **any single signed-in account** having `analyticsOptOut == true` disables device-wide `Analytics` (`Analytics.swift:292-295,315`).
2. **Android In-App UI Toggles (Dual Device + Per-Account):**
   - **Device-wide switch** on Privacy screen (`Privacy.kt:55-99`, `PrivacyTags.ANALYTICS_SWITCH`) calling `graph.analyticsSwitch.set(on)`.
   - **Per-account synced switch** on Account screen (`SettingsRoutes.kt:450`; `AccountScreen.kt:119,180`; `AccountsOptedOut.kt:61-65`); any signed-in account with `analyticsOptOut == true` triggers `Analytics.revoke()` (`AppGraph.kt:435-440`; `AccountsOptedOut.kt:64`).
3. **`DO_NOT_TRACK` Environment Variable:** **Not implemented (`false`).** `docs/analytics-catalog.json:1213` states `"On for a new install, with the Settings switch to turn it off (and DO_NOT_TRACK=1 on desktop once wired, account analyticsOptOut on mobile)."`, and negative `rg` confirms zero code references to `DO_NOT_TRACK` in any package.
4. **CLI / Config Opt-Out:** Not applicable for product analytics in `happy-cli` (no analytics exist); debug remote logging (`DANGEROUSLY_LOG_TO_SERVER_FOR_AI_AUTO_DEBUGGING`) is **OFF by default** (`ui/logger.ts:56`; `ui/doctor.ts:29,208`).

### Opt-Out Enforcement, Fail-Closed Behavior & Remaining Edge Cases
- **Clean In-Memory & Disk Teardown on Opt-Out:**
  - **iOS (`Analytics.stop`, `Analytics.swift:226-238`; `AnalyticsEngine.clear`, `AnalyticsEngine.swift:110-117`):** Cancels flush ticker, cancels in-flight `/flags` request (`flagAsking?.cancel()`), clears in-memory flag cache, stops `Happenings` observation, resets `ConnectionWatch`, increments `generation` so pending tasks abort before running, clears event queue, throttle, and `seenFailures`, and deletes `"analytics.installID"` from `UserDefaults`. Wait-on-account consent prevents pre-settlement events from dropping/leaking (`Analytics.swift:199,243-245`), and `AccountController.analyticsOptOuts()` fails closed (`?? true`, `AccountController.swift:278`) if an account snapshot cannot be read.
  - **Android (`AnalyticsClient.end`/`forget`/`allow`, `Analytics.kt:414-476`):** Increments `generation`, cancels all active OkHttp calls (`posthog.cancelAll()`), clears queue/throttle/exposures/`seenErrors`, clears `AppExits` mark (`mark(null)`), disarms and deletes `noBackupFilesDir/crash_signature` (`CrashSignatures.arm(false)`, `CrashSignature.kt:86-89`), removes `"install_id"` from `SharedPreferences`, and **refuses re-enabling** (`allow()` returns `false`) if deleting `"install_id"` fails (`Analytics.kt:414-426`). `AnalyticsSwitch.read` (`Analytics.kt:279-283`) and `Settings.analyticsOptOut` (`Settings.kt:143-146` via `AccountsOptedOut`) also fail closed on unreadable/corrupt preferences.
- **Minor Edge Cases / Bypasses:**
  1. **iOS `UserDefaults` non-string type edge case (`Analytics.swift:312`):** `storage.string(forKey: "analytics.deviceOptOut") != nil` evaluates to `false` (analytics ON) if a corrupt non-string/non-numeric object (e.g., boolean/dict) were present in `UserDefaults`, though normal writes always store `"1"` or delete the key (`Analytics.swift:341`), and write failures return `false` to Settings (`Analytics.swift:343`).
  2. **iOS MetricKit listener remains registered after opt-out (`CrashReports.swift:15-17,41-43`):** `MXMetricManager.shared.add(self)` is not removed on opt-out, though delivered payloads call `Analytics.shared.log(...)` which immediately drops them when `recording == nil` (`Analytics.swift:199,244`).
  3. **Non-analytics update/push/billing requests do not follow analytics opt-out:** `NativeUpdateCheck` (`POST /v1/version`, `NativeUpdateCheck.swift:128-135`; `HomeServices.kt:333-345`), RevenueCat offerings/receipts (`RevenueCat.swift:69-91`; `RevenueCatApi.kt:74-93`), and CloudPush/Expo push registration (`CloudPush.swift:46`; `HappyPushRelay.kt:73`) remain governed by their respective feature flows rather than the analytics switch.

---

## 2. External Auth / Local Non-Workspace File Reads

Negative `rg` checks confirm that **none of Happy's analytics modules (`HappyAnalytics` or `:analytics`) read external AI CLI auth files** (`.claude`, `.codex`, `.gemini`, `.openclaw`, `.ssh`, keychains).

All non-workspace home-directory reads occur exclusively in `happy-cli` for explicit local agent execution, transcript mirroring, or user-invoked OAuth registration:
1. **Gemini CLI / GCP Credentials & Config (`packages/happy-cli/src/gemini/utils/config.ts:38-106`, `src/index.ts:173-230`):**
   - `readGeminiLocalConfig()` checks `~/.gemini/oauth_creds.json`, `~/.gemini/config.json`, `~/.config/gemini/config.json`, `~/.gemini/auth.json`, and `~/.config/gemini/auth.json` (`config.ts:38-44`) to extract `access_token`/`token`/`apiKey`/`GEMINI_API_KEY`, `model`, and Google Cloud project fields, and falls back to `execSync('gcloud auth application-default print-access-token')` (`config.ts:92`).
   - Invoked by `happy gemini` / `happy gemini model` (`index.ts:173-230`, `agent/factories/gemini.ts:78-86`) purely to authenticate local Gemini runs—never sent to PostHog or any telemetry sink.
2. **Claude Code Config & Transcripts (`packages/happy-cli/src/claude/utils/claudeSettings.ts:21-51`, `path.ts:4-8`):**
   - Reads `$CLAUDE_CONFIG_DIR/settings.json` or `~/.claude/settings.json` solely for `includeCoAuthoredBy` (`claudeSettings.ts:22,40-44`).
   - Resolves `$CLAUDE_CONFIG_DIR/projects/<projectId>` or `~/.claude/projects/<projectId>` (`path.ts:6-7`) solely to mirror local Claude Code transcripts into the user's E2EE Happy session.
3. **OpenClaw Gateway Token (`packages/happy-cli/src/openclaw/runOpenClaw.ts:85-105`, `src/utils/detectCLI.ts:46,70`):**
   - `detectCLI.ts:46,70` checks existence (`existsSync`) of `~/.openclaw/openclaw.json` solely to populate boolean `cliAvailability.openclaw` in E2EE machine metadata.
   - `runOpenClaw.ts:87-105` reads `gateway.auth.token` from `$OPENCLAW_CONFIG_PATH`, `$OPENCLAW_STATE_DIR/openclaw.json`, or `~/.openclaw/openclaw.json` strictly when connecting locally to an OpenClaw gateway.
4. **Codex Skills Discovery (`packages/happy-cli/src/codex/codexSkills.ts:75-99`):**
   - Scans `$CODEX_HOME/skills` or `~/.codex/skills` and `plugins/cache` solely for slash-command discovery (never reads `~/.codex/auth.json`).
5. **User-Invoked `happy connect` Vendor OAuth Custody (`packages/happy-cli/src/commands/connect.ts:78-118`, `packages/happy-server/sources/app/api/routes/connectRoutes.ts:249-333`, `prisma/schema.prisma:310-323`):**
   - Running `happy connect {codex,claude,gemini}` runs local OAuth PKCE and uploads the vendor token (`openai`, `anthropic`, `gemini`) to `POST /v1/connect/:vendor/register` (and updates `~/.gemini/oauth_creds.json` for Gemini, `connect.ts:112`).
   - On `happy-server`, `connectRoutes.ts:261,291,330` encrypts/decrypts `ServiceAccountToken.token` using server-held **`HANDY_MASTER_SECRET`** (`encryptString` / `decryptString`) — meaning vendor OAuth tokens are encrypted at rest for the Happy Server operator, **not** client-side zero-knowledge E2EE.

---

## 3. Persistent Device / Install IDs & Hashing

| Identifier | Scope & Purpose | Generation & Storage | Opt-Out / Linking / Rotation | Citation |
|---|---|---|---|---|
| **PostHog `distinct_id` (`installID` / `install_id`)** | PostHog analytics & `/flags` | Random lowercase UUID v4. iOS: `UserDefaults(suiteName: "happy-analytics")` key `"analytics.installID"`. Android: `SharedPreferences("analytics")` key `"install_id"`. | **Deleted on opt-out** (`storage.set(nil, ...)`, `prefs.edit().remove("install_id").commit()`); regenerated fresh if re-enabled. **Isolated** from Happy accounts & RevenueCat. | `AnalyticsEngine.swift:16,57-65,116`; `Analytics.swift:110`; `Analytics.kt:461-476,629-632,644` |
| **Account `anonID` / `anonId`** | Local multi-account ID & Settings UI display ("Anonymous ID") | First 16 lowercase hex chars of `deriveKey(masterSecret, "Happy Coder", ["analytics", "id"])`. | **NOT sent to PostHog** on iOS or Android (kept local for directory names, SQLite paths, in-memory `ConnectionWatch` keys, and Account Settings display). | `AccountKeys.swift:19,29`; `AccountEncryption.kt:22,37-39`; `AccountSession.swift:119,199`; `SettingsAccountPages.swift:65`; `SettingsRoutes.kt:373,448` |
| **Unused `providerAccountKey`** | Legacy/vector crypto parity | `deriveKey(masterSecret, "Happy Coder", ["analytics", "provider-account"])`. | **Unused at runtime** (`rg` confirms only `AccountEncryption.kt:25,40` and `TsVectorsTest.kt` / `happy-ts-vectors.json` reference it). | `AccountEncryption.kt:25,40` |
| **RevenueCat Customer ID** | App Store / Play Store Pro subscription checks & cross-device receipt adoption | Random `"$RCAnonymousID:"` + 32 lowercase hex digits of UUID v4; persisted in `UserDefaults(suiteName: "happy.subscription")` (`"subscription.install"`) / `SharedPreferences`. Embedded in StoreKit `appAccountToken` & Play `obfuscatedAccountId`. | Never sent to Happy Server or PostHog (`Subscription.swift:12,75`; `voiceRoutes.ts:7`); never deleted on analytics opt-out. | `Subscription.swift:12,75-78,397-420`; `StoreKitSubscriptionStore.swift:34,99`; `Subscription.kt:32,382,392`; `PlayBilling.kt:171` |
| **ElevenLabs Voice User ID** | Server-side 30-day voice usage quota (`participant_name` / `user_id`) | `u_${base64url(HMAC-SHA256(HANDY_MASTER_SECRET, happyUserId))}` computed on `happy-server` / `happy-server-rs`. | Keyed HMAC-SHA256 using server secret (`HANDY_MASTER_SECRET`); links voice calls within ElevenLabs without exposing raw `Account.id`. | `voiceRoutes.ts:13-23`; `packages/happy-server-rs/COMPAT.md:226` |
| **Expo Installation ID** | Push token exchange (`exp.host`) & paired Happy Agent device identifier | Persistent UUID: iOS Keychain `"EXDeviceInstallationUUIDKey"` (`AppDelegate.swift:72` passes to `PUT /v0/environment` `deviceId`); Android `noBackupFilesDir/expo_notifications_installation_uuid.txt`. | Not linked to PostHog; identifies device for push delivery & paired Happy Agent environment registration. | `ExpoPushTokenExchange.swift:10`; `AppDelegate.swift:72`; `ExpoPushTokens.kt:62,100,146` |
| **CloudPush IDs (`happy-push-server`)** | Direct APNs/FCM push registration on `push.cluster-fluster.com` | iOS: `UserDefaults(suiteName: "happy-ios.cloud-push")` `"cloudPush.device"` + per-connection `"cloudPush.registration.<id>"`. Android: uses `installationId`. | Acts as device bearer credential for `happy-push-server` (`devices` table in D1 keyed by `(app_id, device_id)`). | `CloudPush.swift:110-122`; `PushNotifications.swift:71`; `happy-push-server/src/app.ts:23,36`; `devices.ts:47-68` |
| **CLI `machineId`** | Machine entity registration on `happy-server` | `randomUUID()` stored in `~/.happy/settings.json` (`auth.ts:277-285`); regenerated on new auth. | Plaintext `Machine.id` on `happy-server` (`POST /v1/machines`); machine metadata/daemonState stay encrypted. | `ui/auth.ts:277-289`; `api/api.ts:182`; `machinesRoutes.ts:13`; `schema.prisma:274-282` |

---

## 4. Exact Outbound Telemetry / Analytics / Crash / OTEL / Relay / Update Endpoints

| Service / Channel | Exact Endpoint(s) | Client(s) | Default State & Notes | Citation |
|---|---|---|---|---|
| **PostHog Event Batch Ingest** | `https://us.i.posthog.com/batch/` (direct `HTTPTransport` / OkHttp, no redirects; loopback `HAPPY_ANALYTICS_CAPTURE` / `debug.happy.analytics_host` in debug) | `happy-ios`, `happy-android` | **ON by default** (release builds with `HappyPostHogKey` / `POSTHOG_KEY`); stops cleanly on device or account opt-out. No PostHog SDK used. | `AnalyticsEngine.swift:23,155`; `Analytics.swift:272-281`; `PostHog.kt:30-31,53,102`; `Analytics.kt:175-184` |
| **PostHog Feature Flags (`/flags?v=2`)** | `https://us.i.posthog.com/flags?v=2` (iOS) / `https://us.i.posthog.com/flags/?v=2` (Android) | `happy-ios`, `happy-android` | **ON by default** in release builds with key; sends `{ api_key, distinct_id }` (iOS also sends `flag_keys_to_evaluate`); **omits `$ip: null` / geoip suppression**! Stops on opt-out. | `MigrationFlag.swift:12,28-41`; `PostHog.kt:66-73,102` |
| **RevenueCat Subscriptions** | `https://api.revenuecat.com/v1/subscribers/{id}/offerings`, `/v1/subscribers/{id}`, `/v1/receipts` | `happy-ios`, `happy-android` | Active when `HappyRevenueCatAppleKey` / `REVENUE_CAT_GOOGLE_KEY` is configured (no RevenueCat SDK; direct REST client). | `RevenueCat.swift:55,69-91`; `RevenueCatApi.kt:72-93`; `Offers.swift:59-63` |
| **ElevenLabs Voice Agent** | Public/configured `agentId` (`agent_6701k211syvvegba4kt7m68nxjmw` / `CALL_AGENT_ID` / `HappyCallAgentID`) via ElevenLabs WebRTC/LiveKit SDK; server token/usage proxy -> `https://api.elevenlabs.io/v1/convai/conversation/token` & `/conversations` | `happy-ios` (direct SDK), `happy-android` (`VoiceApi` -> `/v1/voice/conversations`, `/v1/voice/usage` + SDK), `happy-server` / `happy-server-rs` | **Opt-in (on user voice call).** Server derives HMAC-SHA256 `u_...` `participant_name` and checks 7,200s / 30-day quota. | `app/build.gradle.kts:171-173`; `Info-Production.plist:37`; `ElevenLabsCallConnection.swift:51-53`; `VoiceApi.kt:42,48`; `voiceRoutes.ts:9-23,142` |
| **Expo Push Service** | `https://exp.host/--/api/v2/push/getExpoPushToken` (device token exchange); `https://exp.host/--/api/v2/push/send` (notification delivery) | `happy-ios`, `happy-android`, `happy-server`, `happy-cli` (`expo-server-sdk` fallback), `happy-push-server` | Token exchange on push opt-in; **`happy-cli` routes session-event pushes via `happy-server` (`POST /v1/sessions/:sessionId/push-event`)** which posts plaintext `title`/`body` to Expo. | `ExpoPushTokenExchange.swift:54`; `ExpoPushTokens.kt:93`; `pushNotifications.ts:79,270-290`; `pushSend.ts:7`; `expo.ts:15` |
| **Happy Push Relay (`happy-push-server`)** | `https://push.cluster-fluster.com/v1/devices/register` (and `/unregister`) -> APNs / FCM / Expo | `happy-ios`, `happy-android` (`HappyCloud` / Agent notifications) | Cloudflare Worker storing device records in D1; rate-limits using `cf-connecting-ip`. | `CloudPush.swift:46`; `HappyPushRelay.kt:73`; `happy-push-server/src/app.ts:57,101`; `devices.ts:47-68` |
| **Happy Relay & Happy Cloud** | Default relay: `https://api.cluster-fluster.com` (user-configurable); Happy Cloud: `https://auth.happy.engineering`, `https://api.happy-clusters.com`, `happybases.com`, `join.happy.engineering` | `happy-cli`, `happy-ios`, `happy-android` | Core E2EE relay + Happy Cloud auth (`GIDClientID` Google OAuth client in `Info-Production.plist:32-33` and `app/build.gradle.kts:80-87`). | `configuration.ts:77`; `ServerConfig.swift:61`; `app/build.gradle.kts:48-87,159`; `HappyCloud.swift:15-30` |
| **Mobile App Update Check** | `POST <serverURL>/v1/version` | `happy-ios`, `happy-android` | Fires **unauthenticated at launch & foreground** regardless of analytics opt-out; sends `{ platform, version, app_id }` (defaulting to `api.cluster-fluster.com`). | `NativeUpdateCheck.swift:128-135`; `HomeServices.kt:333-345`; `versionRoutes.ts:7-45` |
| **Debug Remote Log Sink (Opt-In Env Var)** | `POST <serverURL>/logs-combined-from-cli-and-mobile-for-simple-ai-debugging` | `happy-cli`, `happy-server` | **OFF by default**; enabled only when `DANGEROUSLY_LOG_TO_SERVER_FOR_AI_AUTO_DEBUGGING` env var is set. | `ui/logger.ts:56`; `api/routes/devRoutes.ts:7`; `utils/log.ts:9` |

**Zero-Telemetry / Negative Endpoint Findings:** Zero PostHog, Sentry, Bugsnag, Datadog, Mixpanel, Amplitude, Rollbar, Crashlytics, Firebase Analytics, Segment, or OpenTelemetry (`@opentelemetry`) endpoints or SDKs exist anywhere in `happy-cli`, `HappyMacApp`, `happy-server`, `happy-server-rs`, or `happy-push-server`. `happy-cli` makes **zero** npm registry or GitHub Releases update-check requests (and sets `DISABLE_AUTOUPDATER='1'` in `scripts/claude_local_launcher.cjs:4`).

---

## 5. Exact Data & Properties Collected

### PostHog Analytics & Crash Schema (Mobile Only)
Both `happy-ios` (`AnalyticsSchema.swift:34-116`) and `happy-android` (`Analytics.kt:596-627`) bypass official PostHog SDKs specifically so **device model, OS build number, language, timezone, screen dimensions, and device names are NOT collected** (`PostHog.kt:25-26`), and enforce a strict send-boundary schema allowlist (`AnalyticsSchema.admit` / `AnalyticsSchema.check`) dropping any uncatalogued property:

- **Fixed Privacy Envelope (`/batch/` only, `AnalyticsEngine.swift:125-134`; `Analytics.kt:605-610`):**
  - `distinct_id`: ephemeral per-install UUID v4 (`installID` / `install_id`, deleted on opt-out).
  - `$insert_id`: per-event random UUID v4.
  - `$lib`: `"happy-ios"` or `"happy-android"`.
  - `$process_person_profile`: `false`.
  - `$ip`: `null`.
  - `$geoip_disable`: `true`.
  - Timestamp rounded down to whole seconds (`now.timeIntervalSince1970.rounded(.down)`, `ChronoUnit.SECONDS`).
- **IP & Geolocation Handling (`$ip` / `$geoip_disable` / Edge IPs):**
  - **`/batch/` requests:** Explicitly set `$ip: null` and `$geoip_disable: true` (`AnalyticsEngine.swift:131-132`; `Analytics.kt:609-610`). Note that connecting directly to `https://us.i.posthog.com/batch/` still exposes the TCP/TLS client IP to PostHog's edge servers on the wire, though PostHog's ingest API is instructed not to store or geolocate it.
  - **`/flags?v=2` GAP:** `MigrationFlag.swift:28-41` (`https://us.i.posthog.com/flags?v=2`) and `PostHog.kt:66-73` (`https://us.i.posthog.com/flags/?v=2`) send `{ api_key, distinct_id }` (plus `flag_keys_to_evaluate` on iOS) **with NO `$ip: null`, `$geoip_disable: true`, or `disable_geoip: true` field**.
  - **Push Worker IP:** `happy-push-server/src/app.ts:101` reads Cloudflare's `cf-connecting-ip` header in memory for registration rate limiting.
- **Common Properties (`AnalyticsSchema.swift:34-44`; `Analytics.kt:614-627`):**
  - `client` (`ios`/`android`), `app_version`, `flavor` (`standard`/`preview`/`development`), `os`, `os_version` (major.minor only, e.g. `15` or `8.1`, `Analytics.kt:215-216`), `device_class` (`phone`/`tablet`), `agent_location` (`remote`/`relay`/null), `agent_os` (`macos`/`linux`/`windows`/null), `happy_agent_version`.
- **Custom / Private / Local Model Name Handling (`ModelFamily`, `AnalyticsCatalog.swift:212-230`; `AnalyticsEvent.kt:438-463`):**
  - **Cleanly Masked:** Raw model IDs are **never sent**. Any optional `providerId:` prefix is stripped and matched strictly against known vendor prefixes (`anthropic/opus`, `anthropic/claude-opus`, `anthropic/sonnet`, `anthropic/claude-sonnet`, `anthropic/haiku`, `anthropic/claude-haiku`, `anthropic/fable`, `openai/gpt`, `xai/grok`, `google/gemini`, `zai/glm`, `moonshotai/kimi`). **Every unknown, custom, Ollama, GGUF, or self-hosted model ID maps strictly to `"other"`.**
- **23 Catalogued Events (`AnalyticsSchema.swift:53-105`):**
  - Usage: `app_opened` (`is_cold_start`, `notifications`, `agents_connected` bucket `0`/`1`/`2`/`3+`), `message_sent` (`source`, `target`, `session_client`, `model_family`, `model_provider_kind`, `effort`, `bot_system_key`, `task_depth`, `has_attachment`, `is_steering`), `conversation_created`, `bot_created`, `task_created`, `mobile_pairing_completed`, `mobile_pairing_cancelled`, `team_server_signed_in`, `voice_session_stopped`, `paywall_presented`, `paywall_purchased`, `paywall_restored`, `paywall_cancelled`.
  - Errors & Crashes: `chat_failure_shown`, `chat_failure_action_clicked`, `agent_connection_lost`, `error_shown`, `mobile_pairing_failed`, `sign_in_failed`, `voice_session_error`, `paywall_error`, and **`app_crashed`**.
- **Prompts, Code, Paths & Stack Traces (`app_crashed`):**
  - **Zero prompts, user text, source code, filenames, or workspace paths** are collected by analytics.
  - **HOWEVER, `app_crashed` DOES collect unsymbolicated stack frames (`crash_frames`) + exception class names:**
    - iOS (`CrashReports.swift:29-38`; `CrashSignature.swift:35-85`): Reads Apple MetricKit diagnostic payloads (`MXDiagnosticPayload`) and attaches `exception_type`, `exception_code`, `signal`, `exception_class`, **`crash_frames`** (up to **5 frames** formatted as `<binaryName>+0x<offset>` on the attributed thread), and **`binary_id`** (main app binary UUID), while discarding termination text, exception messages, memory addresses, and thread names.
    - Android (`CrashSignature.kt:20-69,105-108`): Uncaught JVM exception handler writes up to **5 stack frames** outside platform prefixes (`java.`, `kotlin.`, `android.`, `androidx.`, etc.) formatted as `<className>.<methodName>:<line>` (`crash_frames`, R8-minified in store builds) alongside `exception_class` and `cause_class` (no exception messages or filenames).

### Unencrypted Server & Third-Party Operational Data (Outside PostHog)
- **Plaintext Push Notifications via `happy-server` & Expo (`pushNotifications.ts:16-29,58-72,259-290`; `pushDispatch.ts:56-65`; `schema.prisma:80-89`):** `happy-cli` constructs plaintext notification `title` (`"It's ready!"`, `"Permission request"`, `"Clarification needed"`) and `body` (**`metadata.summary.text` — AI-generated session summary — or the last workspace path segment/folder name**) plus deep-link `/session/<sessionId>` and sends them over HTTPS to `happy-server` (`POST /v1/sessions/:sessionId/push-event`), which sends them in plaintext to `https://exp.host/--/api/v2/push/send` (using `AccountPushToken` stored in plaintext Postgres). Similarly `happy-push-server` stores `appId, deviceId, platform, provider, environment, push_token, metadata` (e.g., `{ locale }`) in Cloudflare D1 (`devices.ts:47-68`).
- **Plaintext Token & Cost `UsageReport` (`apiSession.ts:901-926`; `usageHandler.ts:64-88`; `schema.prisma:253-268`):** `happy-cli` emits unencrypted `tokens` (`total`, `input`, `output`, `cache_creation`, `cache_read`) and `cost` (`total`, `input`, `output`) via Socket.IO `usage-report`, persisted in `UsageReport.data` in server Postgres alongside `accountId` and `sessionId`.
- **Server-Custodied Vendor OAuth Tokens (`connectRoutes.ts:249-333`; `schema.prisma:205-213,310-323`):** `ServiceAccountToken` (`openai`, `anthropic`, `gemini`) and `GithubUser.token` are encrypted with server-side `HANDY_MASTER_SECRET` (readable by server operator, not zero-knowledge E2EE), and `GithubUser.profile` is stored as plaintext JSON.
- **Voice Call Audio & Context (`CallContext.swift:18-90`; `StoreVoiceContext.kt:24-100`; `voiceRoutes.ts:93-175`):** Sent unencrypted (outside E2EE) directly to ElevenLabs over LiveKit/WebRTC during active calls. Notably, current `CallContext` sends sidebar bot/task/project directory titles, active focus, and last 5 turns/messages (`CallContext.swift:26-64`) using opaque `OrderKey` references, whereas Android `StoreVoiceContext` still carries legacy reducer session context (`StoreVoiceContext.kt:46-100`).
- **Server HTTP/WS Telemetry (`monitoring/metrics2.ts:37-95`; `HappyAPI.swift:177`; `HappyHttp.kt:89`):** Every API request sends `X-Happy-Client` (e.g., `ios/<ver>`, `android/<ver>`, `cli-daemon/<ver>`), feeding Prometheus labels (`client`, `client_type`).

---

## 6. Docs / Privacy Policy vs Code Discrepancies

`PRIVACY.md` (dated **October 8, 2026**, `PRIVACY.md:3`) describes the retired Expo/React Native client architecture and contains **multiple material contradictions and omissions** relative to live `HEAD` (`cd91953de6` / `2026-11-24`):

| Policy Claim (`PRIVACY.md`) | Actual Code at `HEAD` | Discrepancy Assessment |
|---|---|---|
| *"What we never send: ... stack traces"* (`PRIVACY.md:30`) | `app_crashed` sends **`crash_frames`** (top 5 `binary+0xoffset` or R8 `class.method:line` frames), `exception_class`, `cause_class`, and `binary_id` (`CrashSignature.swift:44-71`; `CrashSignature.kt:20-69`; `AnalyticsSchema.swift:64-73`). | **Direct Contradiction.** Even though unsymbolicated and free of exception messages/paths, transmitting top 5 stack frames contradicts claiming stack traces are never sent. |
| *"Push notifications are sent directly from your devices to each other, not from our backend. This means: We never see the content of your notifications ... Expo's push notification service solely as a delivery mechanism"* (`PRIVACY.md:69-73`) | `happy-cli` posts plaintext `title` and `body` (`metadata.summary.text` or workspace folder basename) + `sessionId` to **`happy-server` `POST /v1/sessions/:sessionId/push-event`** (`pushNotifications.ts:16-29,254-290`; `pushDispatch.ts:56-125`), which forwards to Expo; Happy Cloud also uses **`happy-push-server`** (`push.cluster-fluster.com`) -> APNs/FCM (`CloudPush.swift:46`; `HappyPushRelay.kt:73`). | **Direct Contradiction.** Happy's backend receives, inspects/logs presence against, and relays plaintext notification titles and session summary/folder-name bodies. |
| *"Events are tied to an ID derived from your account's secret key... What PostHog receives automatically: your device model, operating system and version, app version and build, language, time zone, and screen size, and the IP address... PostHog uses that IP address to estimate an approximate location (country and city)"* (`PRIVACY.md:25,29,53`) | Native iOS/Android use an **ephemeral random UUID v4 deleted on opt-out** (`AnalyticsEngine.swift:57-65,116`; `Analytics.kt:629-632`), never account-derived `anonID`; removed PostHog SDK to strip device model/build/language/timezone/screen size (`PostHog.kt:25-26`); and set **`$ip: null` + `$geoip_disable: true`** on `/batch/` (`AnalyticsEngine.swift:131-132`; `Analytics.kt:609-610`) — **except `/flags?v=2` still omits GeoIP suppression** (`MigrationFlag.swift:28-35`; `PostHog.kt:66-73`). | **Materially Outdated (Overstates Collection on `/batch/`, Under-discloses `/flags`).** Code on `/batch/` is substantially more private than `PRIVACY.md`, while `/flags?v=2` still exposes IP without GeoIP suppression. |
| *"When you send a message: ... a short hash that tells your provider accounts apart without naming them"* (`PRIVACY.md:28`) | Native iOS/Android do **not** send any provider-account hash in `message_sent` (`AnalyticsSchema.swift:84-90`; `AccountEncryption.kt:40` derives unused `providerAccountKey` only tested against vectors). | **Outdated.** Legacy Expo-era property removed in native rewrite. |
| Zero-knowledge E2EE claims (`PRIVACY.md:12,78-81`) & *"We cannot read code or conversations"* (`PRIVACY.md:129`) | **Undisclosed server-readable / plaintext data:** (1) `UsageReport` stores unencrypted token/cost usage per session (`usageHandler.ts:64-88`; `schema.prisma:253-268`); (2) `happy connect` `ServiceAccountToken` & `GithubUser.token` are encrypted with server `HANDY_MASTER_SECRET`, not zero-knowledge E2EE (`connectRoutes.ts:261,291`; `schema.prisma:208,315`); (3) `NativeUpdateCheck` pings `POST /v1/version` unauthenticated at launch/foreground (`NativeUpdateCheck.swift:128-135`; `HomeServices.kt:341-345`). | **Omission.** Server-side custody of `ServiceAccountToken`, plaintext `UsageReport`, `happy-push-server` D1 metadata (`locale`), and `/v1/version` update checks are undisclosed. |
| `docs/analytics-catalog.json:1213` mentions `DO_NOT_TRACK=1 on desktop once wired` & `PRIVACY.md:42,45,107` describes web/Expo & `happy-app` | `packages/happy-app` and web/Desktop telemetry do not exist in this repo (`AGENTS.md:7-8`), and `DO_NOT_TRACK` is unimplemented across all packages. | **Stale Cross-Repo Documentation.** |

---

## 7. Zero-Telemetry / Negative-Check Verification

Read-only `rg -n` verification scans were executed across `packages/`, `docs/`, and `.github/` at SHA `cd91953de6` (filtering out test fixtures, localized strings, shader/audio waveform `amplitude`, and `mermaid.min.js`):

1. **Third-Party Analytics, Crash & Error SDKs (`sentry`, `bugsnag`, `datadog`, `mixpanel`, `amplitude`, `crashlytics`, `opentelemetry`, `instabug`, `appcenter`, `firebase-analytics`, `firebase_analytics`, `@segment`, `segment.io`, `analytics-node`, `plausible`, `fathom`, `umami`, `logrocket`, `fullstory`, `rollbar`, `honeycomb`, `statsig`, `launchdarkly`):**
   - **0 telemetry/crash SDK dependencies or calls** anywhere in `packages/`. The only literal match in production code is `"datadog"` as a UI brand icon name (`packages/happy-ios/Sources/HappyCore/Agents/ClayIcons.swift:134`), `CloudEmail.plausible` email validation (`CloudScreens.kt:1182`), and UI shake `amplitude` (`Feedback.kt:22`; `AgentInputCardMotion.swift:32`).
   - Note on Android Firebase config: `packages/happy-android/app/build.gradle.kts:302` emits `google_crash_reporting_api_key` only if `google-services.json` is supplied locally, as part of manual resource extraction replacing the Google Services Gradle plugin; **zero Crashlytics or Firebase Analytics dependencies exist** (only `firebase-messaging`).
2. **CLI, macOS (`HappyMacApp`), Server & Push Worker Cleanliness:**
   - **0 `posthog` or `revenuecat` occurrences** across `packages/happy-cli`, `packages/happy-server`, `packages/happy-server-rs`, `packages/happy-server-self-host`, `packages/happy-push-server`, `packages/happy-wire`, and `packages/happy-ios/Sources/HappyMacApp`. (`packages/happy-server/deploy/handy.yaml:106` only references an unused Kubernetes secret key `/handy-revenuecat`, confirmed unused in `voiceRoutes.ts:7`.)
   - **0 `npm` / GitHub Releases update checks** in `packages/happy-cli`, `packages/happy-server`, `packages/happy-ios`, or `packages/happy-android` (`registry.npmjs.org` appears only in `.github/workflows/release-happy-cli.yml` CI publishing and `package.json` `publishConfig`, while `scripts/claude_local_launcher.cjs:4` explicitly sets `DISABLE_AUTOUPDATER = '1'`).
3. **`DO_NOT_TRACK` & `providerAccountKey` Checks:**
   - **`DO_NOT_TRACK`:** **1 hit total** (`docs/analytics-catalog.json:1213` documentation note only) — **0 code hits**.
   - **`providerAccountKey`:** **Only 2 non-test hits** (`packages/happy-android/crypto/src/main/kotlin/com/slopus/happy/crypto/AccountEncryption.kt:25,40` unused crypto key derivation) + 3 test-vector hits (`TsVectorsTest.kt:137`, `happy-ts-vectors.json:361,373`) — **0 runtime telemetry callers**.
4. **Verified Privacy Hardening Positives:**
   - Both iOS (`AnalyticsEngine.swift`) and Android (`PostHog.kt`) stripped official PostHog and RevenueCat SDKs, disable HTTP redirects, strictly validate events against `AnalyticsSchema`, mask unknown/custom models to `"other"`, delete `installID`/`install_id` on opt-out, suppress `/batch/` IP/GeoIP (`$ip: null`, `$geoip_disable: true`, `$process_person_profile: false`), and keep CLI, macOS, and relay servers free of product analytics.

---

```json
{
  "slug": "happy",
  "platform": "Happy",
  "group": "A",
  "repo": "slopus/happy",
  "sha": "cd91953de600de2db4b67d3726459bae5d925a5b",
  "commit_date": "2026-11-24T19:46:30-08:00",
  "license": "MIT",
  "status": "Active",
  "default_state": "ON by default in release mobile apps (happy-ios, happy-android) for PostHog analytics, PostHog /flags, and crash diagnostics when PostHog API keys are configured (gated on account-settings read); OFF / zero product analytics in happy-cli, HappyMacApp, happy-server, happy-server-rs, and happy-push-server",
  "opt_out_controls": [
    "iOS UI device-wide switch in Settings (Analytics.shared.setDeviceOptOut -> UserDefaults suite 'happy-analytics' key 'analytics.deviceOptOut')",
    "iOS UI per-account switch in Account Settings (session.updateSettings analyticsOptOut; any opted-out signed-in account disables analytics device-wide)",
    "Android UI device-wide switch in Settings > Privacy (AnalyticsSwitch -> SharedPreferences 'analytics' key 'device_on')",
    "Android UI per-account switch in Account Settings (settings.analyticsOptOut; any opted-out signed-in account calls Analytics.revoke())",
    "Debug/test/automation builds disable analytics automatically unless loopback capture host env/sysprop is set",
    "happy-cli / HappyMacApp / servers ship zero product telemetry (CLI remote logging DANGEROUSLY_LOG_TO_SERVER_FOR_AI_AUTO_DEBUGGING is OFF by default)"
  ],
  "do_not_track": false,
  "opt_out_bypasses": [
    "No DO_NOT_TRACK support in code (docs/analytics-catalog.json:1213 defers DO_NOT_TRACK=1 to an external desktop app)",
    "PostHog /flags?v=2 calls on iOS and Android omit $ip:null / $geoip_disable:true suppression properties (though /flags stops on opt-out)",
    "NativeUpdateCheck (POST /v1/version) fires unauthenticated at launch/foreground on iOS and Android regardless of analytics opt-out",
    "iOS AnalyticsConsent initializes deviceOptOut via UserDefaults.string(forKey:) != nil (fails open if a non-string value were stored, though normal UI writes '1' or nil and accounts fail closed); CrashReports keeps MXMetricManager subscribed after opt-out (events dropped at Analytics.shared.log)"
  ],
  "persistent_ids": [
    "Ephemeral per-install random UUIDv4 distinct_id ('analytics.installID' in iOS UserDefaults / 'install_id' in Android SharedPreferences) — DELETED on opt-out",
    "RevenueCat anonymous UUIDv4 '$RCAnonymousID:<32hex>' persisted per install and written into StoreKit appAccountToken / Play obfuscatedAccountId",
    "Expo installation UUID in iOS Keychain ('EXDeviceInstallationUUIDKey') and Android noBackupFilesDir ('expo_notifications_installation_uuid.txt'), also sent to paired Happy Agents as deviceId",
    "CloudPush 'cloudPush.device' UUID in iOS UserDefaults (suite 'happy-ios.cloud-push') / installationId on Android stored in happy-push-server Cloudflare D1",
    "CLI machineId randomUUID() in ~/.happy/settings.json sent as plaintext Machine.id",
    "Account.id CUID and server-computed ElevenLabs HMAC-SHA256 user ID ('u_<base64url>')"
  ],
  "hashing": "KeyTree HMAC-SHA512 / key derivation from account masterSecret yields 16-hex-char anonID (['analytics', 'id']) kept local and NOT sent to PostHog, plus unused providerAccountKey (['analytics', 'provider-account']). ElevenLabs voice user ID uses server HMAC-SHA256(HANDY_MASTER_SECRET, userId).",
  "external_auth_reads": "None in analytics. happy-cli reads ~/.gemini/{oauth_creds,config,auth}.json + ~/.config/gemini/* + gcloud application-default access token (for happy gemini), ~/.claude/settings.json & ~/.claude/projects/* (Claude launcher/transcript mirroring), ~/.openclaw/openclaw.json (gateway token & existence check), and ~/.codex/skills (skill listing). Explicit 'happy connect {codex,claude,gemini}' uploads vendor OAuth tokens encrypted on happy-server with server-held HANDY_MASTER_SECRET (not zero-knowledge E2EE).",
  "telemetry_endpoints": [
    "https://us.i.posthog.com/batch/ (mobile analytics & app_crashed, custom direct HTTP client, no PostHog SDK)",
    "https://us.i.posthog.com/flags?v=2 (iOS) and https://us.i.posthog.com/flags/?v=2 (Android) (PostHog feature flags)",
    "https://api.revenuecat.com/v1/subscribers/* & /v1/receipts (mobile direct REST client)",
    "https://api.elevenlabs.io/v1/convai/* (server voice proxy) and ElevenLabs WebRTC/LiveKit SDK (client audio & call context)",
    "https://exp.host/--/api/v2/push/getExpoPushToken & /send (mobile, happy-server, happy-cli fallback, happy-push-server)",
    "https://push.cluster-fluster.com/v1/devices/register (happy-push-server Cloudflare Worker -> APNs/FCM/Expo)",
    "https://api.cluster-fluster.com (default Happy relay, POST /v1/version, POST /v1/sessions/:id/push-event, Socket.IO usage-report)",
    "https://auth.happy.engineering, https://api.happy-clusters.com, https://happybases.com, https://join.happy.engineering (Happy Cloud)"
  ],
  "ip_geo_handling": "Mixed: mobile /batch/ requests explicitly strip device fingerprint fields and send $ip: null and $geoip_disable: true (though direct TLS connections to us.i.posthog.com still expose client IP at transport layer); however, mobile POST /flags?v=2 requests to us.i.posthog.com omit $ip: null and geoip suppression parameters. happy-push-server reads Cloudflare cf-connecting-ip for rate limiting.",
  "custom_model_handling": "Cleanly masked via strict prefix allowlist (ModelFamily in AnalyticsCatalog.swift:212-230 & AnalyticsEvent.kt:438-463): any unknown, custom, local, or self-hosted model ID maps strictly to 'other' and raw model strings never leave the device.",
  "docs_discrepancies": [
    "PRIVACY.md:30 claims 'What we never send: ... stack traces', directly contradicted by app_crashed sending top 5 crash_frames + exception_class + binary_id (CrashSignature.swift, CrashSignature.kt)",
    "PRIVACY.md:69-73 claims push notifications are sent device-to-device 'not from our backend' and 'We never see the content of your notifications', contradicted by happy-cli posting plaintext title/body (summary text or folder name) via happy-server POST /v1/sessions/:sessionId/push-event and happy-push-server relaying tokens/notifications",
    "PRIVACY.md:25,28,29,53 describes retired Expo SDK behavior (account-derived PostHog ID, provider-account hash, device model/timezone/screen/city GeoIP), contradicting native mobile's ephemeral install_id, stripped SDK metadata, and $ip:null / $geoip_disable:true on /batch/ (while missing /flags GeoIP omission)",
    "PRIVACY.md:12,78-81 omits plaintext server UsageReport (tokens/cost), server HANDY_MASTER_SECRET custody of ServiceAccountToken/GithubUser tokens, happy-push-server D1 device metadata (locale), and unauthenticated POST /v1/version update pings",
    "PRIVACY.md and docs/analytics-catalog.json:1213 still reference removed happy-app/web/Desktop/DO_NOT_TRACK=1 flows not present in this repository"
  ],
  "privacy_rating": "C",
  "one_line_verdict": "Strong privacy-engineered mobile analytics pipeline earning C (zero third-party analytics/crash SDKs, custom model masking to 'other', /batch/ $ip:null + $geoip_disable:true, ephemeral install_id deleted on opt-out, zero telemetry in CLI/Mac/servers), held below B by default-on mobile analytics lacking DO_NOT_TRACK, missing IP/GeoIP suppression on direct PostHog /flags?v=2 requests, unauthenticated launch/foreground /v1/version pings, and material PRIVACY.md contradictions denying stack traces (crash_frames) and backend push-content routing."
}
```
