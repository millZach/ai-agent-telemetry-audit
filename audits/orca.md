# Forensic Telemetry & Privacy Audit: Orca (`stablyai/orca`)

- **Slug:** `orca`
- **Platform:** Orca (`stablyai/orca`)
- **Group:** `A`
- **Repository:** `https://github.com/stablyai/orca`
- **License:** `MIT` (`README.md:340`)
- **Status:** Active (`main`)
- **Audited SHA:** `14b04d30f631a5e49f2e649ce2f8ebb6832ddb6b` (`2026-11-25T04:55:09Z`, `feat(agents): offer agent CLI updates in a bottom-right card (#48610)`)
- **Privacy Rating:** **`C`** (Moderate / Mixed — Strong Schema & GeoIP Hygiene with Opt-Out/Banner Flaws: strict Zod `.strict()` allow-lists with zero model names or prompts, `disableGeoip: true`, `$process_person_profile: false`, `DO_NOT_TRACK`/`ORCA_TELEMETRY_DISABLED`/`CI` overrides, local-only Crashpad/trace files, and clean Mobile/Docs apps; held back from `B` by ON-by-default official builds without install-time onboarding disclosure, pre-opt-out `telemetry_opted_out` transmission before calling `client.optOut()`, `FirstLaunchBanner` close `X` button silently persisting opt-in and firing `app_opened`, persistent `installId` retained across opt-out, and material `telemetry.mdx` discrepancies on `main_thread_hang_detected` redacted native stack frames and Orca Cloud enterprise machine hostnames)

---

## 1. Default State & Opt-Out Mechanism

### Default State
- **Compile-time official build gating:** Telemetry transmission only activates in official CI builds where `.github/workflows/release-cut.yml:1227,1231` injects `ORCA_BUILD_IDENTITY` (`stable` or `rc`) and `ORCA_POSTHOG_WRITE_KEY`, which `electron.vite.config.ts:30-46,228-229` compiles into main-process constants (`src/main/telemetry/client.ts:35,38-49,77-79`). Contributor, local dev, and PR builds compile `ORCA_BUILD_IDENTITY` and `ORCA_POSTHOG_WRITE_KEY` to literal `null` (`electron.vite.config.ts:36,41`) and fail closed (`src/main/telemetry/client.ts:77-79,141-143`).
- **Fresh installs (`ON by default`):** On first load when the persisted profile/settings file did not exist (`fileExistedOnLoad === false`), `migrateTelemetryIdentity` generates `installId: randomUUID()`, sets `existedBeforeTelemetryRelease: false`, and defaults `optedIn` to **`true`** (`src/main/persistence/loading-store/profile-loading/loaded-cohort-migrations.ts:192-215`). Secondary profiles initialized via `initializeNewOrcaProfileState` (`src/main/orca-profiles/storage/new-profile-state.ts:35-48`) likewise start at `optedIn: true` unless inheriting an existing profile's consent.
- **No install-time telemetry disclosure in Onboarding:** Despite code comments in `src/renderer/src/components/onboarding/telemetry/TelemetryFirstLaunchSurface.tsx:1` and `src/renderer/src/components/onboarding/telemetry/FirstLaunchBanner.tsx:1` stating *"new users receive install-time disclosure"*, `OnboardingFlow.tsx` and the onboarding steps contain no telemetry notice or opt-out toggle (confirmed across `src/renderer/src/components/onboarding/` and `src/renderer/src/app-shell/AppRootSurfaces.tsx:316-318`). New users are silently opted in (`optedIn: true`) and must navigate post-hoc to `Settings -> Privacy & Telemetry`.
- **Pre-telemetry existing users (`pending_banner`):** Upgraded profiles where `fileExistedOnLoad === true` receive `existedBeforeTelemetryRelease: true` and `optedIn: null` (`src/main/persistence/loading-store/profile-loading/loaded-cohort-migrations.ts:205-210`). While `optedIn === null`, `resolveConsent` returns `{ effective: 'pending_banner' }` (`src/main/telemetry/consent.ts:88`), causing `track()` (`src/main/telemetry/client.ts:194-197`) and `isTelemetryEnabled()` (`src/main/telemetry/client.ts:129-137`) to reject product events—**until** the user clicks any button on `FirstLaunchBanner` (see bypasses below).
- **Diagnostics & Crash Reporting (Separate Channels):**
  - Local OpenTelemetry-shaped tracing (`initObservability` in `src/main/observability/index.ts:63-96`) writes local rotated NDJSON logs (`localFileEnabled: true`) unless disabled by `CI` or `ORCA_DIAGNOSTICS_DISABLED=1`/`true`. Upload to Orca Support is **explicitly manual only** via a mandatory preview-before-upload workflow in `Settings -> Privacy & Telemetry` (`src/renderer/src/components/settings/privacy/PrivacyDiagnosticsSection.tsx:58-149`; `src/main/ipc/diagnostics.ts:101-105`).
  - Crashpad is initialized with `uploadToServer: false` (`src/main/crash-reporting/capture/crashpad-capture.ts:51-57`); crash reports are only sent when a user explicitly submits `CrashReportDialog` (`src/renderer/src/components/crash-report/CrashReportDialog.tsx:28-51`; `src/main/ipc/crash-reporting/crash-reporting-submission.ts:43-120`).
- **Mobile & Docs:** Both `mobile/` (`mobile/package.json:25-84`) and `docs/site/` (`docs/site/package.json:14-26`) have zero analytics or telemetry SDKs.

