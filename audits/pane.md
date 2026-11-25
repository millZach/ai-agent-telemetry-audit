# Forensic Telemetry & Privacy Audit: Pane (`greenfield-inc/Pane`)

- **Slug:** `pane`
- **Platform:** Pane (`greenfield-inc/Pane`)
- **Group:** `A`
- **Repository:** `https://github.com/greenfield-inc/Pane`
- **Branch:** `main`
- **Audited SHA:** `25299c46356c369ea0ecd99355419be211fac89d` (`2026-11-24T16:08:12-08:00`, `Merge pull request #1651 from greenfield-inc/fix/t45-sidebar-enter-highlighted`)
- **License:** `AGPL-3.0` (`package.json:11`)
- **Status:** Active (`v2.4.254` at `package.json:3`)
- **Privacy Rating:** **`D`** (Aggressive / Leaky — default-on PostHog analytics, 24h heartbeat, and `runpane` CLI telemetry; unconditional `gh api user` network lookup, `git config --global` read, `installId` creation, and `config.json` PII persistence even when opted out or `DO_NOT_TRACK=1`; direct HTTP `$create_alias` + plaintext-PII `analytics_opted_out` requests fired *after* toggling off in Settings; unsalted SHA-256 + plaintext email/username cross-ID stitching; closed-source `runpane.com/api/c` proxy without `$geoip_disable`/`$ip:null`; CLI wrapper fail-open on malformed `config.json`; opt-in leaderboard sending verbatim transcript model strings; and material doc gaps claiming "random"/"anonymous" IDs and zero writes when off)

---

## 1. Default State & Opt-Out Mechanism

### Default State: **ON by default** on Desktop & CLI wrappers; **OFF/Clean** on Opt-In Leaderboard, Headless Daemon, Mobile, Glasses & Remote PWA

- **Desktop Product Analytics (`posthog-js` via `https://runpane.com/api/c`):** Defaults to **ON** (`enabled: true` in `defaultAnalyticsConfig()` at `main/src/services/configManager.ts:56-62`). Undecided installs lacking `analytics_consent_shown=true` and `analytics_default_applied=true` run the `analytics_default_on` experiment in `frontend/src/App.tsx:476-535`, initializing PostHog with `enabled: true`, aliasing `installId` and any web visitor `attribution_ref`, and immediately sending `analytics_default_enabled` and `app_first_opened` via `captureUnconditionally()` (`frontend/src/App.tsx:506-522`, `frontend/src/services/posthog.ts:344-361`). *(Older configs that already saved `analytics.enabled: false` are preserved as off, `frontend/src/App.tsx:497-500`.)*
- **Main-Process Install Heartbeat (`app_heartbeat`), Crash/Exception Tracking (`$exception`, `runpane_local_control_failed`), & Shutdown Ping (`app_closed`):** Controlled by `ConfigManager.isAnalyticsEnabled()` (`main/src/services/configManager.ts:833-836`), which returns `true` by default whenever `analytics.enabled !== false`, environment opt-out is inactive, and `config.json` has been read cleanly (`!unread`). `startHeartbeat()` fires at launch and every 24h on packaged default-directory builds (`main/src/services/analyticsManager.ts:15,118-138`); `app_closed` fires via `net.fetch` during shutdown (`main/src/index.ts:1873-1909`).
- **CLI Wrappers (`runpane` npm & `runpane` PyPI):** ON by default for download lifecycle events (`requested`, `succeeded`, `failed`, `github_fallback_used`) and non-quiet command failures (`runpane_wrapper_command_failed`) to `https://runpane.com/api/runpane/telemetry` (`packages/runpane/src/telemetry.ts:13,267-293`, `packages/runpane-py/src/runpane/telemetry.py:16,195-216`).
- **Opt-In / Telemetry-Free Components:**
  - **Usage Leaderboard (`LeaderboardService`):** **OFF by default** (`config.leaderboard?.optIn` must be explicitly enabled via Settings/Usage UI; `main/src/services/leaderboardService.ts:160-166,200-203,279-280`).
  - **Mobile (`native/`, `mobile/`), Even G2 Glasses (`glasses/`), Remote PWA (`frontend/src/remote/`):** Zero product analytics or crash SDKs (`posthog`, `telemetry`, `analytics`, `capture(` returned 0 matches across `frontend/src/remote/`, `native/src/`, `mobile/`, `glasses/`).
  - **Headless Daemon (`main/src/daemon/startHeadless.ts:19-94`):** Does not instantiate `AnalyticsManager` or run renderer PostHog, though `usageManager.start()` (`startHeadless.ts:73`) starts `OpenRouterPriceProvider`.

### Available Opt-Out Controls
1. **Desktop UI Toggle (`frontend/src/components/settings/categories/PrivacySettings.tsx:81-88`):** `Settings › Privacy › Allow product analytics` calls `persistence.saveConfig('analytics', { analytics: { ...analytics, enabled } })`.
2. **Config File Key (`<PANE_DIR>/config.json`):** `analytics.enabled: false` (`main/src/services/configManager.ts:835`; also checked by npm/PyPI `runpane` wrappers at `packages/runpane/src/telemetry.ts:279-280` and `packages/runpane-py/src/runpane/telemetry.py:201-205`).
3. **Environment Variables:**
   - `DO_NOT_TRACK=1` or non-empty `RUNPANE_TELEMETRY_DISABLED` disables desktop event/error/heartbeat emission (`main/src/services/configManager.ts:52-54,835`, `main/src/ipc/config.ts:27-31`) and `runpane` wrapper telemetry (`packages/runpane/src/telemetry.ts:403-405`, `packages/runpane-py/src/runpane/telemetry.py:326-331`).
   - CI variables (`CI`, `GITHUB_ACTIONS`, `BUILDKITE`, `GITLAB_CI`, `CIRCLECI`, `TF_BUILD`, `JENKINS_URL`, `TEAMCITY_VERSION`, `CODEBUILD_BUILD_ID`) disable `runpane` wrapper telemetry (`telemetry.ts:21,404`, `telemetry.py:24,330`), while desktop continues emitting tagged with `ci: true` (`main/src/utils/analyticsEnvironment.ts:11,79-93`, `docs/ANALYTICS_INVARIANTS.md:135`).
   - `LeaderboardService` checks `DO_NOT_TRACK` in env or via login shell (`execFileSync(shell, ['-l', '-c', 'echo $DO_NOT_TRACK'])`), honoring any non-empty value other than `'0'` or `'false'` (`main/src/services/leaderboardService.ts:59-80,196,279`).
