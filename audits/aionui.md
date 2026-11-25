# Privacy & Telemetry Forensic Audit: AionUi

## Metadata

- **Slug:** `aionui`
- **Platform:** `AionUi`
- **Group:** `A`
- **Repository:** `https://github.com/iOfficeAI/AionUi` (`iOfficeAI/AionUi`)
- **Branch:** `main`
- **Commit SHA:** `e108b6b56b1a6eb99a00904878d0582f657b2908`
- **Commit Date:** `2026-11-05T05:31:17Z`
- **Commit Subject:** `docs: add architecture overview and correct stale contributor docs (#4366)`
- **License:** `Apache-2.0`
- **Status:** `Active`
- **Privacy Rating:** **`F` (Severe Red Flags)**
- **Audit Scope Boundary:** Static read-only audit of all source, build, installer, mobile, web, CI, and documentation files in `iOfficeAI/AionUi`. As documented in `docs/architecture/overview.md:15-20`, `packages/shared-scripts/src/prepare-aioncore.js:24-25`, and `package.json:264`, the core backend binary `aioncore` (`aioncoreVersion: "v0.2.2"`) is built from a separate external repository (`iOfficeAI/AionCore`) and downloaded precompiled into `resources/bundled-aioncore/` during packaging (`packages/desktop/electron-builder.yml:110-112`). Behavior inside the `aioncore` Rust binary itself is outside static source visibility here, though its log outputs and stdout/stderr are ingested and exfiltrated by the audited TypeScript desktop layer.

---

## 1. Default State & Opt-Out Mechanism

**Default State:** **ON by default** in official desktop releases. Release builds compile with `SENTRY_DSN` injected from GitHub Actions secrets into Vite's build-time `define` table (`.github/workflows/_build-reusable.yml:437, 530, 561`; `packages/desktop/electron.vite.config.ts:155, 318`) and into a generated Windows NSIS header (`scripts/build-with-builder.js:480-487`).

At runtime, four telemetry/network mechanisms activate automatically without consent or opt-in prompts:

1. **Main-process Sentry crash & exception capture:** `initSentry()` runs unconditionally at module import time at the very top of the Electron main entry point (`packages/desktop/src/index.ts:16`, `packages/desktop/src/sentry.ts:116-138`), before `--webui`, `--resetpass`, or `--version` CLI submode branches (`packages/desktop/src/index.ts:198-201, 696-700, 911-923`).
2. **Renderer-process Sentry SDK & automatic integrity failure telemetry:** `packages/desktop/src/renderer/main.tsx:10-33` initializes `@sentry/electron/renderer` whenever `window.electronAPI` is present, bridged over IPC by `packages/desktop/src/preload/main.ts:11`. `captureRuntimeInstallationIntegrityFailure()` sends Sentry error events automatically without prompting on runtime integrity failures (`packages/desktop/src/renderer/main.tsx:182-198`).
3. **Automatic 24-hour startup log upload (`startup-log-report`):** When the main window opens (`packages/desktop/src/index.ts:507`), `scheduleStartupLogReport(mainWindow)` waits 30 seconds after `did-finish-load` (`packages/desktop/src/sentry.ts:409, 551-565`) and calls `runStartupLogReport()`, which reads recent `.log` files from Electron's logs directory and backend `logs/` subdirectory, gzips up to 19 MiB, and transmits them as `aionui-logs.log.gz` via `Sentry.captureMessage('startup-log-report', 'info')` (`packages/desktop/src/sentry.ts:341, 408, 473-539`), throttled only to once per 24 hours via `userData/sentry-log-report-state.json` (`packages/desktop/src/sentry.ts:407, 410, 477-481, 534`).
4. **Automatic startup update check:** 3 seconds after `createWindow()`, the desktop app initializes `electron-updater` against `https://static.aionui.com/releases` and runs `checkForUpdatesAndNotify()` (`packages/desktop/src/index.ts:550-568`; `packages/desktop/src/process/services/autoUpdaterService.ts:134-143`; `packages/desktop/src/process/services/updateFeed.ts:10`).