### Opt-Out Controls & Precedence
Consent resolution (`resolveConsent` in `src/main/telemetry/consent.ts:69-89`) applies the following hierarchy:
1. **`DO_NOT_TRACK` env var (`1` or `true`, case-insensitive):** Disables PostHog telemetry (`{ effective: 'disabled', reason: 'do_not_track' }`, `src/main/telemetry/consent.ts:37-53`) AND disables diagnostic bundle upload in `resolveObservabilityConsent()` (`{ localFileEnabled: true, bundleEnabled: false }`, `src/main/observability/index.ts:83-90`). Does not mutate stored preferences (`src/main/telemetry/consent.ts:1`).
2. **`ORCA_TELEMETRY_DISABLED` env var (`1` or `true`):** Identical semantics to `DO_NOT_TRACK` (`src/main/telemetry/consent.ts:54-56`; `src/main/observability/index.ts:83-90`).
3. **CI environment variables (`CI`, `GITHUB_ACTIONS`, `GITLAB_CI`, `CIRCLECI`, `TRAVIS`, `BUILDKITE`, `JENKINS_URL`, `TEAMCITY_VERSION` non-empty):** Disables PostHog telemetry (`src/main/telemetry/consent.ts:11-20,58-60`) and disables both local diagnostic logging and bundle uploads (`src/main/observability/index.ts:29-38,69-75`).
4. **Additional diagnostic kill switch (`ORCA_DIAGNOSTICS_DISABLED=1`/`true`):** Disables both local diagnostic NDJSON tracing and diagnostic bundle uploads (`src/main/observability/index.ts:66,76-82`).
5. **In-App UI Toggle:** `Settings -> Privacy & Telemetry -> Share anonymous usage data` (`src/renderer/src/components/settings/navigation/settings-navigation-remote-sections.ts:84`; `src/renderer/src/components/settings/privacy/PrivacyPane.tsx:105-133`), which sets `settings.telemetry.optedIn = false` via IPC `telemetry:setOptIn` (`src/main/ipc/telemetry.ts:121-140`). Opt-out mutations bypass `consumeConsentMutationToken()` rate-limiting (`src/main/ipc/telemetry.ts:133-136`). When `DO_NOT_TRACK`, `ORCA_TELEMETRY_DISABLED`, or a CI variable is active, the toggle is disabled and a helper explains the env override (`PrivacyPane.tsx:77,130,141-165`).

### Opt-Out Leaks, Bypasses & Consent-UI Quirks
1. **Pre-opt-out `telemetry_opted_out` beacon sent BEFORE `client.optOut()` (`src/main/telemetry/client.ts:257-292`; `src/main/telemetry/opt-out-capture.ts:10-33`; `src/main/telemetry/posthog-transport.ts:70-73`):**
   - `shouldOptOutSdkAtInit(consent)` only calls `client.optOut()` at startup when `consent.effective === 'disabled'`—deliberately keeping `pending_banner` (`optedIn === null`) **un-opted-out** *"so the direct Turn Off capture can enqueue before optOut"* (`posthog-transport.ts:70-73`).
   - Whenever a user turns telemetry OFF (whether clicking **"Opt out"** on `FirstLaunchBanner` from `pending_banner` or toggling OFF in `Settings -> Privacy & Telemetry`), `setOptIn(via, false)` calls `ensurePostHogClient()` first (`client.ts:262`), and unless an env override (`DO_NOT_TRACK`/`ORCA_TELEMETRY_DISABLED`/`CI`) is active or shutting down, calls `captureOptOutBeforeSdkOptOut(client, commonProps, via)` (`client.ts:279-289`), which directly calls `client.capture({ distinctId: common.install_id, event: 'telemetry_opted_out', ... })` (`opt-out-capture.ts:24-29`)—transmitting `install_id`, `session_id`, `app_version`, `platform`, `arch`, `os_release`, `orca_channel`, A/B cohort flags, and `via` (`'first_launch_banner'` or `'settings'`) to `https://us.i.posthog.com` **before** `await client.optOut()` (`client.ts:289`). For pre-telemetry users who had sent **zero** events while in `pending_banner`, clicking "Opt out" initiates PostHog and transmits a network event carrying their `install_id`.
2. **`FirstLaunchBanner` close (`X`) button silently opts in (`src/renderer/src/components/onboarding/telemetry/FirstLaunchBanner.tsx:112-121`):**
   - Clicking the top-right `X` (`aria-label="Dismiss notice"`) invokes `handleAcknowledge` (`FirstLaunchBanner.tsx:30-48,116`), calling `acknowledgeBanner()` -> `persistBannerAcknowledgeWithoutEmitting()` (`src/main/telemetry/client.ts:295-309`), which writes `optedIn: true`, calls `posthog.optIn()`, and immediately fires `app_opened` (`trackAppOpenedOnce()`). Notice comment at line 112 explicitly acknowledges: `// Dismiss persists opt-in; it does more than hide the notice.`
3. **Held events & delayed capture replay:** Prior to `runAfterFirstWindowShown`, `deferStart` holds captures in memory (`src/main/telemetry/posthog-client-start.ts:16-38`); `takeStart` (`:85-94`) and `hasSendableHeldCaptures` (`:56-58`) re-check `resolveConsent(...) === 'enabled'` and drop held events if consent was withdrawn before client start, though `setOptIn(via, false)` calls `ensurePostHogClient()` *before* updating settings (`client.ts:262-268`), meaning any events held during the current enabled session before opt-out are replayed and flushed alongside `telemetry_opted_out`.
4. **Once opted out (or env-disabled):** `track()` returns `false` before queueing (`client.ts:193-197`), `posthog.optOut()` suppresses SDK captures, and telemetry-only system probes (`WslUsageReporter` at `src/main/telemetry/wsl-usage-reporter.ts:252-254`, `runTccDenialCapture` at `src/main/daemon/permissions/tcc-denial-capture.ts:80-84,117`, `AgentTokenUsageReporter` at `src/main/usage/reporting/token-usage/agent-token-usage-reporter.ts:89-91`, and `assignBrowserExtensionsBanner` at `src/main/telemetry/browser-extensions-banner.ts:61`) cleanly abort before probing disk/registry/keychains.

---

## 2. External Auth / Local Non-Workspace File Reads

Orca reads an extensive set of external agent credentials, configs, OS keychains, browser cookie databases, and host machine identifiers for **product functionality** (local account hot-swapping, rate-limit/quota meters, browser cookie import, and theme import). Crucially, **our audit found zero evidence of external tool credentials or third-party account identities being harvested into PostHog telemetry or Orca Cloud.** Specific reads and their exact data destinations:

1. **Claude Code OAuth credentials & macOS Keychain (`src/main/rate-limits/providers/claude/claude-oauth-credentials.ts:118-145`; `src/main/claude-accounts/runtime-auth/keychain.ts:7-45`):**
   - Reads macOS generic-password Keychain items `Claude Code-credentials` / `Claude Code-credentials-<sha256_8>` (`keychain.ts:7,38-45`) and `~/.claude/.credentials.json` (or `$CLAUDE_CONFIG_DIR/.credentials.json`, `claude-oauth-credentials.ts:130-136`). Used locally for account switching and direct Anthropic usage-limit API polling while the usage view/window is active (`src/main/rate-limits/service/service-polling.ts:56-69`).
2. **OpenAI Codex credentials (`src/main/rate-limits/providers/codex/codex-backend-auth.ts:62-90`):**
   - Reads `~/.codex/auth.json` (or `$CODEX_HOME/auth.json`, extracting `tokens.access_token` and `tokens.account_id`) solely to send `Authorization` + `ChatGPT-Account-Id` headers directly to OpenAI Codex usage/backend endpoints and manage Codex hot-swap profiles.
3. **Cursor, Grok, Kimi, ZCode credentials:**
   - **Cursor (`src/main/rate-limits/providers/cursor/cursor-auth-paths.ts:6-33`):** Reads macOS `~/Library/Application Support/Cursor/User/globalStorage/state.vscdb`, `~/.cursor/auth.json`, `~/.cursor/cli-config.json`, or Windows `%APPDATA%\Cursor\...` / Linux XDG equivalents + OS keychain for Cursor usage polling.
   - **Grok (`src/main/rate-limits/providers/grok/grok-auth.ts:5-70`):** Reads `~/.grok/auth.json` (`key`, `user_id`, `email`, `team_id`, `oidc_client_id`) for Grok quota meters; comment at `:42` explicitly prohibits exposing filesystem paths/usernames to renderer/mobile state.
   - **Kimi (`src/main/rate-limits/providers/kimi/kimi-fetcher.ts:21-80`):** Reads local/WSL Kimi `credentials/*.json` (`access_token`) for Kimi usage API queries.
   - **ZCode (`src/main/zcode/zcode-v2-plan-credentials.ts:51-131`):** Reads `~/.zcode/v2/provider_config.json` and `~/.zcode/v2/credentials.json` (decrypting `enc:v1:` AES-256-GCM secrets using `ZCODE_CREDENTIAL_SECRET` or fallback key) for Z.ai/BigModel coding-plan quota meters.
4. **Browser cookies (`src/main/browser/cookies/sources/browser-cookie-detection.ts:14-80`):**
   - Detects Safari `Cookies.binarycookies`, Chromium profile `Cookies` DBs (with Keychain decryption metadata), and Firefox profile roots for explicit user-initiated cookie import into Orca's embedded browser.
5. **Ghostty terminal config (`src/main/ghostty/discovery.ts:5-69`; `src/renderer/src/components/onboarding/ThemeStep.tsx:85-123`):**
   - **Automatic on onboarding step mount (macOS only):** `ThemeStep.tsx:85-123` calls `previewGhosttyImport()` on mount, reading `~/.config/ghostty/config*` and `~/Library/Application Support/com.mitchellh.ghostty/config*` (`discovery.ts:6-30`), and emits **`onboarding_ghostty_discovered`** (`state: 'found' | 'absent'`, `field_group_count_bucket`) via PostHog if telemetry is enabled (`src/shared/telemetry/schemas/telemetry-onboarding-event-schemas.ts:220-227`). Actual theme values stay local and are applied only on user click.
6. **Hardware/OS machine identifiers (`MachineGuid`, `IOPlatformUUID`, `/etc/machine-id`):**
   - Read by `profileStateAccessMachineIdentity` (`src/main/persistence/profile-state/access/profile-state-access-identity.ts:16-64`) and `readHostIdentity` (`src/main/agent-hooks/installation/locking/managed-hook-owner-identity.ts:108-147`) solely for local SQLite/lockfile ownership and clone detection across shared network/NFS homes (plus SHA-256 `machineTag` scoping of `orcad-secret-store-<hash16>.json` at `src/main/orcad/credentials/orcad-secret-key-record.ts:87-97`). Verified via `rg` not to flow into PostHog or diagnostic uploads.
7. **macOS TCC logs, WSL Registry, GitHub CLI `user/starred` probes:**
   - **Gated telemetry probes:** `daemon_pty_cwd_denial_capture` queries macOS TCC logs, helper code signatures, and Full Disk Access ONLY when `isTelemetryEnabled()` returns `true` at every await (`tcc-denial-capture.ts:80-94,117,139,190,210`); `WslUsageReporter` reads `HKLM\...\Lxss\MSI` and WSL distro registry ONLY when `isTelemetryEnabled()` is `true` (`wsl-usage-reporter.ts:41-54,252-254`).
   - **Non-telemetry GitHub probe:** `StarNagService` automatically invokes `gh api --include user/starred/stablyai/orca` (`src/main/github/client/fetch/orca-star.ts:20-31`) when threshold/value-moment conditions trigger (`src/main/star-nag/service.ts:94,107`) to suppress star prompts if already starred, emitting `star_nag_outcome` / `app_starred_orca` only when telemetry is enabled.

---

## 3. Persistent Device / Install IDs & Hashing