4. **Update Check Control (`main/src/autoUpdater.ts:122`, `main/src/index.ts:1617`):** `autoCheckUpdates: false` suppresses `AutoUpdater` and `VersionChecker` startup/4h checks.

### Opt-Out Bypasses, Races & Leaks When Disabled
1. **Unconditional Identity Lookup (`gh api user`, `git config`), `installId` Creation, & PII Persistence When Opted Out or `DO_NOT_TRACK=1`:**
   - In `frontend/src/App.tsx:545-584`, `initializeAnalytics()` executes **`await resolveAnalyticsIdentity()`** (`App.tsx:561`), **`initPostHog({ enabled: analyticsEnabled, ... })`** (`App.tsx:565-570`), and **`syncDistinctId(distinctId)`** (`App.tsx:581-584`) **unconditionally** even when `analyticsEnabled === false` (pre-configured `analytics.enabled: false`, `DO_NOT_TRACK=1`, or `RUNPANE_TELEMETRY_DISABLED=1`).
   - `ipcMain.handle('analytics:get-identity')` (`main/src/index.ts:1310-1338`) has **zero enabled check** and unconditionally:
     - calls `configManager.getOrCreateAnalyticsInstallId()` (`1312`), generating `install_<uuid>` and writing it to `config.json` (`main/src/services/configManager.ts:842-850`);
     - spawns `gh api user --jq .login`, `gh api user --jq '.email // empty'` (**outbound network API calls to GitHub**), `git config --global user.email`, and `git config --global user.name` (`main/src/services/analyticsIdentity.ts:47-52`);
     - writes plaintext `githubUsername`, `githubEmail`, `gitEmail`, `gitEmailHash`, `gitUserName`, `distinctId`, and `identitySource` into `config.json` via `configManager.setAnalyticsIdentity(identity)` (`main/src/index.ts:1332`, `main/src/services/configManager.ts:856-870`).
   - *(Note: When `config.json` exists on disk but fails to parse as JSON, `ConfigManager.holdUpdate()` at `main/src/services/configManager.ts:629-631` drops `analytics` writes until the file reads cleanly.)*
2. **Direct HTTP `$create_alias` + Plaintext PII Requests Fired *After* Opting Out (`PrivacySettings.tsx:57-71`, `posthog.ts:129-168,271-318`):**
   - Clicking **Allow product analytics** OFF (`saveAnalytics(false)`) saves `{ enabled: false }` (`PrivacySettings.tsx:67`), **then** calls `await resolveIdentity()` (`PrivacySettings.tsx:70` — triggering `analytics:get-identity` / `gh api user` again), and runs `syncAnalytics(false, identity)`, which sends **up to three direct `fetch('${host}/capture/')` HTTP POSTs** bypassing PostHog SDK opt-out:
     - `aliasInstallIdentityDirect(identity)` (`PrivacySettings.tsx:57`, `posthog.ts:285-298`) — `$create_alias` linking `install:<id>` and `email:`/`github:`/`git_name:`;
     - `aliasWebVisitorDirect(identity)` (`PrivacySettings.tsx:59`, `posthog.ts:271-283`) — `$create_alias` linking `attribution_ref` web visitor ID (and unlinks `attribution_ref` on line 60);
     - `captureAndOptOut('analytics_opted_out', undefined, identity)` (`PrivacySettings.tsx:62`, `posthog.ts:307-318`) — sends `analytics_opted_out` with `$set: personProperties(identity)` containing plaintext `github_username`, `github_email`, `git_email`, `git_email_sha256`, `git_user_name`, `install_id`.
   - Opening Privacy Settings while enabled also uses `captureUnconditionally('analytics_settings_disclosure_viewed', ...)` (`PrivacySettings.tsx:25-29`).
3. **`runpane` CLI Wrappers Fail Open on Corrupted `config.json`:**
   - Unlike desktop (`configManager.ts:834-835`), if `<PANE_DIR>/config.json` cannot parse (`status === 'invalid'`, which could hold `analytics.enabled: false`), both `packages/runpane/src/telemetry.ts:279-281,352` and `packages/runpane-py/src/runpane/telemetry.py:201-206,277` skip the opt-out check, create/read `<PANE_DIR>/runpane-wrapper-identity.json`, and **still send telemetry**.
4. **`DO_NOT_TRACK` Parsing Inconsistency:**
   - `configManager.ts:53`, `packages/runpane/src/telemetry.ts:404`, and `packages/runpane-py/src/runpane/telemetry.py:328` check strict `DO_NOT_TRACK === '1'` only, **ignoring** `DO_NOT_TRACK=true` (whereas `leaderboardService.ts:77-79` honors `DO_NOT_TRACK=true`).
5. **Unconditional Third-Party Network Fetches Ignoring Analytics Opt-Out:**
   - `OpenRouterPriceProvider.start()` (`main/src/services/usage/openRouterPriceProvider.ts:106-110,137-143`, invoked unconditionally by `usageManager.start()` at `main/src/services/usage/usageManager.ts:116-117`) fetches `https://openrouter.ai/api/v1/models` on startup and every 6 hours with no opt-out toggle.