**Opt-Out Controls & Bypasses:**
- **UI Opt-Out Controls:** **None.** Searching all settings components and configuration storage (`packages/desktop/src/renderer/components/settings/`, `packages/desktop/src/renderer/components/settings/SettingsModal/contents/AboutModalContent.tsx:1-233`, `packages/desktop/src/common/config/storage.ts`) finds zero telemetry, crash-reporting, analytics, or Sentry opt-out toggles.
- **Config / CLI / Env Opt-Out Controls for Telemetry:** **None.** Because `packages/desktop/electron.vite.config.ts:155, 318` statically replaces `process.env.SENTRY_DSN` via `JSON.stringify(process.env.SENTRY_DSN ?? '')` at compile time, packaged official binaries hardcode the DSN string literal and **ignore runtime `SENTRY_DSN=""` environment overrides** (`packages/desktop/src/sentry.ts:118, 485`). Only auto-update checks can be suppressed via environment variables (`AIONUI_DISABLE_AUTO_UPDATE=1`, `AIONUI_E2E_TEST=1`, `CI=true`/`1`, `GITHUB_ACTIONS=true` at `packages/desktop/src/index.ts:547-550`, or `IS_DISCONTINUED_BUILD=true` compile-time flag at `packages/desktop/electron.vite.config.ts:158` and `packages/desktop/src/index.ts:564`).
- **Do Not Track (`DNT`):** **Not honored.** Negative search confirms zero references to `doNotTrack`, `do_not_track`, or `navigator.doNotTrack`.
- **Silent Windows Installer Bypass:** During interactive Windows NSIS installer failures, `AIONUI_FAIL_UX` prompts the user with `MB_YESNO` defaulting to `IDNO` (`resources/windows/installer-errors-sentry.nsh:27-44`). However, when `${If} ${Silent}` is active (such as silent auto-update installs), the macro sets `$9 = "auto"` and unconditionally invokes `AIONUI_REPORT_TO_SENTRY_NOUI` without user interaction or consent (`resources/windows/installer-errors-sentry.nsh:23-24, 47-49, 87-98`), posting the full installer `.jsonl` log and installation ID directly to Sentry (`resources/windows/support/report-installer-failure.ps1:321-368`).
- **Headless / Web / Mobile Modes:** Running Electron with `--webui` or `--resetpass` still executes `initSentry()` (`packages/desktop/src/index.ts:16`) and `setSentryDeviceId()` (`packages/desktop/src/index.ts:718`), bypassing only `createWindow()` (`scheduleStartupLogReport` at line `507`) and `autoUpdaterService` (`packages/desktop/src/index.ts:911-992`). By contrast, the standalone Node packages (`packages/web-host/src/static-server.ts:1-60`, `packages/web-cli/src/index.ts:80-115`) and the Expo app (`mobile/package.json:1-73`, `mobile/app.config.ts:1-46`, `mobile/src/services/api.ts:1-44`) contain no Sentry SDK or outbound telemetry clients.

---

## 2. External Auth / Local Non-Workspace File Reads

**Clean on External Credential Harvesting:** Static search across `packages/`, `mobile/`, and `resources/windows/` confirms that AionUi's TypeScript/Electron/installer code **does not read other tools' auth or secret stores** (zero code reads of `~/.claude`, `~/.codex`, `~/.gemini`, `~/.cursor`, `~/.ssh`, `~/.aws` credentials; `~/.aws/config` appears solely as UI placeholder helper text for Bedrock profile naming in locale files such as `packages/desktop/src/renderer/services/i18n/locales/en-US/settings.json:431`).