| Identifier / Artifact | Generation & Storage | Transmitted To & Semantics | Hashing / Rotation |
|---|---|---|---|
| **`installId` (`install_id`)** | `randomUUID()` generated on first profile load (`loaded-cohort-migrations.ts:211-214`) and persisted in profile `settings.telemetry.installId`. | Sent as `distinctId` and `commonProps.install_id` on every PostHog event (`client.ts:246-248`, `posthog-transport.ts:23`). **Retained on disk even when opted out** (`consent.ts:91,100`), and emitted once on `telemetry_opted_out` (`opt-out-capture.ts:25`). | Random UUIDv4; NOT derived from hardware IDs. Locally hashed via **unsalted SHA-256** (`sha256("browser-extensions-banner:" + installId)`) solely for deterministic local UI banner bucketing (`src/main/telemetry/browser-extensions-banner.ts:38-43`). |
| **`sessionId` (`session_id`)** | `randomUUID()` per app launch (`src/main/telemetry/client.ts:89`). | Sent on every PostHog event (`posthog-transport.ts:24`) including `telemetry_opted_out`. | Ephemeral per-process UUIDv4. |
| **`bundleSubmissionId` (`bundle_submission_id`)** | Random bytes (`bundle.ts:30,38`) generated per manual diagnostic bundle preview. | Sent only on manual support upload (`diagnostic-bundle-upload.ts:43`). File header comment at `src/main/observability/bundle.ts:1,29` explicitly enforces *"never use `install_id`"*. | Random per-upload ID; cannot join `install_id` history. |
| **`exact_incarnation_correlation`** | Daemon `launchNonce` hashed on emit (`daemon-audit-eligibility-event.ts:116-123`). | Optional field on `daemon_audit_eligibility` PostHog event (`telemetry-daemon-event-schemas.ts:228-242`). | Domain-prefixed **unsalted SHA-256** (`v1:${sha256("orca:daemon-audit-eligibility:incarnation:v1\0" + launchNonce).slice(0,32)}`) of an ephemeral daemon launch nonce. |
| **`orcad` `machineTag` & lock identities** | Local `/etc/machine-id`, Windows `MachineGuid`, macOS `IOPlatformUUID` (`profile-state-access-identity.ts:16-64`). | **Local only** (lockfiles & `orcad-secret-store-<hash16>.json` filenames at `orcad-secret-key-record.ts:87-97`); never transmitted in telemetry. | Unsalted SHA-256 truncated to 16 hex chars (`orcad-secret-key-record.ts:88-91`). |
| **Orca Cloud hostname `label` & Session Identity** | `os.hostname()` + OIDC session tokens (`desktop-client-report.ts:63-72`). | **Only when signed into Orca Cloud** with an org enabling desktop reporting; sent as `x-orca-client` header (`label=<hostname>`) to `login.onorca.dev` (`profile-cloud-client.ts:262-269`). | Cleartext URI-encoded machine hostname + OAuth access/refresh tokens. |

---

## 4. Exact Outbound Telemetry / Analytics / Crash / OTEL / Relay / Update Endpoints

1. **Product Analytics (PostHog US Cloud):**
   - `https://us.i.posthog.com` (`src/main/telemetry/posthog-transport.ts:82`) — `posthog-node` `^5.52.5` (`package.json:245`), initialized only in official CI builds with `ORCA_POSTHOG_WRITE_KEY` (`.github/workflows/release-cut.yml:1231`).
2. **Manual Support & Crash Endpoints (User-Action Triggered Only):**
   - `https://orca.dev/v1/feedback` (`src/main/ipc/feedback/feedback-request.ts:9`; `mobile/src/diagnostics/connection-diagnostics-submission.ts:3`) — receives user-submitted feedback, manual `CrashReportDialog` submissions (with optional user-approved crash logs and optional GitHub login/email if unchecking anonymous), and mobile user-tapped connection diagnostics (`submitAnonymously`/mobile `githubLogin: null, githubEmail: null`).
   - `https://orca.dev/diagnostics/token` & token-returned `upload_url` on `orca.dev` host (`.github/workflows/release-cut.yml:1230`; `src/main/observability/diagnostic-upload-endpoint.ts:1-30`; `src/main/observability/diagnostic-bundle-upload.ts:40-109`) — short-lived upload tokens & NDJSON trace bundle upload/delete (`/diagnostics/delete/<ticketId>`), triggered only after user explicitly creates, previews, and confirms upload in `Settings -> Privacy & Telemetry`. Upload host validation rejects mismatches with `tokenEndpoint.host` (`diagnostic-bundle-upload.ts:107-109`).
   - Note: Crashpad auto-upload is explicitly disabled (`uploadToServer: false`, `crashpad-capture.ts:53`).
3. **Orca Cloud, Mobile Relay & Push Infrastructure (Account / Mobile Pairing Features):**
   - `https://login.onorca.dev` (`src/main/orca-profiles/cloud-auth/profile-cloud-auth-config.ts:25,69-92`) — OIDC auth endpoints (`/v1/desktop/auth/{authorize,session,refresh,capabilities,profile,org,logout,relay-token,connector-token}`). When signed into an org with `desktopReporting` enabled, `capabilitiesEndpoint` receives `x-orca-client`, `x-orca-app-version`, and `x-orca-required-plugin-failures` (`profile-cloud-client.ts:257-270`; `desktop-client-report.ts:49-73`).
   - `https://relay.onorca.dev` (`profile-cloud-auth-config.ts:27,94`) and `https://connector.orca.dev` (`src/main/connector-cloud/connector-origin.ts:5`) — mobile pairing & E2EE relay transport.
   - `https://push.onorca.dev` (`src/main/runtime/push/push-gateway-origin.ts:4`) — host-key authenticated push notification gateway.
   - Cloud worker operational telemetry (Cloudflare Workers Analytics Engine): `cloud/apps/relay-edge/src/metrics.ts:57-78` and `cloud/apps/connector-wake/src/metrics.ts:24-44` record connection/close/wake metrics with edge `<continent>/<colo>` (and verified ingress country in `place`) — no host IDs are written in `connector-wake` (`metrics.ts:2`).