---

## 2. External Auth / Local Non-Workspace File Reads

### Telemetry-Linked External Reads & Identity Resolution
1. **GitHub CLI (`gh api user`) & Global Git Config (`analyticsIdentity.ts:47-54`):**
   - Runs unconditionally on every desktop launch (even when analytics or `DO_NOT_TRACK=1` is off) via `analytics:get-identity`:
     - `gh api user --jq .login` and `gh api user --jq '.email // empty'` (`analyticsIdentity.ts:48-49`) — uses user's `gh` authentication to query GitHub's API over HTTPS;
     - `git config --global user.email` and `git config --global user.name` (`analyticsIdentity.ts:50-51`).
   - All four values (`githubUsername`, `githubEmail`, `gitEmail`, `gitUserName`) plus unsalted `sha256(email)` (`gitEmailHash`) are saved to `config.json` (`main/src/services/configManager.ts:856-870`) and transmitted in plaintext to PostHog whenever analytics are enabled or during `PrivacySettings` opt-out (`frontend/src/services/posthog.ts:52-66`; `main/src/index.ts:1895-1900`).
2. **Web Installer Attribution File (`<PANE_DIR>/attribution_ref`):**
   - `readWebAttribution()` (`main/src/services/analyticsIdentity.ts:83-100`) reads base64url `<webDistinctId>|<timestamp>` written outside the app to `<appDir>/attribution_ref` and aliases it to the user's desktop PostHog person profile (`posthog.ts:250-258,271-283`), deleting the file after redemption (`main/src/index.ts:1340-1352`).

### Non-Telemetry Local Non-Workspace Reads (Verified Not Sent to Telemetry/Cloud)
- **Claude & Codex Auth/Account Files (`main/src/services/usage/usageAccounts.ts:7-88`):**
  - Reads `~/.claude.json` (`oauthAccount.accountUuid`, `organizationUuid`, `emailAddress`, `planDisplayName`) and `~/.codex/auth.json` (`tokens.account_id`, JWT payload claims `email`, `chatgpt_account_id`, `chatgpt_user_id`, `chatgpt_plan_type`).
  - **Verification:** `usageAccounts.ts` is called only by `usageManager.getReport()` (`main/src/services/usage/usageManager.ts:175`) for the local Usage & Limits view and synced only between the user's own paired machines (`main/src/services/usage/README.md:48-54`). `LeaderboardService` decodes `usage:get-report` with `usageReportSchema` containing strictly `totals` and `byModel` (`main/src/services/leaderboardService.ts:35-42`), excluding `accounts`, `byProject`, and `byPane`.
- **Agent Transcripts (`main/src/services/usage/usageManager.ts:48-56`):** Scans `~/.claude/projects/**/*.jsonl` and `~/.codex/sessions/**/*.jsonl` locally; only opt-in `LeaderboardService` submits 30-day aggregate token/cost totals and `byModel` summaries (`leaderboardService.ts:82-115`).
- **Codex Live Rate-Limit Probe (`main/src/services/agentUsageService.ts:375-379`):** Spawns local `codex app-server` (`account/read`, `account/rateLimits/read`) on demand via `agent-usage:get` (`main/src/ipc/agentUsage.ts:52-76`).
- **Claude MCP Config (`main/src/services/panels/claude/claudeCodeManager.ts:849-881`) & Repo Finder (`main/src/services/repoFinder.ts:15-75`):** Reads `~/.claude.json` (`mcpServers` and `projects` keys) and Codex `config.toml` (`[projects."..."]`) locally for MCP loading and onboarding repo suggestions (`onboarding` events report counts/statuses only, never repo paths).
- **Third-Party Agent Session Import (`main/src/ipc/sessionImport.ts:36-85`):** Reads `~/.claude`, `~/.codex`, Conductor (`~/Library/Application Support/com.conductor.app/conductor.db`, `sessionImport/sources/conductor.ts:46`), T3 Code (`~/.t3`, `sessionImport/sources/t3code.ts:38-40`), Paradev/Superset, Herdr, Emdash, and Orca stores when importing sessions. `captureSessionImport()` (`frontend/src/components/sessionImport/sessionImportCopy.ts:106-125`) sends counts per source/outcome only, never paths, titles, or IDs.
- **SSH Config (`main/src/services/sshConfigHosts.ts:18-60`):** Reads `~/.ssh/config` (and `Include` files; never key files) for UI host suggestions. `docs/ANALYTICS_INVARIANTS.md:198-200` prohibits logging SSH host aliases and applies `ph-no-capture`.
- **MCP/Skill Auto-Registration (`main/src/services/paneHomeSkill.ts:49-60`, `main/src/services/paneMcpRegistration.ts:357-420`):** Reads/writes Pane MCP and `SKILL.md` registrations under `~/.claude.json`, `~/.claude/skills/`, `~/.agents/skills/`, `~/.codex/config.toml`, `~/.cursor/mcp.json` (on host and WSL distros).
- **FCM Server Push Operator Credentials (`main/src/daemon/mobilePushSender.ts:726-745`):** Reads `PANE_FCM_SERVICE_ACCOUNT_PATH` or `GOOGLE_APPLICATION_CREDENTIALS` / `~/.config/gcloud/application_default_credentials.json` only when a host operator configures FCM push notifications for paired phones.

---

## 3. Persistent Device / Install IDs & Hashing