**Observed Non-Workspace Local Reads/Writes (Non-Auth):**
- **Node/NVM PATH discovery (macOS/Linux):** On startup, `packages/desktop/src/index.ts:127-143` lists directory names in `process.env.NVM_DIR || path.join(HOME, '.nvm/versions/node')` to prepend existing `bin/` directories to `process.env.PATH`.
- **Shell profiles & Windows Registry PATH hydration (Windows):** On Windows startup (`packages/desktop/src/index.ts:145`), `hydrateWindowsProcessPath()` (`packages/desktop/src/process/startup/windowsPath.ts:259-274`) queries `HKCU\Environment` and `HKLM\SYSTEM\CurrentControlSet\Control\Session Manager\Environment` (`windowsPath.ts:11-12, 262-263`) and reads text from `%USERPROFILE%\Documents\{PowerShell,WindowsPowerShell}\{Microsoft.PowerShell_profile.ps1,profile.ps1}` and `%USERPROFILE%\{.bashrc,.bash_profile,.profile,.zshrc}` (`windowsPath.ts:214-235`) solely to extract PATH entries, plus checks fallback binary dirs (`.bun/bin`, `.cargo/bin`, `.local/bin`, Scoop, npm at `windowsPath.ts:237-257`).
- **Windows installer diagnostics & `handle.exe`:** On installer failure, `resources/windows/support/report-installer-failure.ps1:68-92` reads `%APPDATA%\AionUi\analytics.json` and `%LOCALAPPDATA%\AionUi\analytics.json`, and `Get-OptionalHandleDiagnostics` executes Sysinternals `handle.exe` (if found on `PATH`) targeting the locked install path (`report-installer-failure.ps1:165-239`) and embeds process handle output into the Sentry payload (`report-installer-failure.ps1:352`).
- **AionUi's own directories/files:** `packages/desktop/src/process/utils/configureChromium.ts:144-153` deletes legacy `~/.aionui-cdp-registry.json`; `packages/desktop/src/process/utils/utils.ts:98-120` creates `~/.aionui` / `~/.aionui-dev` and `~/.aionui-config` / `~/.aionui-config-dev` symlinks in non-packaged/config paths; `packages/web-cli/src/index.ts:97` defaults standalone CLI storage to `~/.aionui-web`; `packages/desktop/src/process/bridge/applicationBridgeCore.ts:41-50` resolves `os.homedir()`, `~/Desktop`, and `~/Downloads` paths for IPC consumers.
- **Self/Auto-Update & Startup Log Reads:** `packages/desktop/src/sentry.ts:490-511` reads all `.log` files in `app.getPath('logs')` and `logs/logs` automatically on launch for Sentry attachment; `packages/desktop/src/process/feedback/logs.ts:11, 171-193` reads `.log`, `.aioncore.log`, and `.aionrs.log` on manual feedback.

---

## 3. Persistent Device / Install IDs & Hashing

- **Persistent Installation UUID (`userData/analytics.json`):** `getOrCreateAnalyticsId()` in `packages/desktop/src/process/utils/analyticsId.ts:12-41` reads or generates a persistent `crypto.randomUUID()` stored in `path.join(app.getPath('userData'), 'analytics.json')` (`mode: 0o600`, lines `22, 34-36`).
- **Cross-Event Identity Binding:** In `packages/desktop/src/index.ts:718`, `handleAppReady()` invokes `setSentryDeviceId()` (`packages/desktop/src/sentry.ts:145-149`), setting both **`Sentry.setUser({ id })`** and **`Sentry.setTag('device_id', id)`** globally on the main Sentry scope. Every subsequent main-process crash, fatal backend startup failure, automatic 24-hour `startup-log-report`, and renderer crash (via Electron IPC scope sync) carries this stable UUID.
- **Installer Continuity:** The Windows failure reporter script `resources/windows/support/report-installer-failure.ps1:68-92, 327, 355-358` reads the exact same UUID from `%APPDATA%\AionUi\analytics.json` / `%LOCALAPPDATA%\AionUi\analytics.json` and attaches it as `user.id` and `tags.user_id` on Sentry installer-failure envelopes, plus generates a per-install 12-char GUID `$AionUiSessionId` (`resources/windows/installer-observability.nsh:86, 92`) attached as `tags.session` / `extra.installerSession`.
- **Contact Email Scoped Isolation (Manual Feedback):** In manual feedback submissions (`packages/desktop/src/renderer/services/feedback/submitFeedbackReport.ts:274-283`), an optional `contactEmail` (from `FeedbackReportModal.tsx:41-47, 94, 193`, populated only when the signed-in auth user object has `.email`) is set inside a temporary `Sentry.withScope(...)` via `scope.setUser({ email: contactEmail })` rather than mutating the global user scope. Note that setting `scope.setUser({ email })` in a child scope overwrites/augments the inherited `user` object for that event while `tags.device_id` remains on the global scope.
- **Hashing:** **None.** The UUID in `analytics.json` is stored and transmitted verbatim in plaintext (not salted or hashed). No hardware serials, MAC addresses, or OS machine IDs are hashed or read by TypeScript code (`os.hostname` / `machineId` negative).
- **24-Hour Report Throttle State:** `packages/desktop/src/sentry.ts:407-430` writes `lastReportAt` timestamps to `userData/sentry-log-report-state.json` locally.

---

## 4. Exact Outbound Telemetry / Analytics / Crash / OTEL / Relay / Update Endpoints

