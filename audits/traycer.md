# Telemetry & Privacy Forensic Audit: Traycer

- **Slug:** `traycer`
- **Platform:** `Traycer`
- **Group:** `A`
- **Repository:** `https://github.com/traycerai/traycer`
- **License:** `MIT`
- **Status:** `Active`
- **Branch / Tag:** `main`
- **Audited Commit SHA:** `15cdf06c24fd6e4e76a7f31a31efa40af0aaa5b7`
- **Commit Date:** `2026-11-25T08:15:38+05:30`
- **Commit Subject:** `feat: a Linux sandbox size that grows shows its range, its price range and the chosen disk (#4142)`
- **Audit Scope & Closed-Source Boundary:** Read-only static audit of all client packages (`clients/desktop`, `clients/gui-app`, `clients/mobile`, `clients/traycer-cli`, `clients/shared`), `protocol/`, `packages/`, `mintlify/`, `package.json`, build/deploy scripts, and `.github/workflows/`. Per `Makefile:12-16`, **Traycer Host (`traycer-host`) and Traycer's cloud backend (`authn.traycer.ai`, `server.traycer.ai`, `relay.traycer.ai`) are closed-source binaries/services outside this repository** and were not directly inspectable beyond client/protocol contracts.

---

## 1. Default State & Opt-Out Mechanism

### Default Telemetry State
- **From unconfigured source builds (`dev` / local):** Both PostHog product analytics and Sentry crash/error reporting default **OFF** only because credentials are omitted:
  - PostHog (`clients/gui-app/src/lib/analytics.ts:3383-3397`): `Analytics` constructor reads `import.meta.env.VITE_POSTHOG_KEY` (`rendererEnvPrefix` in `clients/desktop/vite.renderer.config.ts:13-21`) and sets `this.enabled = !!this.key && import.meta.env.MODE !== "test"`.
  - Desktop Main Sentry (`clients/desktop/src/config.ts:87` & `clients/desktop/src/electron-main/app/crash-reporter.ts:82-105`): `config.sentryDsn` is `""` in source; `SentryElectron.init` runs only if `dsn.length > 0`. Note that Electron's local native `crashReporter.start(...)` runs at `crash-reporter.ts:89-95` with `submitURL: ""`, `uploadToServer: false`.
  - Desktop Renderer Sentry (`clients/desktop/src/config.ts:90` & `clients/desktop/src/renderer-shell/main.tsx:39-84`): `bridge.sentryRendererDsn` is `""` in source; `Sentry.init` runs only if `length > 0`.
  - Mobile Sentry (`clients/mobile/src/sentry.ts:59-64` & `clients/mobile/src/web/main.tsx:181-184`): `sentryInitOptions` returns `null` when `config.sentryDsn` (`TRAYCER_MOBILE_SENTRY_DSN`) is empty.
  - CLI Sentry (`clients/traycer-cli/src/sentry.ts:10-51`): `Sentry.init` runs only if `process.env.TRAYCER_CLI_SENTRY_DSN` is truthy.
- **In shipped release builds:** Telemetry is **ON unconditionally by default** whenever release build workflows bake credentials:
  - `.github/workflows/publish-cli-package-managers.yml:204-208` passes `TRAYCER_CLI_SENTRY_DSN: ${{ secrets.TRAYCER_CLI_SENTRY_DSN }}` into `clients/traycer-cli/scripts/build-cli-npm.cjs:129` (`--define:process.env.TRAYCER_CLI_SENTRY_DSN=...`), hardcoding the DSN into `@traycerai/cli` on npm.
  - `clients/desktop/scripts/set-deploy-target.cjs:27-28` bakes `TRAYCER_DESKTOP_SENTRY_DSN` and `TRAYCER_DESKTOP_SENTRY_RENDERER_DSN` into `clients/desktop/src/config.ts` during release stamping, while Vite bakes `VITE_POSTHOG_KEY` (`clients/desktop/vite.renderer.config.ts:18`) into the renderer bundle (`README.md:106` confirms Sentry and PostHog are enabled in release builds).