### Persistent Identifiers
1. **`installId` (`install_<uuidv4>`):** Created in `<PANE_DIR>/config.json` under `analytics.installId` (`main/src/services/configManager.ts:842-850`) **and shared** with both the npm and PyPI `runpane` wrappers (`packages/runpane/src/telemetry.ts:320-353`, `packages/runpane-py/src/runpane/telemetry.py:247-277`). Reused across PostHog `install:<installId>`, `app_heartbeat`, `$exception`, `runpane_local_control_failed`, `app_closed`, `runpane_wrapper_*`, and opt-in `LeaderboardService` submissions (`leaderboardService.ts:88`).
   - **Generated even when opted out / `DO_NOT_TRACK=1`** on desktop launch (`App.tsx:561` → `main/src/index.ts:1312`).
2. **`<PANE_DIR>/runpane-wrapper-identity.json` (`install_<uuidv4>`):** Fallback persistent wrapper ID created when `<PANE_DIR>/config.json` fails JSON parsing (`telemetry.ts:323,352,371-386`, `telemetry.py:250,277,291-305`).
3. **Desktop `distinctId` (`config.json` + PostHog `localStorage`):** `email:<lower>`, `github:<login>`, `git_name:<lower>`, or fallback `install:<installId>` (`main/src/services/analyticsIdentity.ts:56-69`, `frontend/src/services/posthog.ts:204`, `main/src/index.ts:1355-1357`).
4. **Web Visitor `attribution_ref` (`webDistinctId`):** Stitched into desktop PostHog person profile via `$create_alias` (`analyticsIdentity.ts:83-100`, `posthog.ts:250-258,271-283`).
5. **`machineId` (`randomUUID()`):** Stored in `<PANE_DIR>/config.json` (`configManager.ts:424`), used locally/remote-daemon rather than PostHog analytics.
6. **Phone `installationId` (`native/src/features/notifications/registration.ts:41-66`):** Shared only directly with the user's paired Pane computer (`mobile:push-status` / `mobile:push-register`).

### Hashing & Cross-Identity Stitching
- **Unsalted SHA-256 Email Hash (`gitEmailHash` / `git_email_sha256`):** `crypto.createHash('sha256').update(value.trim().toLowerCase()).digest('hex')` (`main/src/services/analyticsIdentity.ts:36-38`). Sent to PostHog alongside **plaintext** `github_email`, `git_email`, `github_username`, and `git_user_name` (`posthog.ts:55-65`), and sent to `LeaderboardService` (`leaderboardService.ts:90`).
- **Unsalted Truncated SHA-256 Session/Pane IDs:** `crypto.createHash('sha256').update(sessionId).digest('hex').substring(0, 16)` (`main/src/services/analyticsManager.ts:216-218`, used for `paneIdHash`/`panelIdHash` local logging in `main/src/ipc/runpane.ts:7295-7296`).
- **Cross-Provider Stitching (`$create_alias`):** `frontend/src/services/posthog.ts:97-105,250-298` merges anonymous `install:<installId>`, web visitor `attribution_ref`, and `email:`/`github:`/`git_name:` identities into one PostHog person profile — including on opt-out (`PrivacySettings.tsx:57-62`).

---

## 4. Exact Outbound Telemetry / Analytics / Crash / OTEL / Relay / Update Endpoints