4. **Catalogs, Remote Policies, Desktop & Agent CLI Updaters (Automatic Background HTTPS Fetches):**
   - **Desktop App Updater:** `https://github.com/stablyai/orca/releases.atom` & `https://github.com/stablyai/orca/releases/latest` (`src/main/updater/releases/updater-prerelease-feed.ts:18,82`), `https://api.github.com/repos/stablyai/orca/releases` (`updater-release-builds.ts:32`), `https://github.com/stablyai/orca/releases/latest/download` (`updater-setup.ts:207`), `https://github.com/stablyai/orca/releases/download` (`updater-release-readiness.ts:4`).
   - **Mobile App Updater:** `https://itunes.apple.com/lookup?bundleId=com.stably.orca.mobile` (`mobile/src/app-update/app-store-update-source.ts:5`) and `https://api.github.com/repos/stablyai/orca` (`mobile/src/app-update/github-release-update-source.ts:6`).
   - **Live Remote Policies (4h background refresh in packaged builds):** `https://github.com/stablyai/orca/releases/download/<tag>/agent-state-rules.json` (`src/main/runtime/agent-state-rules/agent-state-rules-live-update.ts:25,41-44`), `https://github.com/stablyai/orca/releases/download/<tag>/wsl-hosts-policy.json` (only on Windows when internal WSL gate is open: `src/main/wsl/host-lifecycle/wsl-hosts-policy-live-update.ts:22-28,70-83`; `src/main/remote-policies/release-asset-policy-live-update.ts:10,39-45`).
   - **Plugins (6h refresh):** `https://orca.dev/plugins/kill-list.json` (`src/main/plugins/catalog/plugin-kill-list-service.ts:11,14`) and `https://orca.dev/plugins/stats.json` (`src/main/plugins/catalog/plugin-install-stats-service.ts:10,14`).
   - **Agent CLI Latest-Version Checks (at most hourly):** `https://cdn.agentclientprotocol.com/registry/v1/latest/registry.json`, `https://registry.npmjs.org/<pkg>/latest`, `https://api.github.com/repos/<repo>/releases/latest` (`src/main/agent-cli-update/agent-cli-latest-release.ts:14,75,80`), and `https://downloads.claude.ai/claude-code-releases` (`src/main/claude/managed-runtime/claude-managed-runtime.ts:13`).

---

## 5. Exact Data & Properties Collected

### Common Properties & Per-Event Experiment Cohort Tags
Every PostHog event attaches `commonProps` (`src/main/telemetry/posthog-transport.ts:12-27`, validated strictly by `commonPropsSchema` at `src/shared/telemetry/telemetry-event-classification.ts:123-149`) plus `$process_person_profile: false` (`src/main/telemetry/client.ts:248`; `src/main/telemetry/opt-out-capture.ts:28`):
- `app_version`, `platform` (`os.platform()`), `arch` (`os.arch()`), `os_release` (`os.release()`), `install_id`, `session_id`, `orca_channel` (`'stable' | 'rc'`), and **5 optional A/B experiment cohort properties** injected at startup via `experimentCohortProps(store)` (`src/main/telemetry/client.ts:93`; `src/main/telemetry/experiment-cohort-props.ts:5-49`): `onboarding_language_v1`, `chat_ui_launch_tip_v1`, `chat_ui_launch_tip_rollout_v1`, `chat_ui_launch_tip_button_emphasis_v1`, and `chat_ui_launch_tip_targeting_v1`.

### Checkpoints Required by Forensic Rubric
- **IP Address & GeoIP (`disableGeoip` / `$geoip_disable` / `$ip`):**
  - `createPostHogClient` sets `disableGeoip: true` in `posthog-node` options and `credentials: 'omit'` on HTTP requests (`src/main/telemetry/posthog-transport.ts:86-91`), with `$process_person_profile: false` per event (`client.ts:248`). No user-level IP address field (`$ip`) is populated, though standard HTTPS TCP/HTTP transport still reaches PostHog US servers (`us.i.posthog.com`) and Orca Cloud/Workers (`relay-edge` and `connector-wake` record Cloudflare `<continent>/<colo>` and ingress country in edge metrics).
- **Custom / Private Model Names:**
  - **Zero model names collected.** Model identifiers do not appear anywhere in `src/shared/telemetry/schemas/` (confirmed via `rg`). Agent identity is closed strictly through `agentKindSchema` (`44` named products + `'other'`, `src/shared/telemetry/schemas/telemetry-property-schemas.ts:11-58`) and `tuiAgentToAgentKind` (`src/shared/agent-status/identity/agent-kind.ts:56-58`), which maps any unknown/custom agent string to `'other'`.
- **Prompts, Code, File Paths & Stack Traces:**
  - **Prompts/Code/Paths:** Blocked by `.strict()` Zod schemas across `src/shared/telemetry/telemetry-event-registry.ts:140-276`. Error classes use fixed enums or errno codes (`errorClassSchema` at `telemetry-property-schemas.ts:62-67`; `ERRNO_CODE_PATTERN` at `telemetry-daemon-event-schemas.ts:301-305`), never raw error messages, file paths, repo names, branch names, URLs, or prompts.
  - **Stack Traces Exception (`main_thread_hang_detected`):** Unlike all other PostHog events, `main_thread_hang_detected` (`src/main/startup/main-process-observers.ts:124-136`; `src/main/hang-watchdog/previous-native-hang-telemetry.ts:15-40`; `src/shared/telemetry/schemas/telemetry-daemon-event-schemas.ts:90-102`) transmits `unresponsive_ms`, `self_recovered`, `macos_build` (`macOS X.Y (Build)`), `menu_bar_icon`, and **`native_top_frames`** (up to 24 native macOS `sample` stack frame strings formatted `${frame.name} [${frame.module}]`). `hangLabel` and `hangModule` (`src/main/hang-watchdog/hang-stacks.ts:47-62`) strip directory paths (basename only) and pass names through `sanitizeCrashReportString(redactString(...))` (`src/shared/crash-reporting/crash-report-redaction.ts:65-81`), preventing local file path or token leakage while still transmitting stack frame symbol/module names.