| Mechanism / Feature | Exact Outbound Endpoint / Host | Transport / SDK | Trigger & Frequency | Source Citation |
| --- | --- | --- | --- | --- |
| **Main & Renderer Crash / Error Telemetry** | Runtime value of compile-time `process.env.SENTRY_DSN` (injected from GitHub secret `secrets.SENTRY_DSN`; e.g., Sentry ingest `/api/<projectId>/envelope/`) | `@sentry/electron/main` (`^7.10.0`), `@sentry/electron/renderer`, `@sentry/electron/preload` | **Automatic on startup** (`initSentry` + `Sentry.init`); fires on uncaught exceptions, renderer crashes, `runtime-installation-integrity-failure`, and fatal `captureBackendStartupFailure` | `package.json:106`; `packages/desktop/src/index.ts:16, 847, 851, 869`; `packages/desktop/src/sentry.ts:116-138, 238-333`; `packages/desktop/src/preload/main.ts:11`; `packages/desktop/src/renderer/main.tsx:10-33, 182-198`; `.github/workflows/_build-reusable.yml:437, 530, 561` |
| **Automatic Startup Log Exfiltration (`startup-log-report`)** | Same compile-time `process.env.SENTRY_DSN` Sentry project endpoint | `@sentry/electron/main` (`Sentry.captureMessage('startup-log-report', 'info')` + `aionui-logs.log.gz`) | **Automatic 30s after first `did-finish-load`**, once every 24h | `packages/desktop/src/index.ts:507`; `packages/desktop/src/sentry.ts:341, 407-565` |
| **User-Initiated Bug Report / Integrity Diagnostics** | Same compile-time `process.env.SENTRY_DSN` Sentry project endpoint | `@sentry/electron/renderer` (`Sentry.captureEvent`) | Manual user action (`FeedbackReportModal` / `InstallationIntegrityDialog` / `FeedbackButton`) | `packages/desktop/src/renderer/services/feedback/submitFeedbackReport.ts:272-313`; `FeedbackReportModal.tsx:180-199`; `InstallationIntegrityDialog.tsx:149-165` |
| **Windows Installer Failure Envelope** | `$uri.Scheme://$uri.Authority/api/$projectId/envelope/` parsed from build-time `_sentry-dsn.generated.nsh` (`AIONUI_SENTRY_DSN`) | PowerShell `Invoke-RestMethod` (`application/x-sentry-envelope`) | On installer failure: **automatic without UI** when `${Silent}`, or `MB_YESNO` (default No) when interactive | `resources/windows/installer-errors-sentry.nsh:4, 23-52, 91-95`; `resources/windows/support/report-installer-failure.ps1:321-368`; `scripts/build-with-builder.js:480-487` |
| **Auto-Update Manifest & Binary CDN** | `https://static.aionui.com/releases` (channel `.yml` manifests e.g. `latest.yml`, `latest-mac.yml`, `latest-arm64-mac.yml`, `latest-linux.yml`, installers) | `electron-updater` (`^6.6.2`, `CdnGenericProvider`) & Node `fetch` (`User-Agent: AionUi`) | **Automatic 3s after `createWindow()`** unless env-disabled or discontinued build; also on manual update check | `packages/desktop/src/index.ts:547-568`; `packages/desktop/src/process/services/updateFeed.ts:10`; `autoUpdaterService.ts:134-143`; `packages/desktop/src/process/bridge/updateBridge.ts:64-74, 358-390` |
| **Manual Update Changelog & Fallback Downloads** | `https://api.github.com/repos/iOfficeAI/AionUi/releases` + allowlisted download hosts (`static.aionui.com`, `github.com`, `objects.githubusercontent.com`, `github-releases.githubusercontent.com`, `release-assets.githubusercontent.com`) | `fetch` (`User-Agent: AionUi`) | Manual update check in UI | `packages/desktop/src/process/bridge/updateBridge.ts:68-74, 270-348, 399-413` |
| **Source Maps Upload (CI Build-Time Only)** | Sentry API configured via `SENTRY_ORG`, `SENTRY_PROJECT`, `SENTRY_AUTH_TOKEN` | `@sentry/vite-plugin` (`^5.1.1`) | CI build-time when `SENTRY_UPLOAD_SOURCE_MAPS === 'true'` (`deleteAfterUpload` `./out/**/*.map`) | `package.json:187`; `packages/desktop/electron.vite.config.ts:70-95, 133, 243`; `.github/workflows/_build-reusable.yml:438-440` |
| **User-Configured LLM Requests Headers** | User-configured OpenAI-compatible provider `base_url` | `OpenAIRotatingClient` (`openai`) | Every LLM API call on OpenAI / fallback protocols (not AionUi servers, but identifies client domain) | `packages/desktop/src/common/api/ClientFactory.ts:78-81, 129-132` (`HTTP-Referer: https://aionui.com`, `X-Title: AionUi`) |
| **Static Links / User-Opened Pages** | `https://www.aionui.com`, `https://github.com/iOfficeAI/AionUi/*`, `https://x.com/WailiVery` | External browser via user click only | Only on explicit user click in About / Integrity modals | `AboutModalContent.tsx:112-138, 165`; `InstallationIntegrityDialog.tsx:7, 37-39` |
| **Prebuilt Backend Subprocess (`aioncore`)** | **Opaque / Unverified** (binary downloaded from `https://api.github.com/repos/iOfficeAI/AionCore/...` during build) | Prebuilt binary | Unknown runtime network behavior beyond spawning local HTTP server on `127.0.0.1` | `packages/shared-scripts/src/prepare-aioncore.js:24-25, 178, 302, 376`; `docs/architecture/overview.md:15-20` |