| Category | Exact Endpoint / Destination | Trigger / Cadence | Opt-Out Honored? | Source Citation |
|---|---|---|---|---|
| **Desktop Product Analytics & Autocapture** | `https://runpane.com/api/c` (`POST /capture/` via `posthog-js`, legacy `https://us.i.posthog.com` migrated) | Launch (`app_first_opened`, `app_opened`, `app_updated`, `app_reactivated_*`), `$pageview`, allowlisted `$autocapture`, feature events, shutdown (`app_closed`) | **Partial** (`enabled: false`, `DO_NOT_TRACK=1`, `RUNPANE_TELEMETRY_DISABLED`; bypassed by `captureUnconditionally`/`captureAndOptOut`/`alias*Direct` and unconditional `analytics:get-identity`) | `frontend/src/services/posthog.ts:5-6,159,177`; `main/src/services/configManager.ts:45-47,398-401`; `main/src/index.ts:1876-1885` |
| **Main Heartbeat & Crash/Error Events** | `https://runpane.com/api/c/capture/` (`$process_person_profile: false`) | `app_heartbeat` at launch & every 24h (packaged default-dir); `$exception` (max 4 concurrent, 20/hr, 5m/sig); `runpane_local_control_failed` | Yes via `isAnalyticsEnabled()` (after unconditional `installId` generation on launch) | `main/src/services/analyticsManager.ts:118-188` |
| **CLI Wrapper Telemetry (`runpane` npm & PyPI)** | `https://runpane.com/api/runpane/telemetry` | `install`/`update` download lifecycle (`requested`, `succeeded`, `failed`, `github_fallback_used`) & non-quiet command failures (1/cause/hr, 10/hr) | **Partial** (`DO_NOT_TRACK=1`, `RUNPANE_TELEMETRY_DISABLED`, CI vars, valid `config.json` `analytics.enabled: false`; **fails open on invalid `config.json`**) | `packages/runpane/src/telemetry.ts:13,267-293,392`; `packages/runpane-py/src/runpane/telemetry.py:16,195-216,310`; `packages/runpane/src/cli.ts:149,581-587,772-789`; `packages/runpane-py/src/runpane/cli.py:445,531-536,1801-1823` |
| **Usage Leaderboard** | `https://runpane.com/api/runpane/leaderboard/submit` (`POST`/`DELETE`) & `GET /api/runpane/leaderboard` | Only when user opts in (`leaderboard.optIn: true`); app open (+8s) or manual Submit/Leave | **Opt-in** (`false` default); respects `DO_NOT_TRACK` (env or shell probe) | `main/src/services/leaderboardService.ts:18-19,176,232,266,278-296` |
| **Identity Enrichment (GitHub CLI)** | GitHub API via `gh api user --jq .login` & `.email // empty` | **Every desktop launch** + on toggling Privacy setting | **NO** — runs unconditionally even when analytics off or `DO_NOT_TRACK=1` | `main/src/services/analyticsIdentity.ts:47-52`; `frontend/src/App.tsx:561`; `PrivacySettings.tsx:70` |
| **OpenRouter Model Pricing Sync** | `https://openrouter.ai/api/v1/models` | Desktop & headless daemon startup (+8s) and every 6 hours (`usageManager.start()`) | **NO** — always fetches public JSON (no user data sent) | `main/src/services/usage/openRouterPriceProvider.ts:7-10,106-143`; `main/src/services/usage/usageManager.ts:116-117` |
| **Desktop Auto-Updates & Version Check** | GitHub Releases (`greenfield-inc/Pane` via `electron-updater`) & `https://api.github.com/repos/greenfield-inc/Pane/releases/latest` | Startup (+1s) and every 4h (`autoUpdater`) / 24h (`VersionChecker`) | Yes via `autoCheckUpdates: false` (`package.json` publish config; `VersionChecker` startup/poll gated at `main/src/index.ts:1617`) | `package.json:259-263`; `main/src/autoUpdater.ts:44,121-128`; `main/src/services/versionChecker.ts:48`; `main/src/index.ts:1615-1620` |
| **CLI Installer Downloads** | `https://runpane.com/api/download?...&source=<npm\|pip>` (fallback `api.github.com` / GitHub releases) | User-invoked `runpane install` / `runpane update` / `doctor` (release lookup) | Explicit user CLI command | `packages/runpane/src/releases.ts:7-8`; `packages/runpane/src/download.ts:28-39`; `packages/runpane-py/src/runpane/download.py:36-40` |
| **Mobile Expo OTA Updates** | `https://pane-native-updates-297294606300.us-central1.run.app/manifest` | `ON_LOAD` in React Native app; Nginx explicitly sets `access_log off` (retains no client IPs) | Built into Expo updates config | `native/app.json:7-10`; `deploy/native-updates/nginx.conf:23-26,39-61` |
| **Monaco Editor CDN** | `https://cdn.jsdelivr.net/npm/monaco-editor@0.52.2/` | Opening code editor panels when `@monaco-editor/loader` loads Monaco assets | Feature use only (allowed in CSP `script-src`/`style-src`/`font-src`) | `frontend/vite.config.ts:9,26-31` |
| **Relay, ACME Certs & Push Delivery** | `us/eu/asia/west.relay.runpane.net`; ACME (`acme-v02.api.letsencrypt.org`, `dv.acme-v02.api.pki.goog`); Google Cloud Logging (`projects/<id>/logs/pane-relay-usage` aggregate counts only); APNs (`api.push.apple.com`), FCM (`fcm.googleapis.com`, `oauth2.googleapis.com`) | Only when Remote Access / Phone Relay pairing & push alerts are enabled by user | Opt-in / Remote feature only; Glasses (`glasses/app.json`) restricted to `*.ts.net:8443` | `main/src/daemon/relay/relayService.ts:25`; `main/src/daemon/relay/computerCertificate.ts:6-11`; `relay/src/usage.rs:29-66`; `main/src/daemon/mobilePushSender.ts:748-808`; `glasses/app.json:12-15` |
| **Voice Transcription & Feedback** | `fal.run/fal-ai/wizper`, `api.deepgram.com`, `openrouter.ai/api/v1/chat/completions` (user-configured BYOK voice); `gh issue create` / `github.com/greenfield-inc/Pane/issues/new` (user-submitted feedback) | Explicit user voice recording or feedback form submit | Explicit user action only | `main/src/services/voiceTranscriptionService.ts:21-25`; `main/src/ipc/feedback.ts:16,126-177` |
| **CI Repo Traffic Snapshot** | `https://us.i.posthog.com/capture/` (`.github/workflows/traffic-snapshot.yml`) | Daily GitHub Actions cron on `greenfield-inc/Pane` | CI server-side only (does not run on user machines) | `.github/workflows/traffic-snapshot.yml:1-43` |

---

## 5. Exact Data & Properties Collected

### A. IP Address & GeoIP Handling
- **No IP/Geo Suppression:** `posthog.init()` (`frontend/src/services/posthog.ts:177-206`), `directCapture()` (`posthog.ts:129-168`), `AnalyticsManager.postCapture()` (`main/src/services/analyticsManager.ts:145-160`), `app_closed` (`main/src/index.ts:1882-1905`), `runpane` wrappers (`telemetry.ts:392`, `telemetry.py:310`), and `LeaderboardService` (`leaderboardService.ts:231-239`) contain **zero** `$geoip_disable: true`, `$ip: null`, or `sendDefaultPii` suppression flags.
- **Closed-Source Ingest Proxy (`https://runpane.com/api/c` & `/api/runpane/*`):** Because the `runpane.com` server source is not in this repository, proxy-side IP stripping cannot be verified statically. However, `docs/ANALYTICS_INVARIANTS.md:119-120` explicitly documents that PostHog *"derives location from the request IP"* even on `app_heartbeat`. *(Positive exception: `deploy/native-updates/nginx.conf:25-26` disables `access_log` on mobile OTA updates specifically to avoid retaining client IPs, and `relay/src/usage.rs:1,45-53` logs only regional relay totals without client IDs or IPs.)*

### B. Custom / Private Model Names
- **Desktop Product Analytics:** Coding-session events (`session_created`, `session_started`) send only coarse `tool_type` (e.g., `'claude'`, `'none'`), **never** custom/local coding LLM model names (`main/src/services/sessionManager.ts:620-635,794-797`). `voice_transcription_used` sends `provider` (`'fal-ai/wizper'` | `'deepgram/nova-3'`), `cleanup_model` (hardcoded `'google/gemini-3.1-flash-lite'`), and `deepgram_model_name` / `deepgram_model_version` (`main/src/services/voiceTranscriptionService.ts:27,436-481`).
- **Opt-In Leaderboard (`LeaderboardService`):** When a user opts in, `buildSubmission()` (`main/src/services/leaderboardService.ts:100-110`) sends up to 50 `byModel[]` entries carrying **verbatim raw `model` strings** parsed from local transcripts (no `custom:<driver>` redaction/masking).