- **Event Catalog & Behavioral Profiling Fields (`src/shared/telemetry/telemetry-event-registry.ts:140-276`):**
  - **Lifecycle & Consent:** `app_opened` (`nth_repo_added`, `agent_dashboard_enabled`), `telemetry_opted_in`, `telemetry_opted_out`.
  - **Repos & Workspaces:** `repo_added`, `add_repo_*`, `workspace_created` / `workspace_create_failed` (detailed timing/prepared-checkout/bucketed worktree & file count metrics), `setup_script_prompt_*`.
  - **Agents & Usage:** `agent_started`, `agent_prompt_sent` (1% sampled), `agent_error` (enum only), `agent_token_usage_daily` (daily cumulative token counts per agent kind, suppressed across `sharingResumedDay + 1` via `isBeforeSharingResumed` in `consent.ts:142-148`), `agent_hook_*`, `codex_trust_grant`, `first_agent_interaction`, `meaningful_activity_day`.
  - **UI, Dashboard & Native Features:** `agent_dashboard_{active,enabled,opened}`, `settings_changed` (limited to 14 whitelisted boolean keys at `telemetry-property-schemas.ts:199-214`), `worktree_*`, `native_chat_*`, `mobile_session_opened`, `feature_interaction_usage_bucket_reached`, `feature_wall_*`, `contextual_tour_*`, `setup_guide_*`, `chat_ui_feature_tip_*`, `orca_cli_feature_tip_*`, `cmd_j_palette_feature_tip_*`, `app_starred_orca`, `star_nag_outcome`, `browser_extensions_*`, `plugin_installed` (public catalog `publisher.id` & version only; custom/git/local plugins excluded).
  - **Onboarding & Environment Discovery:** `onboarding_*` (step timing, language picker, selected agent kind, `onboarding_ghostty_discovered` + `field_group_count_bucket`, Windows Terminal & task-source snapshots), `activation_checklist_item_completed`.
  - **System, Daemon, SSH & Windows WSL Diagnostics:** `daemon_*` (including `daemon_pty_cwd_denial_capture` reporting macOS TCC auth/reason, Full Disk Access grant state, helper code signature status, iCloud/dataless/volume type, and bucketed time since system wake/unlock/user activity via `tcc-denial-capture.ts:212-235`; `daemon_audit_eligibility` at 0.1% rate with `v1:` launch-nonce hash), `wsl_usage_snapshot` (daily Windows registry MSI WSL version, WSL1/WSL2 distro count buckets, WSL project counts, policy state via `telemetry-wsl-usage-event-schemas.ts:23-57`), `wsl_move*`, `ssh_remote_runtime_resolved`, `ssh_host_server_*`, `profile_state_authority_selected`, `codex_history_move_status`, `codex_legacy_history_import`, `legacy_journal_retirement`.
- **Enterprise Orca Cloud Desktop Reporting (`src/main/orca-profiles/cloud-auth/desktop-client-report.ts:49-73`):** Only when signed into Orca Cloud for an org configured in `getDesktopReportingOrgIds`, adds HTTP headers `x-orca-client` (`v`, `os`, `ch`, `upd`, `err`, `label=<encodeURIComponent(hostname())>`), `x-orca-app-version`, and `x-orca-required-plugin-failures` to `capabilitiesEndpoint` refreshes (`profile-cloud-client.ts:262-269`).

---

## 6. Docs / Privacy Policy vs Code Discrepancies

Comparing `docs/site/content/docs/telemetry.mdx` (and code comments/README) against actual source reveals **material inaccuracies**:

1. **Contradiction — *"No raw error messages or stack frames"* vs. automatic `native_top_frames` upload (`docs/site/content/docs/telemetry.mdx:45`):**
   - `telemetry.mdx:45` states: *"No raw error messages or stack frames. Per-incident error context stays in a local diagnostic trace file on your machine and only reaches us if you explicitly share a diagnostic bundle."*
   - In reality, `main_thread_hang_detected` (`src/main/startup/main-process-observers.ts:124-136`; `src/main/hang-watchdog/previous-native-hang-telemetry.ts:32-38`; `src/shared/telemetry/schemas/telemetry-daemon-event-schemas.ts:90-102`) automatically sends redacted macOS native `sample` stack frames (`native_top_frames`, up to 24 `"name [module]"` strings) and `macos_build` to PostHog on subsequent app launch.
2. **Contradiction — *"Orca has no account system"* vs. Orca Cloud + Enterprise Hostname Reporting (`docs/site/content/docs/telemetry.mdx:46`):**
   - `telemetry.mdx:46` states: *"No user account information (Orca has no account system)."*
   - Orca actually implements an OIDC/OAuth Orca Cloud account system (`https://login.onorca.dev`, scope `'openid profile email offline_access'`, `src/main/orca-profiles/cloud-auth/profile-cloud-auth-config.ts:24-97`), and for signed-in orgs with desktop reporting enabled, transmits `x-orca-client` headers containing machine hostname (`label`), OS/arch, update state/error codes, and required-plugin failures (`desktop-client-report.ts:49-73`).
3. **GeoIP vs. `disableGeoip: true` contradiction (`docs/site/content/docs/telemetry.mdx:47`):**
   - `telemetry.mdx:47` claims *"country is the only geographic signal derived from the request"*, whereas `posthog-transport.ts:86` sets `disableGeoip: true` (*"Disable SDK GeoIP/IP enrichment"*). Conversely, optional mobile Cloud/Relay workers (`cloud/apps/relay-edge/src/metrics.ts:57-75`; `cloud/apps/connector-wake/src/metrics.ts:24-44`) record Cloudflare `<continent>/<colo>` and ingress country in Cloudflare Analytics Engine.
4. **Documented retired events (`docs/site/content/docs/telemetry.mdx:29`):**
   - `telemetry.mdx:29` devotes a full bullet to *"Tab actions — for a consistent 10% sample... Close Tabs To The Left / Close Tabs To The Right"*, yet `src/shared/telemetry/schemas/telemetry-app-event-schemas.ts:32-34` states `"current builds no longer emit them"` and `tab_close_*` events have been removed from `telemetry-event-registry.ts`.
5. **Undocumented shipped telemetry categories (`docs/site/content/docs/telemetry.mdx:16-35`):**
   - `telemetry.mdx` omits multiple active event classes present in `telemetry-event-registry.ts:140-276`: `wsl_usage_snapshot` (Windows WSL/registry census), `main_thread_hang_detected`, `daemon_pty_cwd_denial_capture` (macOS TCC/Full Disk Access/iCloud/idle timing buckets), `daemon_audit_eligibility` (`v1:` hashed launch nonce), `onboarding_ghostty_discovered` (automatic macOS Ghostty config probing), `onboarding_language_*`, `onboarding_task_sources_snapshot`, `onboarding_windows_terminal_snapshot`, `star_nag_outcome` / `app_starred_orca`, `browser_extensions_*`, `wsl_move*`, `ssh_host_server_*`, `codex_history_move_status` / `codex_legacy_history_import`, `legacy_journal_retirement`, and the 5 A/B experiment flags stamped onto every event via `commonPropsSchema` (`telemetry-event-classification.ts:133-147`).