---

## 5. Exact Data & Properties Collected

### A. IP Address & Geolocation Handling
- **`sendDefaultPii`:** **Not set** (omitted from both `Sentry.init` calls in `packages/desktop/src/sentry.ts:117-133` and `packages/desktop/src/renderer/main.tsx:14-30`).
- **IP / Geo Suppression:** **None.** Because AionUi does not use PostHog, there is no `$geoip_disable` or `$ip` property (`rg` negative), and neither Sentry `beforeSend` hook (`packages/desktop/src/sentry.ts:120-132`; `packages/desktop/src/renderer/main.tsx:15-29`) nor the PowerShell NSIS uploader (`report-installer-failure.ps1:367`) masks or suppresses the client's connecting source IP at ingest. Standard HTTPS connections to Sentry and `static.aionui.com` therefore expose the client's public IP address unless Sentry server-side project settings strip IPs (unverifiable from client source).

### B. Custom / Private Model Names & Provider Configuration
- **Not Scrubbed in Automatic Daily Log Uploads:** `runBackendMigrations.ts:106-114` logs `result.provider.id`, `result.provider.platform`, `result.model`, and `api key present: yes/no` via `console.info`, which `configureConsoleLog.ts:109-115` persists to the daily `.log` file that `runStartupLogReport()` automatically gzips and ships to Sentry every 24 hours without redaction (`packages/desktop/src/sentry.ts:507-532`).
- **Included in Manual Feedback DB Diagnostics:** Per `docs/prds/settings/feedback-db-diagnostics.md:60-83` (requested by `submitFeedbackReport.ts:157, 178-192` via `GET /api/system/diagnostics/feedback-report`), manual bug reports attach `db-diagnostics.json.gz` containing current and recent (up to 20 in 24h) `provider_id`, `model_id`, `provider.platform`, `provider.name`, `base_url` host, and conversation titles/IDs.

### C. Prompts, Code, File Paths, Stack Traces & Redaction Gaps
- **Automatic Daily Logs (`startup-log-report`) — UNREDACTED:** `runStartupLogReport()` (`packages/desktop/src/sentry.ts:505-532`) reads raw `.log` files directly from `fs.readFileSync(f.path, 'utf8')` and uploads up to **19 MiB gzipped** (`aionui-logs.log.gz`, line `408, 525`) with **zero redaction or PII/secret filtering**. Main-process `electron-log` (`configureConsoleLog.ts:68-115`) captures at `info` level:
  - Full local OS filesystem paths including usernames (e.g., `[aioncore] starting: ${binaryPath} ...` and `data-dir: ${dbPath}` in `packages/web-host/src/backend-launcher.ts:676, 894, 898`; `Loading renderer file: ${fallbackFile}` in `packages/desktop/src/index.ts:590`).
  - All `aioncore` backend stdout/stderr lines prefixed with `[aioncore]` (`backend-launcher.ts:830, 837`; `configureConsoleLog.ts:98-107`).
  - Renderer logs forwarded over IPC (`[Renderer:${tag}] ${message}`, `data` in `packages/desktop/src/process/bridge/applicationBridge.ts:162-174`).
  - *Prompt dump safety check:* `--dump-prompts` via `AIONUI_DUMP_PROMPTS=1` is gated on `!config.isPackaged` (`packages/web-host/src/backend-launcher.ts:210`), preventing prompt dump flags in packaged builds, though raw conversation IDs, errors, paths, and model strings in logs still upload unredacted.