### C. Prompts, Code, File Paths, & Crash Stack Traces
- **Prompts & Code:** **Never collected.** `posthog.init()` sets `mask_all_text: true`, `mask_all_element_attributes: true`, `disable_external_dependency_loading: true`, `advanced_disable_flags: true`, `capture_exceptions: false`, and restricts `autocapture` to interactive selectors (`button`, `a`, `[role="button"]`, `[role="tab"]`, `[role="menuitem"]`, `input[type="checkbox"]`, `input[type="radio"]`, `select`) (`frontend/src/services/posthog.ts:177-206`). Sensitive remote/SSH UI sections use `ph-no-capture` (`docs/ANALYTICS_INVARIANTS.md:196,200`).
- **File Paths:** Excluded from telemetry events (`session_import` sends per-source counts only at `frontend/src/components/sessionImport/sessionImportCopy.ts:106-125`; `setup_imported` sends section IDs/counts only at `frontend/src/components/settings/setupTransfer.ts:138-142`; `remoteAnalytics.ts:169-194` sanitizes to strict enum properties).
- **Stack Traces (`$exception`, `main/src/services/analyticsManager.ts:90-112,190-211`):** Renderer fatal handlers (`frontend/src/desktopBootstrap.tsx:37-89`, `frontend/src/components/ErrorBoundary.tsx:27-40`) pass errors over IPC to `main/src/index.ts:1396-1406`. `AnalyticsManager.captureException()` strictly scrubs raw error messages, causes, usernames, home/repo paths, URLs, and third-party dependency frames via `failureDetails()` (`main/src/utils/failureTelemetry.ts:12-40` — allowlisted built-in JS error types, errno/Pane `ERR_*` codes, coarse category) and `appStackFrames()` (`analyticsManager.ts:190-211` — max 10 frames matching `main/dist/.../*.js` or `frontend/dist/assets/*.js`, with `function: '[app]'` and relative filenames + numeric line/col only).

### D. Behavioral Profile, Identity & Device Context Fields
- **Person Profile Properties (`$identify` & `$set`, `frontend/src/services/posthog.ts:52-66`):** `install_id`, `identity_source` (`'email' | 'github' | 'git_name' | 'posthog' | 'anonymous'`), **`github_username`**, **`github_email`**, **`git_email`**, **`git_email_sha256`**, **`git_user_name`**, `app_version`, `platform`.
- **Event Super-Properties (`contextProperties`, `posthog.ts:68-85`; `analyticsEnvironment.ts:84-93`):** Attaches `install_id`, `identity_source`, `github_username`, `git_email_sha256`, `app_version`, `platform`, `electron_version`, `web_attributed`, `web_attribution_present`, `is_first_launch`, `previous_version`, plus environment flags `ci`, `ci_provider`, `container`, `display` (`native`/`wayland`/`x11`/`virtual`/`none`), `in_wsl`, `packaged`, `custom_pane_dir`, `automated`.
- **Event Inventory:** Lifecycle/retention (`analytics_default_enabled`, `analytics_opted_in`, `analytics_opted_out`, `analytics_settings_disclosure_viewed`, `app_first_opened`, `app_opened`, `app_updated`, `app_reactivated_after_7d`, `app_reactivated_after_30d`, `app_heartbeat`, `app_closed`), onboarding/import/setup (`onboarding_started`, `onboarding_project_supported_after_setup`, `onboarding_project_supported`, `session_import`, `setup_imported`, `live_preview_prompt`, `PaneChatIntroOffer`, `discord_clicked`), core workflow (`project_*`, `folder_*`, `session_*`, `panel_*`, `terminal_panel_created`, `run_script_executed`, `git_worktree_*`, `git_rebase_executed`, `git_squash_executed`, `git_operation_failed`, `git_diff_viewed`, `voice_transcription_used`), remote/pairing (`remote_pane_*`, `pairing_opened`, `pairing_failed`, `pairing_completed`, `certificate_issued`), errors (`$exception`, `runpane_local_control_failed`), CLI wrapper (`runpane_wrapper_download_*`, `runpane_wrapper_github_fallback_used`, `runpane_wrapper_command_failed`), and opt-in leaderboard (`installId`, `githubUsername`, `gitEmailHash`, 30-day token/cost totals, up to 50 `byModel[]` rows, `paneVersion`).

---

## 6. Docs / Privacy Policy vs Code Discrepancies

1. **`claude-plugin/README.md:41` Claims Wrapper Sends "a random install id":**
   - States `runpane` sends *"a random install id"*, and `packages/runpane/README.md:520` calls it an *"anonymous `install_id`"*.
   - **Code Reality:** Both CLI wrappers persist `install_<uuid>` in `<PANE_DIR>/config.json` (`telemetry.ts:320-353`, `telemetry.py:247-277`) — the exact same `installId` desktop aliases via `$create_alias` to plaintext GitHub username/email and git email/name (`posthog.ts:97-105,260-269,285-298`), and that `LeaderboardService` shares with `githubUsername`/`gitEmailHash` (`leaderboardService.ts:88-90`).