6. **Install-time disclosure promise vs. reality & banner `X` semantics:**
   - Code comments in `TelemetryFirstLaunchSurface.tsx:1` and `FirstLaunchBanner.tsx:1` state *"new users receive install-time disclosure"*, but `OnboardingFlow.tsx` displays none; meanwhile `FirstLaunchBanner.tsx:112-121` silently sets `optedIn: true` and emits `app_opened` when users click the `X` close button.
7. **Settings menu path drift (`docs/site/content/docs/telemetry.mdx:54,58` vs. UI):**
   - Docs instruct users to open **`Settings → Privacy`**, whereas both `FirstLaunchBanner.tsx:85` and `settings-navigation-remote-sections.ts:84` name the section **`Settings -> Privacy & Telemetry`**. Mobile's `Privacy Policy` link points to `https://orca.dev/privacy` (`mobile/src/settings/settings-menu-screen.tsx:58`), while Desktop links to `https://orca.dev/docs/telemetry` (`src/renderer/src/diagnostics/telemetry/telemetry.ts:14`).

---

## 7. Zero-Telemetry / Negative-Check Verification

Read-only `rg -n` checks across `src/`, `mobile/`, `cloud/`, and `docs/site/` (excluding lockfiles, unit tests, fixtures, and i18n locale dictionaries) confirmed:

- **No third-party analytics or crash SDKs in Desktop, Mobile, or Docs:** Searching for `@sentry/`, `@opentelemetry/`, `mixpanel`, `amplitude`, `@segment/`, `analytics.js`, `google-analytics`, `\bgtag\b`, `plausible.io`, `fathom`, `usefathom`, `logrocket`, `bugsnag`, `\brollbar\b`, `node-machine-id`, and `posthog` returned **zero** analytics/crash SDK imports in `mobile/` (`mobile/package.json:25-84`), `docs/site/` (`docs/site/package.json:14-26`), or Electron renderer (`src/renderer/`; only UI brand assets/catalog metadata in `BotAppBrandLogo.tsx:21-22`, `bot-mcp-apps.ts:160`, and webhook adapter name `'sentry'` in `cloud/packages/connector-contract/src/webhook-names.ts:75`). Sole analytics SDK in the repository is `"posthog-node": "^5.52.5"` in root `package.json:245`, confined strictly to `src/main/telemetry/`.
- **Clean custom-model & prompt exclusion:** Searching `src/shared/telemetry/` and `src/main/telemetry/` for `model`, `prompt`, `message_text`, `stdout`, `stdin`, and `cwd` confirmed zero model-name properties, zero prompt/terminal text properties, and strict closed-enum mapping via `tuiAgentToAgentKind` -> `'other'` (`src/shared/agent-status/identity/agent-kind.ts:56-58`).
- **Clean local OS ID containment:** All hits for `MachineGuid`, `IOPlatformUUID`, and `/etc/machine-id` (`src/main/persistence/profile-state/access/profile-state-access-identity.ts:23,25-64`; `src/main/agent-hooks/installation/locking/managed-hook-owner-identity.ts:117-145`; `src/main/orcad/credentials/orcad-secret-key-record.ts:87-97`) are strictly confined to local multi-host locking, clone detection, and per-machine local secret filenames—never referenced by `src/main/telemetry/`, `src/main/observability/`, or `src/main/ipc/feedback/`.

---