- **Fatal Backend Startup & Uncaught Exceptions:** `captureBackendStartupFailure()` (`packages/desktop/src/sentry.ts:238-333`) attaches full stack traces (`Sentry.captureException`), tags (`aionui.backend_startup.*`), `autoUpdateDiagnostics` (`auto-update-diagnostics.json`), and `BackendInstallDiagnostics` including verbatim `execPath`, `resourcesPath`, `binaryPath`, `bundledDirPath`, `runtimeDirPath`, `manifestFiles`, file sizes/mtimes, and `appVersion` (`packages/desktop/src/process/startup/backendInstallDiagnostics.ts:26-52`; `sentry.ts:292, 315-325`).
- **Windows Installer Reports (`report-installer-failure.ps1`):** Uploads the full unredacted `.jsonl` installer session log (`$logText` at `report-installer-failure.ps1:325, 363`), `$INSTDIR` / `$AionUiSessionLogPath` (which include Windows username paths under `%LOCALAPPDATA%` or `%TEMP%`), installer failure codes, `blockingDiagnostics`, and optional `handle.exe` process handle output (`report-installer-failure.ps1:165-239, 345-353`).
- **Scoped Redaction Only on Specific Manual Paths:** `redactErrorText()` (`packages/desktop/src/renderer/pages/conversation/platforms/acp/errorDiagnostics.ts:21-97`) strips `sk-`, `AIza`, AWS `AKIA`/`ASIA` keys, JWTs, Bearer tokens, URL passwords, `api_key`/`token`/`password`/`secret` pairs, emails, and `/Users/<user>`, `/home/<user>`, `C:\Users\<user>` segments—but **only** for `buildSendFailureError.ts:111` (`rawError` in `MessageTips.tsx:140`), NOT for `Sentry.beforeSend`, `startup-log-report`, `feedbackBridge` `logs.gz`, or `report-installer-failure.ps1`. Meanwhile, `feedback-db-diagnostics.md:135` explicitly notes conversation titles in `db-diagnostics.json.gz` are **not** redacted for `sk-...`, `token`, or `bearer` patterns, and `FeedbackReportModal.tsx:162-178` attaches user-included or auto-captured window screenshots (`feedbackBridge.ts:76-94`) and raw 3-day `logs.gz` (`feedback/logs.ts:11, 171-193`).

### D. Behavioral & Environment Profile Fields
- Global Sentry tags (`packages/desktop/src/sentry.ts:135-137, 147-148`): `app.arch`, `app.version`, `os.name`, `device_id` (`analytics.json` UUID), `user.id` (`analytics.json` UUID), `environment` (`production` vs `development`).
- Startup failure tags & buckets (`packages/desktop/src/sentry.ts:253-313`): `install_path_kind` (`user_local_programs` / `program_files` / `mac_applications` / `linux_opt` / `custom`), Rosetta translation status (`rosetta_translated`), `package_arch` vs `device_arch`, health polling timing/attempt buckets, `last_update_status`, `seconds_since_quit_and_install`.
- Runtime integrity failures (`packages/desktop/src/renderer/main.tsx:189-195`): `aionui.installation_integrity`, `aionui.runtime_resource`, `aionui.runtime_resource_id`, `aionui.runtime_scope`.

---

## 6. Docs / Privacy Policy vs Code Discrepancies

1. **Direct Material Misrepresentation in English & Localized READMEs:**
   - `readme.md:579-580` (`Q: Is my data secure?`) states verbatim:
     > `A: All data is stored locally in a SQLite database. Nothing is uploaded to any server.`
   - The identical absolute claim appears across the translated READMEs under `docs/readme/` (`docs/readme/readme_ch.md:575`, `docs/readme/readme_tw.md:575`, `docs/readme/readme_jp.md:572`, `docs/readme/readme_es.md:572`, `docs/readme/readme_ko.md:572`, `docs/readme/readme_pt.md:572`, `docs/readme/readme_tr.md:572`, and `docs/readme/readme_ru.md:586` *"AionUi не загружает их на сторонние серверы"*).
   - **Code Reality:** Official desktop builds automatically assign a persistent UUID (`analytics.json`), send crash/startup/integrity telemetry to Sentry, upload up to 19 MiB of **unredacted** daily frontend+backend log files every 24 hours (`startup-log-report` in `packages/desktop/src/sentry.ts:473-539`), silently upload Windows installer logs during silent updates (`resources/windows/installer-errors-sentry.nsh:23-24, 48-49`), and query `https://static.aionui.com/releases` on every launch (`packages/desktop/src/index.ts:565-567`).