2. **`docs/ANALYTICS_INVARIANTS.md:62-73` & `CONTRIBUTING.md:73-76` ("Analytics Off Means Off") vs. Unconditional `gh api user` & `config.json` Writes:**
   - `CONTRIBUTING.md:73-76` tells developers/QA that writing `analytics.enabled: false` or setting `RUNPANE_TELEMETRY_DISABLED=1` stops all analytics, and `docs/ANALYTICS_INVARIANTS.md:62-73` claims analytics off / `DO_NOT_TRACK=1` ensures nothing is sent and `config.json` writes are avoided.
   - **Code Reality:** `frontend/src/App.tsx:561` + `main/src/index.ts:1310-1338` still execute `resolveAnalyticsIdentity()` (`gh api user` network call + `git config --global`), create/save `installId`, and persist plaintext `githubUsername`, `githubEmail`, `gitEmail`, `gitEmailHash`, and `gitUserName` into `config.json` even when `analytics.enabled: false`, `DO_NOT_TRACK=1`, or `RUNPANE_TELEMETRY_DISABLED=1` is set — and `runpane` CLI wrappers fail open onto `runpane-wrapper-identity.json` if `config.json` fails JSON parsing (`telemetry.ts:352`, `telemetry.py:277`).
3. **`docs/ANALYTICS_INVARIANTS.md:104-108` ("Opt-Out Is Still Identified") & `PrivacySettings.tsx:84` Omit Post-Opt-Out `$create_alias` Stitching & Live `gh` Lookup:**
   - `docs/ANALYTICS_INVARIANTS.md:104-108` describes only capturing `analytics_opted_out`, and `PrivacySettings.tsx:84` describes using *"locally available GitHub or Git account details"*. Neither discloses that toggling analytics OFF re-runs `gh api user` (`PrivacySettings.tsx:70`), fires `$create_alias` HTTP calls (`aliasInstallIdentityDirect`, `aliasWebVisitorDirect` at `PrivacySettings.tsx:57-59`), and attaches plaintext GitHub/Git PII to `analytics_opted_out` *after* saving `enabled: false`, nor that `DO_NOT_TRACK=true` is ignored while `DO_NOT_TRACK=1` is required.
4. **No In-Repo Privacy Policy File:**
   - The repository contains no `PRIVACY.md` or website privacy policy source (`claude-plugin/README.md:44` links externally to `https://runpane.com/privacy#claude-plugin`), and `README.md:1-615` mentions telemetry only in the opt-in leaderboard paragraph (`README.md:496-499`).

---

## 7. Zero-Telemetry / Negative-Check Verification

Static read-only `rg` checks across `repos/pane` (excluding lockfiles and `NOTICES`):
- **Third-Party Analytics / Crash / Error SDKs (`\b(sentry|@sentry|opentelemetry|mixpanel|amplitude|bugsnag|datadog|crashlytics|rollbar|honeycomb|newrelic|plausible|fathom|matomo|rudderstack|@segment/analytics|gtag)\b` across `main`, `frontend`, `packages`, `native`, `mobile`, `glasses`, `relay`, `shared`, `claude-plugin`, `.github`):** **Zero SDK usages.** Only non-telemetry matches were test fixture branch name `scrub-sentry` (`frontend/src/utils/paneTitle.test.ts:20,40,48,50`) and prose occurrences of the English word "fathom" inside bundled `SKILL.md` guides.
- **OpenTelemetry / OTLP (`\b(otel|otlp|opentelemetry|@opentelemetry|opentelemetry-api)\b`):** **Zero telemetry/exporter SDKs.** Only matches were the local-only `session-trace` skill docs/script (`main/src/services/paneChatBundle/skills/session-trace/SKILL.md:28`, `scripts/build-trace.mjs:2,289,313`, `references/page-guide.md:32,79`) that writes local `trace.otlp.json` files on user request.
- **Hardware / OS Machine Fingerprinting (`node-machine-id|/etc/machine-id|IOPlatformUUID|MachineGuid`):** **0 matches.**
- **IP / PII Suppression Flags (`\$geoip_disable|geoip_disable|sendDefaultPii|\$ip`):** **0 telemetry matches** (the only `$ip` hits were bash variable `$ip` in `relay/deploy/deploy.sh:90-94`).
- **Mobile, Glasses, & Remote PWA Telemetry (`posthog|analytics|telemetry|capture\(` across `frontend/src/remote/`, `native/src/`, `mobile/`, `glasses/`):** **0 matches** — confirmed clean.

---