- **Local-only performance NDJSON (`clients/gui-app/src/lib/perf/perf-telemetry.ts:18-65`):** Writes `[traycer-perf]` events locally to `<userData>/traycer-perf.ndjson` (on in `dev`, **OFF** by default in production unless `localStorage["traycer:perf:telemetry"] === "1"`). Does not transmit over the network.

### Opt-Out Controls, Do Not Track, and Bypass/Gap Findings
- **Zero user opt-out controls for PostHog or Sentry:** Repo-wide non-test search across `clients/` and `protocol/` found **no** setting, UI toggle, CLI flag, config file key, or runtime environment variable (`telemetryEnabled` exists only in `clients/gui-app/src/lib/perf/perf-telemetry.ts:35` for the local perf NDJSON logger).
  - Settings panels (`clients/gui-app/src/lib/settings-sections.ts:40-81`, `clients/gui-app/src/components/settings/panels/general-settings.definitions.ts:19-100`, `clients/gui-app/src/components/settings/panels/app-diagnostics-settings-panel.tsx:44-95`) expose log-level verbosity and manual heap inspection only; there is no toggle to disable PostHog analytics or Sentry crash/error reporting.
  - PostHog's `opt_out_capturing()` API is never called anywhere in the codebase.
- **Do Not Track (`DO_NOT_TRACK` / `navigator.doNotTrack`):** Neither `process.env.DO_NOT_TRACK` nor `navigator.doNotTrack` / Global Privacy Control (`navigator.globalPrivacyControl`) is checked anywhere in `clients/` or `protocol/`.
- **`privacyMode` does NOT gate client telemetry:** `protocol/src/auth/_internal/schemas.ts:123,144,167` defines `privacyMode: z.boolean()` on organization/user/team auth payloads (controlling server-side prompt/conversation logging per `README.md:104` and `mintlify/account/teams.mdx:82`), but **no client code reads `privacyMode` to suppress PostHog or Sentry** (`rg` across all non-test files in `clients/` returns zero matches).

---

## 2. External Auth / Local Non-Workspace File Reads

- **Desktop Browser Login Import (`clients/desktop/src/electron-main/browser-view/storage/login-import/import-logins.ts:66-88` & `sources.ts:76-89`):**
  - Reads external browser profiles/cookie databases (`chrome`, `edge`, `brave`, `arc`, `vivaldi`, `opera`, `aside`, `helium`, `chromium`, `firefox`, `safari`, or user-selected cookie files) and unlocks Chromium encryption keys via OS Keychain / Linux Secret Service / Windows DPAPI.
  - **Trigger & boundary:** Runs **only** on explicit user initiation in Settings (scan performs metadata `stat` + `Local State` / `profiles.ini` discovery without touching OS keystores; keystore unlock occurs only after explicit user confirmation on Import). Imported cookies are written only into Electron's local browser partition and pushed to Traycer Hosts as the user's browser jar—`import-logins.ts:73-77` explicitly isolates failures to closed enums/errno codes so decrypted cookies, profile paths, or keychain answers never enter logs or Sentry.
- **Claude / Codex Session Import (`clients/gui-app/src/components/session-import/use-session-import-scan.ts:20-28`):**
  - Instructs the connected host to scan `~/.claude` and `~/.codex` **only** while the user opens Settings Session Import or reaches the onboarding import step (`if (!active)` closes immediately).
- **CLI Workspace-Scoped `.claude` Read (`clients/traycer-cli/src/internal/claude-scheduled-tasks.ts:18-38`):**
  - `countClaudeScheduledTasks(cwd)` reads `<cwd>/.claude/scheduled_tasks.json` strictly inside the workspace directory `cwd` to count scheduled tasks before host actions; it does not read user home auth files.
- **GitHub CLI Auth (`clients/shared/github-release-auth/github-cli.ts:12-42`):**
  - Invokes `gh auth token --hostname github.com` only for staging/private release feed downloads (`clients/desktop/src/electron-main/app/updater.ts:51-69`), never sent to telemetry endpoints.