2. **Absence of Privacy Policy & Undisclosed Always-On Tracking:**
   - The repository contains **no `PRIVACY.md` or privacy disclosure** describing Sentry crash collection, `analytics.json` device tracking, or the 24-hour `startup-log-report` uploader. Even UI disclosure (`bugReportAutoInfo` at `packages/desktop/src/renderer/services/i18n/locales/en-US/settings.json:839`: *"System info and logs from the last 3 days will be automatically attached..."*) only informs users when manually opening the Bug Report modal, leaving background `startup-log-report` and `analytics.json` UUID tracking completely undisclosed.
   - By contrast, internal contributor documentation explicitly acknowledges tracking (`CONTRIBUTING.md:29`: *"Team chat scroll fix + Sentry user tracking + office preview performance optimization = 3 PRs"* and `CONTRIBUTING.zh.md:29`: *"Sentry 用户追踪"*), confirming maintainer awareness while public user docs still state *"Nothing is uploaded to any server."*
3. **Closed/External Backend Boundary (`aioncore`):**
   - While `readme.md:570` describes AionUi as *"completely free and open source"*, `docs/architecture/overview.md:15-20` notes `aioncore` is *"built from a separate repository and bundled into the installer"*, and `packages/shared-scripts/src/prepare-aioncore.js:24-25` fetches prebuilt binaries from `iOfficeAI/AionCore`.

---

## 7. Zero-Telemetry / Negative-Check Verification

**Not Zero-Telemetry** (`desktop` main, preload, renderer, and Windows NSIS installer actively transmit telemetry to Sentry and CDN update endpoints). However, negative `rg` checks across `packages/`, `mobile/`, `resources/`, `scripts/`, `.github/`, and `package.json` verify the **absence** of other third-party analytics/telemetry SDKs, DNT checks, and external-tool credential reads:

- **Third-Party Analytics / Telemetry / Crash SDKs & DNT Negative Check (`rg -n -i` word-boundary check across `packages/`, `mobile/`, `resources/`, `scripts/`, `.github/`, `package.json`):**
  - Pattern tested: `(\bposthog\b|\bmixpanel\b|\bamplitude\b|\bumami\b|\bplausible\b|segment\.io|\bgtag\b|google-analytics|\bmatomo\b|\bbugsnag\b|\brollbar\b|\bhoneycomb\b|\bdatadog\b|\bopentelemetry\b|\bcrashlytics\b|expo-insights|geoip_disable|doNotTrack|do_not_track|sendDefaultPii|navigator\.doNotTrack)`
  - **Result:** Exit code `1` (**0 matches**). Only `@sentry/electron` (`^7.10.0`), `@sentry/vite-plugin` (`^5.1.1`), `electron-updater` (`^6.6.2`), and direct Sentry `/envelope/` HTTP POSTs in PowerShell exist.
- **Settings UI & Config Opt-Out Negative Check (`rg -n -i "(sentry|telemetry|analytics|opt[-_ ]?out|crash[-_ ]?report|do[-_ ]?not[-_ ]?track)"` on `packages/desktop/src/renderer/components/settings` and `packages/desktop/src/common/config/storage.ts`):**
  - **Result:** Exit code `1` (**0 matches** — no user-facing telemetry control exists).
- **External Auth Credential File Reads Negative Check (`rg -n "(\.claude|\.codex|\.ssh|\.aws|\.gemini|\.cursor|os\.homedir|homedir\(\)|USERPROFILE)"` across `packages/`, `mobile/`, `resources/windows/`):**
  - **Result:** **0 code reads** of external agent/cloud/SSH auth directories (`~/.claude`, `~/.codex`, `~/.ssh`, `~/.aws`, `~/.gemini`, `~/.cursor`); `.aws/config` matches are solely AWS Bedrock UI placeholder strings in `settings.json` locale files (`en-US/settings.json:431`), and `os.homedir()`/`USERPROFILE` usage is confined to `~/.aionui-web` (`packages/web-cli/src/index.ts:97`), `.aionui-cdp-registry.json` cleanup (`configureChromium.ts:146`), Windows shell-profile PATH parsing (`windowsPath.ts:215, 238`), and `applicationBridgeCore.ts:43`.
- **Clean Sub-Packages:** `packages/web-host/src/static-server.ts`, `packages/web-cli/src/index.ts`, and `mobile/` (`mobile/package.json:1-73`, `mobile/app.config.ts:1-46`, `mobile/src/services/api.ts:1-44`) contain zero telemetry SDKs and communicate only with the user's own `aioncore` backend instance (except when `packages/desktop/src/index.ts` itself is launched with `--webui`, which initializes main-process Sentry).