```json
{
  "slug": "pane",
  "platform": "Pane",
  "group": "A",
  "repo": "https://github.com/greenfield-inc/Pane",
  "sha": "25299c46356c369ea0ecd99355419be211fac89d",
  "commit_date": "2026-11-24T16:08:12-08:00",
  "license": "AGPL-3.0",
  "status": "Active",
  "default_state": "ON by default for desktop PostHog analytics (via runpane.com/api/c), 24h app_heartbeat, manual JS $exception / runpane_local_control_failed, app_closed, and runpane npm/PyPI CLI wrapper telemetry; OFF by default for Leaderboard (optIn: false), Headless daemon analytics, Mobile (native/mobile), Even G2 glasses, and Remote PWA",
  "opt_out_controls": [
    "Settings -> Privacy -> Allow product analytics toggle (PrivacySettings.tsx:81-88)",
    "Config file <PANE_DIR>/config.json key analytics.enabled: false (checked by desktop and valid-JSON runpane CLI wrappers)",
    "Env vars DO_NOT_TRACK=1 or non-empty RUNPANE_TELEMETRY_DISABLED (plus CI env vars for CLI wrappers only)",
    "autoCheckUpdates: false disables Electron autoUpdater and VersionChecker startup/periodic checks",
    "Leaderboard requires explicit opt-in and honors DO_NOT_TRACK env or login-shell probe"
  ],
  "do_not_track": "Partial (DO_NOT_TRACK=1 stops event/heartbeat emission and CLI wrapper telemetry, while DO_NOT_TRACK=true is ignored; LeaderboardService honors any non-0/false DO_NOT_TRACK; desktop still runs gh api user, git config --global, installId creation, and PII config.json writes when DO_NOT_TRACK=1)",
  "opt_out_bypasses": [
    "frontend/src/App.tsx:561 + main/src/index.ts:1310-1338 unconditionally run analytics:get-identity when analyticsEnabled is false or DO_NOT_TRACK=1, spawning gh api user (outbound GitHub API network call), reading git config --global, creating installId, and writing plaintext PII to config.json and PostHog localStorage",
    "PrivacySettings.tsx:57-71 re-runs resolveIdentity() and sends direct fetch() $create_alias (installId & attribution_ref) plus analytics_opted_out with plaintext PII AFTER saving analytics.enabled: false",
    "packages/runpane/src/telemetry.ts:279-281,352 & telemetry.py:201-206,277 fail open on malformed config.json, falling back to runpane-wrapper-identity.json and emitting telemetry",
    "OpenRouterPriceProvider.start() unconditionally fetches https://openrouter.ai/api/v1/models on startup and every 6h regardless of analytics opt-out"
  ],
  "persistent_ids": [
    "<PANE_DIR>/config.json analytics.installId (install_<uuidv4>, created unconditionally when disabled, shared across desktop, npm/PyPI runpane wrappers, and leaderboard)",
    "<PANE_DIR>/runpane-wrapper-identity.json (fallback install_<uuidv4> if config.json fails JSON parse)",
    "config.json analytics.distinctId & PostHog localStorage (email:<lower>, github:<login>, git_name:<lower>, or install:<id>)",
    "<PANE_DIR>/attribution_ref (webDistinctId stitched into desktop profile via $create_alias)"
  ],
  "hashing": "Unsalted SHA-256 of lowercase trimmed email (gitEmailHash / git_email_sha256) sent alongside plaintext github_email, git_email, github_username, and git_user_name; unsalted 16-char truncated SHA-256 of paneId/panelId for local logs; $create_alias stitches web visitor, installId, and email/GitHub identities",
  "external_auth_reads": "Telemetry runs gh api user (.login, .email) and git config --global (user.email, user.name) unconditionally on every desktop launch (even when opted out). Non-telemetry local/paired features read ~/.claude.json & ~/.codex/auth.json (usageAccounts.ts), ~/.ssh/config (sshConfigHosts.ts), and ~/.claude, ~/.codex, Conductor, T3 Code, Paradev, Herdr, Emdash, Orca stores (sessionImport) — verified not exfiltrated to PostHog or Leaderboard",
  "telemetry_endpoints": [
    "https://runpane.com/api/c/capture/ (PostHog ingest proxy for desktop events, heartbeat, $exception, app_closed; legacy us.i.posthog.com migrated)",
    "https://runpane.com/api/runpane/telemetry (npm & PyPI runpane CLI wrapper download/error telemetry)",
    "https://runpane.com/api/runpane/leaderboard/submit & /api/runpane/leaderboard (opt-in leaderboard)",
    "GitHub CLI gh api user HTTPS requests (unconditional launch identity lookup)",
    "https://openrouter.ai/api/v1/models (unconditional 6h pricing sync)",
    "https://api.github.com/repos/greenfield-inc/Pane/releases/latest & GitHub Releases (desktop auto-updater / VersionChecker)",
    "https://runpane.com/api/download (CLI wrapper downloads, GitHub fallback)",
    "https://pane-native-updates-297294606300.us-central1.run.app/manifest (Expo OTA updates with nginx access_log off)"
  ],
  "ip_geo_handling": "Zero $geoip_disable or $ip:null flags in PostHog SDK, directCapture(), main postCapture(), or CLI wrappers behind closed-source runpane.com/api/c proxy; docs/ANALYTICS_INVARIANTS.md:119-120 confirms PostHog derives location from request IP. Positive exception: mobile OTA nginx disables access_log and relay cloud logging writes aggregate counts only",
  "custom_model_handling": "Session analytics events emit only tool_type (no coding model names); voice_transcription_used emits hardcoded cleanup_model ('google/gemini-3.1-flash-lite') and deepgram_model_name; opt-in LeaderboardService sends up to 50 verbatim raw byModel[].model strings from local transcripts without masking",
  "docs_discrepancies": [
    "claude-plugin/README.md:41 calls wrapper ID a 'random install id' and packages/runpane/README.md:520 calls it 'anonymous install_id', whereas it persists in shared config.json and is $create_alias-linked to plaintext GitHub/Git email and username",
    "docs/ANALYTICS_INVARIANTS.md:62-73 & CONTRIBUTING.md:73-76 claim analytics.enabled:false / DO_NOT_TRACK=1 / RUNPANE_TELEMETRY_DISABLED=1 stop analytics work and config writes, whereas App.tsx:561 + main/src/index.ts:1310-1338 still execute gh api user, git config, installId creation, and PII writes into config.json",
    "docs/ANALYTICS_INVARIANTS.md:104-108 & PrivacySettings.tsx:84 omit that toggling analytics OFF re-runs gh api user and fires direct $create_alias HTTP calls + plaintext PII on analytics_opted_out after saving enabled:false",
    "No in-repo PRIVACY.md or privacy policy source; README.md omits default-on PostHog telemetry disclosure"
  ],
  "privacy_rating": "D",
  "one_line_verdict": "Default-on PostHog and CLI wrapper telemetry with UI/config/DO_NOT_TRACK=1 controls, downgraded to D due to unconditional launch gh api user network calls and config.json PII/installId writes even when opted out, post-opt-out direct $create_alias and plaintext PII transmissions, unsalted email hashing, missing IP suppression behind closed-source runpane.com/api/c, wrapper fail-open on malformed config.json, and 'random/anonymous' ID doc claims."
}
```