- **Closed-Source Host Note (`protocol/provider-auth.md`):**
  - While the open-source clients do not harvest third-party agent auth files for telemetry, `Makefile:13-15` notes `traycer-host` is external/closed-source and `protocol/provider-auth.md` documents that the host checks local provider credentials (e.g., Keychain, `.claude.json`, `.env`, OAuth files) for agent orchestration.

---

## 3. Persistent Device / Install IDs & Hashing

- **PostHog `$device_id`, `distinct_id`, `$session_id`, `$window_id` + Plaintext User Link (`clients/gui-app/src/lib/analytics.ts:2941-3016,3029-3117,3434-3465` & `clients/gui-app/src/stores/auth/auth-store.ts:481-485`):**
  - Before sign-in, PostHog SDK persists a client-side anonymous UUID (`$device_id`, initially equal to `distinct_id`) across renderer restarts (`analytics.ts:3430-3442`).
  - Upon account verification (`setSignedIn` at `auth-store.ts:484`), Traycer calls `Analytics.getInstance().identify(contextMetadata.userId, safeProfile.email)`, which invokes `posthog.identify(userId, { email })` (`analytics.ts:3461`).
  - `sanitizePostHogCaptureResult` (`analytics.ts:3064-3099`) explicitly permits `$identify` events containing `$anon_distinct_id` (the pre-login `$device_id` UUID), `distinct_id` (Traycer's `userId` UUID), and **`$set: { email }`** (plaintext user email, validated by regex at `analytics.ts:3033-3040` *"deliberately sent so PostHog dashboards can look a user up"*). Cross-account transitions reset identity via `posthog.reset()` (`analytics.ts:3451-3459`).
  - Every `AnalyticsEvent` also carries persistent UUID `distinct_id`, SDK-generated `$session_id`, and `$window_id` (`analytics.ts:2950-2969,3007-3016,3101-3116`).
- **Traycer Host UUID (`hostId`) in CLI Crash Telemetry (`clients/traycer-cli/src/host/crash-telemetry.ts:249,341-372,380-395` & `clients/traycer-cli/src/host/pid-metadata.ts:22-27`):**
  - Persistent per-host UUID (`hostId`) generated/maintained by the host installation (`clients/traycer-cli/src/host/install-auth.ts:599-612`, `clients/traycer-cli/src/host/update-run.ts:3755,3940-3942`) is attached **verbatim (unhashed)** to Sentry host-crash events and throttled crash summaries via `scope.setUser({ id: event.hostId })` (`crash-telemetry.ts:369,391`) so Sentry's "users" count tracks distinct hosts.
- **No Machine/Hardware Fingerprinting for Telemetry:** No reads of `/etc/machine-id`, `IOPlatformUUID`, Windows `MachineGuid`, or `node-machine-id` exist in `clients/`. (`os.hostname()` is sent as `hostLabel` / `host_label` only to Traycer's first-party `authnBaseUrl` during CLI device login and host credential minting at `clients/traycer-cli/src/auth/login-flow.ts:51` and `clients/traycer-cli/src/auth/host-credential-mint.ts:88-91` for the user's Sessions/Devices UI.)
- **SHA-256 Usage (Non-Telemetry):** `createHash("sha256")` is used strictly for update archive/recovery integrity digests and hashing staged one-time team install tokens for local receipt verification (`clients/traycer-cli/src/host/team-install-token.ts:175,204-208`).

---

## 4. Exact Outbound Telemetry / Analytics / Crash / OTEL / Relay / Update Endpoints

| Surface / Purpose | Target Endpoint | Auth / Gate | Source Citation |
| --- | --- | --- | --- |
| **Product Analytics (PostHog)** | `https://us.i.posthog.com` | Enabled whenever `VITE_POSTHOG_KEY` is baked (release builds; off in test/unconfigured source) | `clients/gui-app/src/lib/analytics.ts:1309-1310,3384-3390` |
| **Crash & Error Reporting (Desktop Main Sentry)** | Dynamic DSN from `config.sentryDsn` (`TRAYCER_DESKTOP_SENTRY_DSN` baked at release stamp) | Empty (`""`) in source; ON in release builds (`@sentry/electron/main` + offline queue) | `clients/desktop/src/config.ts:87`; `clients/desktop/scripts/set-deploy-target.cjs:27`; `clients/desktop/src/electron-main/app/crash-reporter.ts:83,108-175` |
| **Crash & Error Reporting (Desktop Renderer Sentry)** | Dynamic DSN from `bridge.sentryRendererDsn` (`TRAYCER_DESKTOP_SENTRY_RENDERER_DSN`) | Empty (`""`) in source; ON in release builds (`makeFetchTransport` directly from renderer) | `clients/desktop/src/config.ts:90`; `clients/desktop/scripts/set-deploy-target.cjs:28`; `clients/desktop/src/renderer-shell/main.tsx:39-84` |
| **User Bug Reports (Desktop Support Feedback)** | Desktop Main Sentry DSN via `Sentry.captureFeedback` | User-initiated form only (optional `name`/`email`, scrubbed logs/traces/screenshots) | `clients/desktop/src/electron-main/app/support.ts:519-609` |
| **Crash & Error Reporting (Mobile Sentry)** | Dynamic DSN from `config.sentryDsn` (`TRAYCER_MOBILE_SENTRY_DSN`) | Empty in local builds; ON in release builds (`@sentry/browser`, errors-only) | `clients/mobile/src/sentry.ts:59-87`; `clients/mobile/src/web/main.tsx:181-184` |
| **Crash & Error Reporting (CLI & Host Supervisor Sentry)** | Dynamic DSN from `process.env.TRAYCER_CLI_SENTRY_DSN` | Unset in source; baked into `@traycerai/cli` npm release via GitHub Actions secret | `.github/workflows/publish-cli-package-managers.yml:206`; `clients/traycer-cli/scripts/build-cli-npm.cjs:129`; `clients/traycer-cli/src/sentry.ts:10-51` |
| **Traycer First-Party Cloud & Relay (Production / Staging)** | `https://authn.traycer.ai`, `https://server.traycer.ai`, `https://traycer.ai`, `wss://relay.traycer.ai/attach` (staging: `*.dev.traycer.ai`); desktop preconnects on launch | Core service auth, push token registration (`mobile`), remote host attach relay | `clients/desktop/src/config.ts:56-79`; `clients/desktop/src/electron-main/app/network.ts:7-30`; `clients/mobile/vite.config.ts:52-68`; `clients/traycer-cli/src/config.ts:34-35`; `clients/mobile/src/push-registration.ts:1-36` |
| **Release & Auto-Update Feeds** | `https://releases.traycer.ai/desktop/`, `https://releases.traycer.ai/{host,cli}/versions.signed.json`, `https://github.com/traycerai/traycer/releases`, `https://api.github.com/repos/...` (staging: `releases.dev.traycer.ai`) | Auto-updater / host-installer / CLI update fetches | `clients/desktop/src/electron-main/app/desktop-release-index.ts:13-53`; `clients/desktop/src/electron-main/app/desktop-release-feed.ts:604,642,678,1009`; `clients/desktop/src/electron-main/app/updater.ts:2088,2120,2224,2301,2373`; `clients/traycer-cli/src/config.ts:49-99`; `clients/traycer-cli/src/registry/manifest-url.ts:1-12` |
| **Curated Assets & Themes** | `https://assets.traycer.ai/start-page/wallpapers/v1.json` (preconnect + wallpaper picker), `https://open-vsx.org/api` (user theme search/install) | First-party static asset CDN / Open VSX (`credentials: "omit"` on wallpapers) | `clients/desktop/src/electron-main/app/network.ts:10`; `clients/gui-app/src/lib/appearance/curated-wallpapers.ts:10,153-183`; `clients/gui-app/src/lib/themes/open-vsx.ts:9,42-54` |
| **Staging Dev Soundtrack Bucket** | Google Cloud Storage bucket `traycerai-dev-soundtrack` (`https://www.googleapis.com/auth/devstorage.read_only`) | Staging builds with bundled service-account key only (`null` in prod) | `clients/desktop/src/electron-main/soundtrack/soundtrack-source.ts:10-25`; `clients/desktop/src/config.ts:161-169` |

---

## 5. Exact Data & Properties Collected

### A. IP Address & Geolocation Handling
- **PostHog (UNSUPPRESSED IP/GeoIP):** `POSTHOG_CONFIG` (`clients/gui-app/src/lib/analytics.ts:1309-1341`) sends directly to `https://us.i.posthog.com` with **neither `ip: false` nor `$geoip_disable: true`**, and no `$ip` override in `registerGlobals` (`3412-3422`) or `sanitizePostHogCaptureResult` (`3059-3117`). Consequently, PostHog Cloud receives the user's client IP address on every ingestion request and applies server-side GeoIP derivation by default.
- **Sentry IP/PII:**
  - Desktop Main (`clients/desktop/src/electron-main/app/crash-reporter.ts:148`), Desktop Renderer (`clients/desktop/src/renderer-shell/main.tsx:49`), and Mobile (`clients/mobile/src/sentry.ts:69`) explicitly set `sendDefaultPii: false` and delete `request.cookies` and `request.headers` (`clients/shared/platform/sentry-scrub.ts:153-154`).
  - CLI Sentry (`clients/traycer-cli/src/sentry.ts:15-51`) omits explicit `sendDefaultPii: false` (relying on `@sentry/node` `^10.76.2` SDK default `false`), and also omits `beforeBreadcrumb` hook registration. Note: Without `server-side` IP scrubbing in Sentry project settings, HTTP transport IP visibility depends on Sentry server configuration.

### B. Custom / Private Model Names
- **Clean / Not Collected in Client Telemetry:** Neither PostHog nor Sentry captures user-configured custom LLM model strings or arbitrary model IDs.
  - PostHog event schemas restrict harness/provider properties to closed union allowlists (`AnalyticsHarness` at `clients/gui-app/src/lib/analytics.ts:221-243`; `AnalyticsProvider` at `358-380`). Model IDs/names appear only in the local/account Usage analytics UI (`mintlify/settings/account.mdx:45-56`), never in `AnalyticsEvent`.

### C. Prompts, Code, File Paths, URLs & Stack Traces
- **PostHog Payload Strict Allowlist (`clients/gui-app/src/lib/analytics.ts:3059-3117,1311-1340`):**
  - Disables `autocapture`, `rageclick`, `capture_pageview`, `capture_pageleave`, `capture_heatmaps`, `capture_dead_clicks`, `capture_exceptions`, `capture_performance`, `disable_session_recording: true`, `disable_surveys: true`, `disable_product_tours: true`, `disable_web_experiments: true`, `advanced_disable_flags: true`, `advanced_disable_feature_flags: true`, `save_campaign_params: false`, `save_referrer: false`, and denylists `$current_url`, `$host`, `$pathname`, `$referrer`, `$referring_domain`, `$initial_current_url`, `$initial_referrer`, `$initial_referring_domain`, `$session_entry_url`.
  - `sanitizePostHogCaptureResult` validates every property against exact per-event schemas; drops any unknown event or property, keeping prompt text, repository names, and file paths out of PostHog.
- **Gratitude Match Content Trigger (`clients/gui-app/src/components/share-on-x/share-on-x-controller.tsx:164-176` & `gratitude-match.ts:1-16`):**
  - Every sent chat message is matched locally against fixed gratitude phrase families (`thank_you`, `appreciate`, `great_job`, etc.); when matched, **no prompt text leaves**, but `AnalyticsEvent.SharePromptShown` emits `{ trigger: "gratitude", match_key, showing }` to PostHog—signaling inferred message sentiment/phrase family linked to the identified account.
- **Sentry Scrubbing & Minidump Isolation (`clients/shared/platform/sentry-scrub.ts:48-50,94-275` & `crash-reporter-guest-scope.ts:76-99`):**
  - Automatic errors, unhandled rejections, React ErrorBoundary catches (`clients/gui-app/src/lib/report-issue-error-capture.ts:59-94`), CLI failures (`clients/traycer-cli/src/runner/runner.ts:222-230`), and desktop `browser`/`gpu`/`utility` native crashes send **stack traces, exception messages, breadcrumbs, tags, and span attributes** (sampled at `0.25` error / `0.1` traces & profiles in Desktop Main production; `0.1` traces & profiles in Desktop Renderer/CLI production; `1.0` error rate in Mobile/Renderer/CLI).
  - `scrubSentryEventInPlace`, `scrubSentryBreadcrumbInPlace`, `scrubSentryTransactionInPlace`, and `scrubSentrySpanInPlace` delete HTTP headers/cookies, strip URL query strings/credentials (`reduceUrlToOriginAndPath` / `reduceRequestTargetToPath`), redact secret tokens (`redactSensitiveText`), and redact OS home-directory names (`redactHomeDirectoryNames`) from messages, fingerprints, tags, extras, and stack frame paths (note: non-home install path suffixes remain for stack grouping per `sentry-scrub.ts:125-128`).
  - `desktopSentryBeforeSend` (`crash-reporter-guest-scope.ts:76-99`) drops **all** Electron renderer `.dmp` minidumps so browser guest cookie-jar/DOM heap memory cannot leak.

### D. Behavioral Profile & User/Device Properties
- **Identified Person Properties (`clients/gui-app/src/lib/analytics.ts:3029-3040,3076-3099`):**
  - `userId` (UUID `distinct_id`), pre-login `$anon_distinct_id` UUID, and **plaintext `email`** via `$identify` / `$set`.
- **Global App Context (`clients/gui-app/src/lib/analytics.ts:3412-3422,2971-3000`):**
  - `app: "gui-app"`, `app_surface` (`desktop` | `mobile`), `app_version`, `platform` (`macos` | `windows` | `linux` | `ios` | `android` | `other`), `release_channel`, plus `$session_id` and `$window_id`.
- **Event Coverage (162 Allowlisted `AnalyticsEvent` Types at `clients/gui-app/src/lib/analytics.ts:490-659`):**
  - App launch (`app_opened`, `restored_tabs`), auth lifecycle, host setup/failover/updates, onboarding steps, session imports, provider configuration changes (`provider`, `operation`—without API keys or paths), task/worktree/chat/queue/diff/terminal/artifact/comment/share/voice actions, settings changes (`setting_changed`), error-report submissions, plus:
    - **Periodic 3-hourly resource telemetry (`clients/gui-app/src/providers/app-telemetry-bridge.tsx:32-38` & `clients/gui-app/src/lib/resources/resource-telemetry.ts:10-42,332`):** Emits `app_resource_sample` and `app_resource_pressure` containing `js_heap_mb`, `js_heap_limit_mb`, `heap_slope_mb_per_h`, `session_age_bucket`, `open_tabs`, and `pressure_tier`.
    - **Daily UI layout telemetry (`app-telemetry-bridge.tsx:39-46`):** Emits `layout_snapshot` (at most once per 24h) detailing user layout configuration.
- **CLI Host Supervisor Crash Telemetry (`clients/traycer-cli/src/host/crash-telemetry.ts:249-313,360-395`):**
  - Unhashed `hostId` as `user.id`, `cli_version`, `host_version`, `host_running_version`, `platform`, `arch`, `os_release`, `crash_kind`, closed-set `host_fatal` / `host_error_class` / `host_error_code`, `uptime_bucket`, `uptimeMs`, `supervisorPid`, `childPid`, `exitCode`, `signal`, and V8 heap memory breakdown (`heapLimitBytes`, `heapUsedBytes`, `rssBytes`, `heapSpaceUsedBytes.*`). Throttled to 3 reports per hour per crash kind (`HOST_CRASH_REPORTS_BEFORE_THROTTLE = 3`).

---

## 6. Docs / Privacy Policy vs Code Discrepancies

1. **"May be enabled in release builds" vs Unconditional On-by-Default & Zero Opt-Out:**
   - `README.md:106` states: *"Crash reporting (Sentry) and analytics (PostHog) may be enabled in release builds."* `mintlify/changelog.mdx:220` states: *"Traycer now collects product usage data. Automatic capture, session recording, page addresses, errors, and your content are all switched off; events are linked to your Traycer account."*
   - **Code reality:** Neither the README nor Mintlify docs disclose that **there is no user opt-out setting, CLI flag, or `DO_NOT_TRACK` / `navigator.doNotTrack` support**, leaving analytics and crash reporting non-disableable in official release builds.
2. **Plaintext Email & Unsuppressed IP/GeoIP Undisclosed:**
   - While `mintlify/changelog.mdx:220` discloses *"events are linked to your Traycer account"*, neither `README.md` nor `mintlify/` documents that **`posthog.identify` sends the user's plaintext `email`** (`clients/gui-app/src/lib/analytics.ts:3029-3040,3461`) or that **`POSTHOG_CONFIG` omits `ip: false` and `$geoip_disable: true`**, allowing PostHog Cloud (`us.i.posthog.com`) to log client IP addresses and GeoIP data.
3. **"Errors... are all switched off" Scope Ambiguity:**
   - `mintlify/changelog.mdx:220` says *"errors, and your content are all switched off"* under the PostHog **Analytics** bullet (accurate for PostHog's `capture_exceptions: false`), which can mislead readers who do not realize automatic **Sentry** error, transaction, and profile telemetry (`clients/desktop/src/electron-main/app/crash-reporter.ts`, `clients/desktop/src/renderer-shell/main.tsx`, `clients/traycer-cli/src/sentry.ts`, `clients/mobile/src/sentry.ts`) is concurrently active in release builds without any opt-out toggle.
4. **Privacy Mode Scope Ambiguity (`mintlify/settings/account.mdx:68` & `README.md:104`):**
   - `mintlify/settings/account.mdx:68` lists *"Turn Privacy Mode on or off -> Web dashboard -> Settings -> Privacy Settings"*, and `mintlify/account/teams.mdx:82` limits its definition to prompt text in usage logs. Neither explicitly clarifies that enabling Privacy Mode does **not** disable client PostHog analytics or Sentry crash telemetry (`privacyMode` is completely unread by `clients/`).

---

## 7. Zero-Telemetry / Negative-Check Verification

Traycer is **not** a zero-telemetry codebase (release builds enable PostHog and Sentry telemetry with no opt-out). However, targeted non-test static searches confirm **zero** presence of other common telemetry/OTEL/analytics SDKs across `clients/`, `protocol/`, and `packages/`:

- **Word-boundary `rg` check (`clients/`, `protocol/`, `packages/`, excluding tests & `bun.lock`):**
  ```text
  rg -n -i --glob "!*.test.*" --glob "!*.spec.*" --glob "!**/__tests__/**" --glob "!bun.lock" \
    "(\bopentelemetry\b|@opentelemetry|\botlp\b|\bmixpanel\b|\bamplitude\b|@segment/|segment\.com|segment\.io|google-analytics|\bgtag\b|\bbugsnag\b|\brollbar\b|\braygun\b|honeycomb\.io|\bdatadog\b|axiom\.co|plausible\.io|\bumami\b|\baptabase\b|\btelemetrydeck\b|\bappcenter\b|@firebase|firebase\.google|\bfullstory\b|\blogrocket\b|\bhotjar\b|\brudderstack\b|\bsnowplow\b|\bmatomo\b|\bstatsig\b|\blaunchdarkly\b)" \
    repos/traycer/clients repos/traycer/protocol repos/traycer/packages
  ```
  - **Result:** 0 real SDK matches (all 5 lines matched were UI/domain prose words `amplitude` in `dictation-waveform.tsx:25` and `segment.command` in `chat-*` files, verified in `chat-activity-groups.ts:418`).
- **Package dependency check (`package.json:100-245`):** Only `@sentry/{browser,core,electron,esbuild-plugin,node,vite-plugin}`, `posthog-js`, and local `electron-log` are declared; zero OpenTelemetry, Mixpanel, Amplitude, Segment, Datadog, or Firebase SDK dependencies exist.
- **Confirmed clean behaviors:** No custom/private LLM model strings exported to telemetry; no PostHog autocapture/session replay/pageviews/heatmaps; all browser renderer minidumps dropped before Sentry upload; browser cookie login import strictly user-triggered and isolated from logs/Sentry.

---

```json
{
  "slug": "traycer",
  "platform": "Traycer",
  "group": "A",
  "repo": "https://github.com/traycerai/traycer",
  "sha": "15cdf06c24fd6e4e76a7f31a31efa40af0aaa5b7",
  "commit_date": "2026-11-25T08:15:38+05:30",
  "license": "MIT",
  "status": "Active",
  "default_state": "OFF in unconfigured source builds (empty keys); ON unconditionally in shipped release builds (VITE_POSTHOG_KEY + Sentry DSNs baked at build time)",
  "opt_out_controls": "None (zero UI toggles, CLI flags, config keys, or runtime env vars for PostHog or Sentry; account privacyMode is unread by client telemetry)",
  "do_not_track": "Ignored (neither DO_NOT_TRACK nor navigator.doNotTrack is checked)",
  "opt_out_bypasses": "No opt-out mechanism exists to bypass; enabling account privacyMode only hides server-side prompt logs and leaves client PostHog and Sentry telemetry active",
  "persistent_ids": "PostHog SDK persisted $device_id UUID ($anon_distinct_id) linked via $identify/$set to Traycer userId UUID + plaintext email, plus $session_id and $window_id; CLI host crash reports send unhashed persistent hostId as Sentry user.id",
  "hashing": "No hashing of userId, email, or hostId in telemetry (plaintexts sent); SHA-256 used only for artifact integrity and team-install token receipt verification",
  "external_auth_reads": "User-initiated Desktop browser cookie/login import (Chrome/Edge/Brave/Arc/Vivaldi/Opera/Aside/Helium/Firefox/Safari + OS keystores on explicit Import only); host scans ~/.claude and ~/.codex only when Session Import UI is open; workspace-local .claude/scheduled_tasks.json read by CLI; closed-source traycer-host checks provider auth per protocol/provider-auth.md",
  "telemetry_endpoints": "https://us.i.posthog.com (PostHog); build-stamped Sentry DSNs (@sentry/electron main/renderer, @sentry/browser mobile, @sentry/node CLI); releases.traycer.ai, github.com, api.github.com (updates); assets.traycer.ai (wallpapers/preconnect); authn.traycer.ai, server.traycer.ai, traycer.ai, wss://relay.traycer.ai/attach (first-party cloud)",
  "ip_geo_handling": "PostHog omits ip: false and $geoip_disable: true (IP/GeoIP collected by us.i.posthog.com); Desktop and Mobile Sentry set sendDefaultPii: false and strip HTTP cookies/headers; CLI Sentry relies on @sentry/node ^10.76.2 default false",
  "custom_model_handling": "Clean: custom/private LLM model names are never sent to PostHog or Sentry; only closed AnalyticsHarness and AnalyticsProvider enum allowlists are permitted",
  "docs_discrepancies": "README and Mintlify docs state Sentry/PostHog 'may be enabled' and events link to accounts, but omit that release builds have zero user opt-out or DNT support, send plaintext email to PostHog without IP/GeoIP suppression, and ignore Privacy Mode for client telemetry",
  "privacy_rating": "D",
  "one_line_verdict": "Strict content/path redaction and minidump protections, but release builds force-enable PostHog and Sentry with zero user opt-out/DNT, send plaintext email + unmasked IP/GeoIP to PostHog, and leave traycer-host closed-source."
}
```