```json
{
  "slug": "orca",
  "platform": "Orca",
  "group": "A",
  "repo": "stablyai/orca",
  "sha": "14b04d30f631a5e49f2e649ce2f8ebb6832ddb6b",
  "commit_date": "2026-11-25T04:55:09Z",
  "license": "MIT",
  "status": "Active",
  "default_state": "ON by default on fresh official stable/rc CI builds (optedIn: true with no install-time onboarding notice); pending_banner (telemetry off until resolved) for upgraded pre-telemetry profiles; OFF/fail-closed on contributor/local/PR builds without ORCA_BUILD_IDENTITY & ORCA_POSTHOG_WRITE_KEY; local NDJSON diagnostic tracing ON by default (manual upload only); Crashpad auto-upload OFF; Mobile & Docs zero telemetry",
  "opt_out_controls": [
    "In-app UI toggle: Settings -> Privacy & Telemetry -> Share anonymous usage data (src/renderer/src/components/settings/privacy/PrivacyPane.tsx:106-133)",
    "Pre-telemetry upgrader notice: Opt out button on FirstLaunchBanner (FirstLaunchBanner.tsx:99-107)",
    "Env var DO_NOT_TRACK=1 or true (disables PostHog telemetry AND support diagnostic bundle uploads, retains local trace files)",
    "Env var ORCA_TELEMETRY_DISABLED=1 or true (same semantics as DO_NOT_TRACK)",
    "CI env vars CI, GITHUB_ACTIONS, GITLAB_CI, CIRCLECI, TRAVIS, BUILDKITE, JENKINS_URL, TEAMCITY_VERSION (disables PostHog + all local/upload diagnostics)",
    "Env var ORCA_DIAGNOSTICS_DISABLED=1 or true (disables local diagnostic NDJSON logging + support uploads)"
  ],
  "do_not_track": true,
  "opt_out_bypasses": [
    "Pre-opt-out capture: setOptIn(via, false) calls ensurePostHogClient() and sends telemetry_opted_out (with install_id, session_id, OS props, cohorts) to us.i.posthog.com BEFORE calling client.optOut(), even when a pre-telemetry user clicks 'Opt out' from pending_banner (client.ts:262,278-290; posthog-transport.ts:70-73; opt-out-capture.ts:10-33)",
    "FirstLaunchBanner close 'X' button calls handleAcknowledge() -> persistBannerAcknowledgeWithoutEmitting(), silently setting optedIn: true and firing app_opened (FirstLaunchBanner.tsx:112-121; client.ts:295-309)",
    "Fresh installs receive no install-time disclosure in OnboardingFlow.tsx despite TelemetryFirstLaunchSurface.tsx:1 and FirstLaunchBanner.tsx:1 claiming new users receive install-time disclosure",
    "installId UUIDv4 stays generated/persisted in profile settings even when opted out (loaded-cohort-migrations.ts:211-214), though DNT/ORCA_TELEMETRY_DISABLED/CI prevent all PostHog beacons including telemetry_opted_out"
  ],
  "persistent_ids": [
    "install_id (random UUIDv4 in profile settings.telemetry.installId; retained across opt-out)",
    "session_id (ephemeral per-launch UUIDv4)",
    "bundle_submission_id (random per-upload ID isolated from install_id for manual diagnostic uploads)",
    "exact_incarnation_correlation (v1:<sha256_32> of ephemeral daemon launchNonce in daemon_audit_eligibility)",
    "Orca Cloud enterprise x-orca-client hostname label (only when signed into Orca Cloud org with desktopReporting enabled)"
  ],
  "hashing": "Unsalted domain-prefixed SHA-256 of installId locally for browser-extensions-banner UI bucketing (browser-extensions-banner.ts:38-43) and of ephemeral daemon launchNonce for daemon_audit_eligibility (daemon-audit-eligibility-event.ts:116-123); unsalted SHA-256(machineId) locally for orcad secret filename scoping only",
  "external_auth_reads": "Extensive local non-telemetry reads for account hot-swap, quota meters, and cookie/theme import (~/.claude/.credentials.json & macOS Keychain Claude Code-credentials*, ~/.codex/auth.json, Cursor Keychain/state.vscdb/auth.json, ~/.grok/auth.json, Kimi credentials, ~/.zcode/v2/*, Safari/Chromium/Firefox cookies, Ghostty config) — zero evidence of credential/account exfiltration to PostHog or Orca Cloud. On onboarding ThemeStep mount (macOS), Ghostty config presence/diff size emits onboarding_ghostty_discovered. Gated WSL/TCC probes run only when isTelemetryEnabled() is true.",
  "telemetry_endpoints": [
    "https://us.i.posthog.com (PostHog US Cloud product telemetry, posthog-node)",
    "https://orca.dev/v1/feedback (manual feedback, crash reports, mobile connection diagnostics)",
    "https://orca.dev/diagnostics/token & https://orca.dev/... upload_url (manual preview-gated diagnostic NDJSON bundle uploads)",
    "https://login.onorca.dev, https://relay.onorca.dev, https://connector.orca.dev, https://push.onorca.dev (optional Orca Cloud OAuth, enterprise desktop reporting header, mobile relay & push; Workers Analytics Engine metrics record edge continent/colo & country)",
    "https://orca.dev/plugins/kill-list.json & https://orca.dev/plugins/stats.json (6h background plugin revocation & install-stats fetches)",
    "https://github.com/stablyai/orca/releases* & https://api.github.com/repos/stablyai/orca/releases (desktop/mobile updaters & 4h live agent-state-rules / wsl-hosts-policy JSON fetches)",
    "https://itunes.apple.com/lookup?bundleId=com.stably.orca.mobile (mobile iOS update checks)",
    "https://cdn.agentclientprotocol.com/registry/v1/latest/registry.json, https://registry.npmjs.org/<pkg>/latest, https://api.github.com/repos/<repo>/releases/latest, https://downloads.claude.ai/claude-code-releases (hourly agent CLI version checks)"
  ],
  "ip_geo_handling": "PostHog client explicitly sets disableGeoip: true and credentials: 'omit' with $process_person_profile: false per event (posthog-transport.ts:86-91; client.ts:248), though standard HTTPS requests still reach us.i.posthog.com; cloud relay-edge and connector-wake Workers log Cloudflare continent/colo & country in Analytics Engine",
  "custom_model_handling": "CLEAN — model names are never collected in telemetry schemas; agent_kind uses strict enum allow-list with tuiAgentToAgentKind fallback to 'other' (src/shared/agent-status/identity/agent-kind.ts:56-58; telemetry-property-schemas.ts:11-58)",
  "docs_discrepancies": [
    "docs/site/content/docs/telemetry.mdx:45 claims 'No raw error messages or stack frames' yet main_thread_hang_detected automatically sends native_top_frames (up to 24 redacted macOS native stack frames) and macos_build",
    "telemetry.mdx:46 claims 'Orca has no account system' despite Orca Cloud OAuth (login.onorca.dev) and enterprise x-orca-client hostname reporting (desktop-client-report.ts:49-73)",
    "telemetry.mdx:47 claims country is derived from requests while posthog-transport.ts:86 sets disableGeoip: true",
    "telemetry.mdx:29 documents retired 10%-sampled tab_close_* events no longer emitted (telemetry-app-event-schemas.ts:32-34)",
    "telemetry.mdx omits wsl_usage_snapshot, main_thread_hang_detected, daemon_pty_cwd_denial_capture, daemon_audit_eligibility, per-event A/B cohort tags, onboarding_ghostty_discovered, star_nag_outcome, browser_extensions_*, and migration events",
    "TelemetryFirstLaunchSurface.tsx:1 and FirstLaunchBanner.tsx:1 comments claim new users get install-time disclosure, which OnboardingFlow.tsx lacks",
    "telemetry.mdx:54,58 names Settings -> Privacy instead of Settings -> Privacy & Telemetry"
  ],
  "privacy_rating": "C",
  "one_line_verdict": "Well-intentioned schema hygiene earning C (Zod .strict(), no model names, disableGeoip: true, $process_person_profile: false, strict DNT/CI gating, clean mobile/docs, local-only Crashpad/traces), held below B by default-on tracking without promised onboarding disclosure, pre-opt-out telemetry_opted_out beacons before client.optOut(), FirstLaunchBanner X-dismiss silent opt-in, retained installId, and material docs contradictions on hang stack frames and Orca Cloud accounts."
}
```