---

```json
{
  "slug": "aionui",
  "platform": "AionUi",
  "group": "A",
  "repo": "https://github.com/iOfficeAI/AionUi",
  "sha": "e108b6b56b1a6eb99a00904878d0582f657b2908",
  "commit_date": "2026-11-05T05:31:17Z",
  "license": "Apache-2.0",
  "status": "Active",
  "default_state": "ON by default (CI-baked SENTRY_DSN enables crash capture, persistent device UUID, 24h automatic 19 MiB unredacted log upload via startup-log-report, silent Windows NSIS failure upload, and 3s startup CDN update checks)",
  "opt_out_controls": "None for telemetry/Sentry (no Settings UI toggle, no config key, no CLI flag, and Vite define inlines SENTRY_DSN at compile time so runtime env override fails; auto-updater only skips if AIONUI_DISABLE_AUTO_UPDATE=1, AIONUI_E2E_TEST=1, CI=true, or compile-time IS_DISCONTINUED_BUILD=true)",
  "do_not_track": false,
  "opt_out_bypasses": [
    "No user opt-out mechanism exists anywhere in UI, config, or CLI",
    "Vite build-time define hardcodes process.env.SENTRY_DSN literal in main & renderer bundles",
    "Automatic 24h startup-log-report gzips and uploads unredacted daily frontend + backend logs (up to 19 MiB) silently 30s after launch",
    "Windows NSIS installer ${Silent} mode (background auto-update) auto-executes AIONUI_REPORT_TO_SENTRY_NOUI without user prompt",
    "Electron --webui and --resetpass CLI submodes still run initSentry() and setSentryDeviceId()"
  ],
  "persistent_ids": [
    "userData/analytics.json random UUIDv4 bound to global Sentry.setUser({ id }) and tags.device_id",
    "Windows %APPDATA%/AionUi/analytics.json & %LOCALAPPDATA%/AionUi/analytics.json read by installer PowerShell reporter",
    "Per-install 12-char GUID $AionUiSessionId in Windows NSIS installer reports"
  ],
  "hashing": "None (random UUID stored and sent plaintext; no hardware ID or email hashing)",
  "external_auth_reads": "None in TypeScript/Electron/NSIS code (clean on ~/.claude, ~/.codex, ~/.ssh, ~/.aws, ~/.gemini, ~/.cursor auth files; non-auth reads limited to ~/.nvm/versions/node on Unix and Windows PATH registry keys + PowerShell/bash/zsh profiles for PATH hydration; prebuilt aioncore Rust binary source is external in iOfficeAI/AionCore)",
  "telemetry_endpoints": [
    "Compile-time SENTRY_DSN (Sentry ingest /api/<projectId>/envelope/ via @sentry/electron and NSIS PowerShell Invoke-RestMethod)",
    "https://static.aionui.com/releases (auto-updater & manual update channel manifests + binaries)",
    "https://api.github.com/repos/iOfficeAI/AionUi/releases (manual update release notes enrichment & fallback asset downloads)",
    "HTTP-Referer: https://aionui.com & X-Title: AionUi headers sent to user-configured OpenAI-compatible LLM endpoints"
  ],
  "ip_geo_handling": "sendDefaultPii not set; no $geoip_disable/$ip or beforeSend IP scrubbing; direct HTTPS connections to Sentry and static.aionui.com expose client IP to ingest endpoints",
  "custom_model_handling": "Unredacted model/platform/provider IDs logged by runBackendMigrations.ts are shipped automatically in 24h startup-log-report logs; manual feedback db-diagnostics.json.gz also includes provider/model IDs and unredacted conversation titles",
  "docs_discrepancies": [
    "readme.md:580 and 8+ translated docs/readme/* files state 'All data is stored locally in a SQLite database. Nothing is uploaded to any server.' while Sentry crash capture, persistent device UUID, 24h startup-log-report, silent NSIS failure reports, and startup CDN checks run by default",
    "No privacy policy or user disclosure for analytics.json UUID tracking or automatic 24h startup-log-report uploads, despite CONTRIBUTING.md:29 referencing 'Sentry user tracking'",
    "aioncore backend binary is built from external repo iOfficeAI/AionCore rather than source in this repository"
  ],
  "privacy_rating": "F",
  "one_line_verdict": "Ships always-on Sentry tracking with a persistent UUID and automatic unredacted 19 MiB daily log uploads with zero opt-out controls, while falsely claiming across its README that 'Nothing is uploaded to any server.'"
}
```
