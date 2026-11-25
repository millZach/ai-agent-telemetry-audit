# Forensic Telemetry & Privacy Audit: Vibe Kanban (`BloopAI/vibe-kanban`)

- **Slug:** `vibe-kanban`
- **Platform:** Vibe Kanban (Local CLI / Axum Server / Tauri Desktop / MCP Server / Remote Sync & Relay)
- **Group:** A
- **Repository:** `https://github.com/BloopAI/vibe-kanban`
- **Audited Commit:** `3f095f0186f6e68d67143d38b02da2238f6524d8` (`2026-11-15T17:05:51+01:00` — `Release from a local script to GitHub Releases and npm (#3507)`)
- **License:** Apache-2.0
- **Status:** Active (Sunsetting announcement in `README.md:18-21`, `packages/web-core/src/shared/components/CloudShutdownExportBanner.tsx:34-43`)

---

## Executive Summary

At commit `3f095f0186f6e68d67143d38b02da2238f6524d8` (v0.1.46 sunsetting release), Vibe Kanban has **hard-disabled all first-party PostHog usage analytics and Sentry crash reporting across the local application (`server`, `tauri-app`, `mcp`, `local-web`, and `remote-web`)**, and `scripts/release-local.sh` strips and verifies the absence of analytics keys/DSNs at build time.

However, the repository is not fully zero-telemetry or local-first clean (`A`):
1. `LocalDeployment::new` still unconditionally computes a persistent, hardware-derived, unsalted `machine_id` (`npm_user_<16-hex>`) on every startup (`crates/local-deployment/src/lib.rs:128`, `crates/services/src/services/analytics.rs:111-165`), exposes it via `GET /api/info`, and transmits it to the Relay server if Relay is configured and signed in.
2. The local server and web UI make un-gated outbound HTTP requests on startup and on a 10-minute timer (`cdn.agentclientprotocol.com`, `discord.com`, `api.github.com`, `registry.npmjs.org` in `npx-cli`, plus `cdn.jsdelivr.net` in preview proxy and `fonts.googleapis.com` in OAuth HTML).
3. The repository's remote/cloud sync server crate (`crates/remote`) was **not** stripped of telemetry: it retains active compile-time PostHog tracking (`crates/remote/src/analytics.rs:12-106`, without `$geoip_disable`/`$ip` suppression or user opt-out) and runtime Azure Application Insights OpenTelemetry tracing (`crates/remote/src/lib.rs:40-117`), while `docs/docs.json:158-163` embeds a live PostHog key for the documentation site.
4. Legacy configuration defaults (`Config::default().analytics_enabled = true`), login side effects, uninitialized frontend PostHog/Sentry calls, CI workflows, and stale documentation still reference the removed telemetry controls.

**Overall Privacy Rating: `B` (Opt-In / High-Privacy — local telemetry hard-disabled; remote server telemetry & OTEL operator-configured only)**

---

## 1. Default State & Opt-Out Mechanism

### Local Application (`server`, `tauri-app`, `mcp`, `local-web`, `remote-web`)
- **Runtime Default State: OFF (Hard-Disabled in Code).**
  - **Backend PostHog (`crates/services`):** `AnalyticsConfig::new()` unconditionally returns `None` with the comment `"Usage analytics are turned off: this version of Vibe Kanban sends no PostHog events, whatever keys are set at build or run time."` (`crates/services/src/services/analytics.rs:22-27`). In `LocalDeployment::new`, `let analytics = AnalyticsConfig::new().map(AnalyticsService::new);` (`crates/local-deployment/src/lib.rs:129`) is therefore always `None`, turning every `track_if_analytics_allowed(...)` (`crates/deployment/src/lib.rs:141-147`) and `analytics.track_event(...)` call across `crates/server`, `crates/local-deployment`, and `crates/services` into a runtime no-op.
  - **Rust Sentry (`crates/utils`):** `SentrySource::dsn()` unconditionally returns `None` (`crates/utils/src/sentry.rs:26-30`: `"Crash reporting is turned off: this version of Vibe Kanban never initializes Sentry, whatever DSN is set at build or run time."`). All four binary entrypoints—`crates/server/src/main.rs:39`, `crates/tauri-app/src/main.rs:125`, `crates/mcp/src/bin/vibe_kanban_mcp.rs:141`, and `crates/remote/src/main.rs:13`—call `sentry_utils::init_once(...)`, which exits early at `crates/utils/src/sentry.rs:42-44` without initializing Sentry.
  - **Frontend PostHog & Sentry (`packages/local-web`, `packages/remote-web`):** Neither `posthog.init(...)` nor `Sentry.init(...)` is called anywhere in `packages/`. Explicit comments confirm both are uninitialized:
    - `packages/local-web/src/app/entry/Bootstrap.tsx:19-20`: `"// Usage analytics (PostHog) and crash reports (Sentry) are turned off: this version of Vibe Kanban never initializes either client, so nothing is sent."`
    - `packages/remote-web/src/app/entry/Bootstrap.tsx:23-24`: `"// Usage analytics (PostHog) are turned off: this version of Vibe Kanban never initializes the client, so nothing is sent."`
  - **Local Release Script (`scripts/release-local.sh`):** Clears `ANALYTICS_ENV=(POSTHOG_API_KEY POSTHOG_API_ENDPOINT SENTRY_DSN SENTRY_DSN_REMOTE VITE_POSTHOG_API_KEY VITE_POSTHOG_API_ENDPOINT VITE_SENTRY_DSN SENTRY_AUTH_TOKEN)` before builds (`scripts/release-local.sh:39-40,135-137`) and executes `verify_no_analytics()` (`scripts/release-local.sh:260-267,367`) to fail the build if any `phc_*` key or Sentry DSN is found in `packages/local-web/dist` or compiled binaries.
- **Residual Legacy Config, UI & Onboarding Artifacts:**
  - `Config::default()` still sets `analytics_enabled: true` (`crates/services/src/services/config/versions/v8.rs:47,144`), and migrations from v7 convert `None` to `true` (`crates/services/src/services/config/versions/v8.rs:76`).
  - `finalize_login` in `crates/server/src/routes/oauth.rs:349-379` still flips `config.analytics_enabled = true` on OAuth login if it was `false`, though `deployment.analytics()` is always `None`.
  - In Settings (`packages/web-core/src/shared/dialogs/settings/settings/GeneralSettingsSection.tsx:795-800`), the interactive toggle has been replaced by static text (`settings.general.privacy.noAnalytics`: `"This version of Vibe Kanban doesn't collect usage analytics or crash reports."`, `packages/web-core/src/i18n/locales/en/settings.json:226-228`).
  - There is no onboarding telemetry consent dialog (`LandingPage.tsx:538-558` only states `"By continuing you agree to the terms and conditions and privacy policy."`).
  - Uninitialized wrappers remain in JSX/actions: `<PostHogProvider client={posthog}>` and `<Sentry.ErrorBoundary ... showDialog>` (`packages/local-web/src/app/entry/Bootstrap.tsx:64-76`), `posthog.opt_in_capturing()` / `posthog.identify(machineId)` (`packages/local-web/src/routes/__root.tsx:19-30`), `posthog?.capture(...)` (`LandingPage.tsx:168`, `OnboardingSignInPage.tsx:105`), and `posthog.displaySurvey('019bb6e8-3d36-0000-1806-7330cd3c727e')` (`packages/web-core/src/shared/actions/index.ts:484`).
- **Config Key / CLI Flag / Env Var / `DO_NOT_TRACK`:**
  - Config key `analytics_enabled` (`config.json` via `crates/utils/src/assets.rs:31-33`) exists in schema v8 (`v8.rs:47`), but has no UI toggle and no effect on backend/frontend SDKs.
  - No CLI flag or runtime env var exists to toggle telemetry.
  - `DO_NOT_TRACK` is **not checked** anywhere in the repository (`rg` negative check: 0 matches).

### Remote Sync Server (`crates/remote`) & Other Active Outbound Channels
- **Remote Server PostHog & OTEL (`crates/remote`):** Unlike `crates/services`, `crates/remote/src/analytics.rs:12-33` retains `AnalyticsConfig::from_env()` reading compile-time `option_env!("POSTHOG_API_KEY")` and `option_env!("POSTHOG_API_ENDPOINT")`. When built with those variables (`crates/remote/Dockerfile:39-48`, `crates/remote/docker-compose.yml:80-81`), `Server::run` (`crates/remote/src/app.rs:171-182`) sends PostHog events with **no user-level opt-out check**. Additionally, `crates/remote/src/lib.rs:40-117` initializes OpenTelemetry span export to Azure Application Insights whenever `APPLICATIONINSIGHTS_CONNECTION_STRING` is set at runtime. Both are **OFF by default** unless configured by the server operator.
- **Un-Gated Non-Telemetry Network Channels (No Opt-Out):**
  - **ACP Registry Fetch on Every Startup:** `crates/server/src/startup.rs:183-187` unconditionally spawns `executors::executors::acp::registry::refresh()` (`crates/executors/src/executors/acp/registry.rs:17,203-219`), issuing a `GET` to `cdn.agentclientprotocol.com`.
  - **Frontend Community Counters:** `SharedAppLayout.tsx:71-72` (and `RemoteAppShell.tsx:128-129`) unconditionally mount `useDiscordOnlineCount()` and `useGitHubStars()`, polling Discord and GitHub APIs every 10 minutes (`useDiscordOnlineCount.ts:8,33`, `useGitHubStars.ts:6,31`).
  - **CLI Update Check:** `npx-cli/src/cli.ts:214-230,261` unconditionally checks `registry.npmjs.org` when running a packaged release.
  - **Desktop Auto-Updater (Tauri):** `crates/tauri-app/src/main.rs:257-261,441-511` unconditionally checks for updates on startup and every 60 minutes (`UPDATE_CHECK_INTERVAL`) in non-debug builds, downloading updates silently (`update.download(...)` at `478`) and installing on exit/restart (`267-274,306-310`). (Desktop builds are no longer distributed by `scripts/release-local.sh` per `RELEASING.md:11-12`, and `local-build.sh:107-114` sets the endpoint to `https://localhost/disabled`, while `.github/workflows/pre-release.yml:865-870` previously injected R2 URLs.)

---

## 2. External Auth / Local Non-Workspace File Reads

All external agent directory and credential-path accesses in Vibe Kanban are strictly for **local functional purposes** (auto-recommending an installed coding agent, managing user-initiated MCP configurations, or explicitly attaching Claude Code sessions during `vibe-kanban-review`). None are transmitted to telemetry endpoints in this build (`AnalyticsConfig::new()` is `None`).

However, note the following **unconditional startup read** when telemetry is disabled:

| Target Path / Source | What Is Accessed | Trigger & Condition | Functional vs Telemetry | Source Citation |
| :--- | :--- | :--- | :--- | :--- |
| `/etc/machine-id` (Linux), `IOPlatformUUID` via `ioreg -rd1 -c IOPlatformExpertDevice` (macOS), `HKLM:\SOFTWARE\Microsoft\Cryptography\MachineGuid` via PowerShell (Windows), `USER`/`USERNAME`, `HOME`/`USERPROFILE` | Machine GUID/UUID, OS username, home path (hashed into `npm_user_<hex>`) | **Unconditional on every `LocalDeployment::new` startup**, even with analytics off and before onboarding | Originally analytics `distinct_id`; still populates `UserSystemInfo.machine_id`, Sentry user scope (no-op), and Relay `machine_id` | `crates/local-deployment/src/lib.rs:128`, `crates/services/src/services/analytics.rs:111-165` |
| `$CODEX_HOME/auth.json` or `~/.codex/auth.json`, `config.toml`, `version.json` | `std::fs::metadata(...).modified()` timestamp & `.exists()` (content **not** read by availability check) | Unconditional on pre-onboarding startup (`!onboarding_acknowledged`) via `get_recommended_executor_profile()` and on `GET /api/agents/check-availability`; `config.toml` read/written on explicit `/api/mcp-config` | Functional (agent detection & MCP editing) | `crates/local-deployment/src/lib.rs:101-105`, `crates/executors/src/profile.rs:497-557`, `crates/executors/src/executors/codex.rs:18-25,257-287` |
| `~/.claude.json` | `std::fs::metadata(...).modified()` timestamp (content **not** read by availability check); JSON read/written only via `/api/mcp-config` | Pre-onboarding startup recommendation & `GET /api/agents/check-availability`; `/api/mcp-config` | Functional (agent detection & MCP editing) | `crates/executors/src/executors/claude.rs:603-622`, `crates/server/src/routes/config.rs:291-363` |
| `~/.claude` | Scans custom slash commands & skills (`*.md`) | On executor option discovery (`discover_options`), triggered on startup (`preload_global_executor_options_cache`) | Functional (slash command autocomplete) | `crates/server/src/startup.rs:178-181`, `crates/executors/src/executors/claude/slash_commands.rs:126-128` |
| `~/.claude/projects` | Reads `.jsonl` session transcripts & branch/first-prompt metadata | Only when user invokes `vibe-kanban-review` CLI and interactively selects a Claude session to include in upload | Functional (user-confirmed PR review context upload) | `crates/review/src/main.rs:149-187`, `crates/review/src/claude_session.rs:48-195` |
| `~/.gemini/oauth_creds.json`, `~/.gemini/settings.json`, `~/.gemini/installation_id` | `metadata().modified()` on `oauth_creds.json`; `.exists()` on `installation_id` & `settings.json`; `settings.json` read/written via `/api/mcp-config` | Pre-onboarding startup recommendation & `GET /api/agents/check-availability`; `/api/mcp-config` | Functional (agent detection & MCP editing) | `crates/executors/src/executors/gemini.rs:151-181` |
| `~/.cursor/mcp.json`, `~/.cursor/projects/<slug>/mcp-approvals.json` | `.exists()` check on startup/availability; reads/writes MCP config & auto-approvals when running Cursor or managing MCP | Pre-onboarding startup recommendation; `/api/mcp-config`; Cursor execution | Functional (agent detection, MCP config & approval bootstrap) | `crates/executors/src/executors/cursor.rs:615-635`, `crates/executors/src/executors/cursor/mcp.rs:52-119` |
| `~/.config/opencode` / `$XDG_CONFIG_HOME/opencode`, `$XDG_DATA_HOME`, `$XDG_STATE_HOME`, `~/.opencode`, `~/.qwen/settings.json`, `~/.qwen/installation_id`, `~/.factory/mcp.json`, `~/.factory/installation_id`, `~/.config/amp/settings.json` | `.exists()` on startup/availability; MCP JSON/JSONC read/written via `/api/mcp-config` | Pre-onboarding startup recommendation; `/api/mcp-config` | Functional (agent detection & MCP editing) | `crates/executors/src/executors/opencode.rs:448-526`, `qwen.rs:157-176`, `droid.rs:198-217`, `amp.rs:149-151` |
| `~/.ssh/config`, `~/.ssh/id_rsa`, `~/.vk-ssh/*` | Reads/prepends `Include ~/.vk-ssh/config` in `~/.ssh/config` when opening remote SSH IDE; uses `~/.ssh/id_rsa` as fallback for git clone | User-initiated remote SSH IDE open or git repo clone | Functional | `crates/desktop-bridge/src/ssh_config.rs:89-113`, `crates/git/src/lib.rs:1593-1598` |
| Repo `.git/config` (`user.name`, `user.email`, remotes) | Checks `user.name`/`user.email` (sets fallback `"Vibe Kanban"` / `"noreply@vibekanban.com"` if missing) and reads git remotes | Git workspace/PR operations | Functional | `crates/git/src/lib.rs:208-231,233-245,1408-1425` |
| OS metadata (`os_info::get()`) | `os_type`, `os_version`, `architecture`, `bitness` | `/api/info`, Relay default host nickname (`<os_type> host (<machine_id>)`), Relay pairing `client_os` | Functional UI/Relay metadata (and legacy `AnalyticsService` device payload) | `crates/server/src/routes/config.rs:76-86`, `relay_registration.rs:16-19`, `crates/relay-client/src/lib.rs:212-227` |

**No telemetry reads of external auth files:** Checked `auth.json`, `oauth_creds.json`, `.claude.json` across all crates; none are read for content outside MCP configuration, and none are sent to analytics or crash reporting.

---

## 3. Persistent Device / Install IDs & Hashing

1. **Deterministic Hardware-Fingerprinted Machine ID (`generate_user_id()`):**
   - **Location:** `crates/services/src/services/analytics.rs:111-165`
   - **Generation Timing:** Called **unconditionally on every server/desktop startup** inside `LocalDeployment::new` (`crates/local-deployment/src/lib.rs:128`), before onboarding, without consent, and even though analytics is hard-disabled.
   - **Inputs Hashed:**
     - **macOS:** stdout line containing `IOPlatformUUID` from `ioreg -rd1 -c IOPlatformExpertDevice` (`analytics.rs:114-126`).
     - **Linux:** contents of `/etc/machine-id` (trimmed) (`analytics.rs:128-133`).
     - **Windows:** stdout of `powershell -NoProfile -Command "(Get-ItemProperty -Path 'HKLM:\\SOFTWARE\\Microsoft\\Cryptography').MachineGuid"` (`analytics.rs:135-152`).
     - **All platforms:** `USER` or `USERNAME` env var (`analytics.rs:154-157`) and `HOME` or `USERPROFILE` env var (`analytics.rs:159-162`).
   - **Algorithm & Format:** Fed into Rust's `std::collections::hash_map::DefaultHasher::new()` (SipHash-1-3 with fixed keys `0, 0`, **no per-install random salt**), formatted as `npm_user_{:016x}` (`analytics.rs:112,164`).
   - **Storage & Deletion:** Not persisted to a deletable config file—recomputed dynamically on every process launch, so it cannot be reset or deleted via opt-out.
   - **Propagation:**
     - Exposed to the frontend via `GET /api/info` as `UserSystemInfo.machine_id` (`crates/server/src/routes/config.rs:92,158`).
     - Passed to `sentry_utils::configure_user_scope(user_id, username, email)` (`crates/deployment/src/lib.rs:131-136`) — currently a no-op since Sentry is uninitialized.
     - Passed to uninitialized `posthog.identify(machineId)` in `packages/local-web/src/routes/__root.tsx:24` if `config.analytics_enabled` is true.
     - Sent as `?machine_id=...` when connecting to the Relay WebSocket server (`/v1/relay/connect`) and stored in the remote `hosts` table (`crates/server/src/runtime/relay_registration.rs:75,130`, `crates/relay-tunnel/src/server_bin/routes/connect.rs:24,40-46`, `crates/remote/migrations/20260226100000_relay_hosts_and_sessions.sql:5,15`).
2. **Relay Ed25519 Server Signing Key & Trusted Client IDs (Functional Auth):**
   - `server_ed25519_signing_key` generated/loaded in asset dir (`crates/local-deployment/src/lib.rs:208`, `crates/utils/src/assets.rs:47-49`).
   - Random `Uuid::new_v4()` `client_id` generated during Relay pairing (`crates/relay-client/src/lib.rs:213`, stored in `trusted_ed25519_public_keys.json` / `relay_host_credentials.json`).

---

## 4. Exact Outbound Telemetry / Analytics / Crash / OTEL / Relay / Update Endpoints

| Category | Exact Endpoint / Host | Build/Runtime Key or Config | Status in `3f095f01` | Trigger & Opt-Out | Source Citation |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Local Backend PostHog | `{posthog_api_endpoint}/capture/` | `POSTHOG_API_KEY`, `POSTHOG_API_ENDPOINT` | **Hard-Disabled (Dead)** (`AnalyticsConfig::new()` returns `None`; `release-local.sh` clears & bans keys) | Never fires | `crates/services/src/services/analytics.rs:22-27,47-50`, `scripts/release-local.sh:39-40,260-267` |
| Local Frontend PostHog | `posthog-js` default/configured host | `VITE_POSTHOG_API_KEY`, `VITE_POSTHOG_API_ENDPOINT` (in legacy CI only) | **Hard-Disabled** (`posthog.init()` never called) | Never fires | `packages/local-web/src/app/entry/Bootstrap.tsx:19-20`, `packages/remote-web/src/app/entry/Bootstrap.tsx:23-24`, `.github/workflows/pre-release.yml:190-191` |
| Sentry Crash Reporting | Sentry DSN | `SENTRY_DSN`, `SENTRY_DSN_REMOTE`, `VITE_SENTRY_DSN`, `SENTRY_AUTH_TOKEN` | **Hard-Disabled** (`SentrySource::dsn()` returns `None`; `Sentry.init()` never called) | Never fires | `crates/utils/src/sentry.rs:26-30,41-44`, `packages/local-web/src/app/entry/Bootstrap.tsx:19-20` |
| **Remote Server PostHog** (`crates/remote`) | `{POSTHOG_API_ENDPOINT}/capture/` | Compile-time `option_env!("POSTHOG_API_KEY")`, `option_env!("POSTHOG_API_ENDPOINT")` | **Active if compiled with env vars** (Docker args supported) | Fires on remote auth/CRUD routes; **no user opt-out** | `crates/remote/src/analytics.rs:12-33,51-54`, `crates/remote/src/app.rs:171-182`, `crates/remote/Dockerfile:39-48` |
| **Remote Server OTEL** (`crates/remote`) | Azure Application Insights (parsed from connection string) | Runtime `APPLICATIONINSIGHTS_CONNECTION_STRING` (`OTEL_SERVICE_NAME` defaults to `vibe-kanban-remote`) | **Active if env var set at runtime** | Exports tracing spans on all remote server ops; **no user opt-out** | `crates/remote/src/lib.rs:40-83`, `crates/remote/Cargo.toml:40-44` |
| **Docs Site PostHog** (`docs/`) | `https://eu.i.posthog.com` | Hardcoded key `phc_V5XpxUvgOfEWk38iGHr0Kve2oZTmWFjDe3mIwTCXzx0` | **Active on hosted Mintlify docs** | Visiting Vibe Kanban docs website | `docs/docs.json:158-163` |
| ACP Registry Sync | `https://cdn.agentclientprotocol.com/registry/v1/latest/registry.json` | None | **Active (Unconditional)** | `GET` on **every server startup** (`startup.rs:186`) + on `/api/acp-registry` refresh; **no opt-out** | `crates/server/src/startup.rs:183-187`, `crates/executors/src/executors/acp/registry.rs:17,203-219` |
| UI Community Widgets | `https://discord.com/api/guilds/1423630976524877857/widget.json` & `https://api.github.com/repos/BloopAI/vibe-kanban` | None | **Active (Unconditional)** | Polled every **10 minutes** whenever `SharedAppLayout` or `RemoteAppShell` renders; **no opt-out** | `packages/web-core/src/shared/hooks/useDiscordOnlineCount.ts:3-39`, `useGitHubStars.ts:3-37`, `SharedAppLayout.tsx:71-72` |
| NPX CLI Update Check & Binary Fetch | `https://registry.npmjs.org/vibe-kanban/latest`, `https://github.com/BloopAI/vibe-kanban/releases/download/<tag>/...` | `VIBE_KANBAN_DOWNLOAD_BASE_URL` overrides binary base | **Active (Packaged npm CLI)** | Every packaged `npx vibe-kanban` launch; **no opt-out** for npm version check | `npx-cli/src/cli.ts:214-230,261`, `npx-cli/src/download.ts:16-21,68-70,319-322` |
| Tauri Desktop Auto-Updater | `__TAURI_UPDATE_ENDPOINT__` (CI: `${R2_BINARIES_PUBLIC_URL}/binaries/tauri-update/latest.json`; `local-build.sh`: `https://localhost/disabled`) | `tauri-plugin-updater` pubkey in `tauri.conf.json:42` | **Active in CI-built desktop binaries** (desktop no longer released in `release-local.sh`) | Startup + every 60 min (`UPDATE_CHECK_INTERVAL`); **no opt-out** | `crates/tauri-app/src/main.rs:26,257-261,441-511`, `crates/tauri-app/tauri.conf.json:41-46`, `.github/workflows/pre-release.yml:865-870` |
| GitHub Releases (Changelog) | `https://api.github.com/repos/BloopAI/vibe-kanban/releases` | None | **Active on upgrade** | Only when `show_release_notes` is `true` after version upgrade (`ReleaseNotesDialog`) | `crates/server/src/routes/releases.rs:14`, `packages/local-web/src/routes/_app.tsx:51-58` |
| Cloud API / Relay Tunnel | `VK_SHARED_API_BASE` (`https://api.vibekanban.com` in `release-local.sh` / `local-build.sh`), `VK_SHARED_RELAY_API_BASE` (`wss://<relay>/v1/relay/connect`, unset in `release-local.sh`) | Env / compile-time `option_env!` | **Opt-In (Requires Login)** | Only contacts remote/relay after user signs in (`get_login_status` exits early if no `credentials.json`) | `crates/local-deployment/src/lib.rs:171-204,405-410`, `crates/server/src/runtime/relay_registration.rs:50-78,123-158` |
| Review CLI (`vibe-kanban-review`) | `https://api.vibekanban.com` (`/v1/review/init`, `/v1/review/start`, `/v1/review/{id}/status`) + presigned R2 URL | `--api-url` / `REVIEW_API_URL` | **Explicit User Invocation** | Interactive disclaimer + Enter confirmation + email prompt; uploads PR repo tarball + optional Claude session | `crates/review/src/main.rs:21,50,54-87,194-217`, `crates/review/src/api.rs:85-150` |
| Preview Proxy Eruda CDN & OAuth Google Fonts | `https://cdn.jsdelivr.net/npm/eruda@3.4.3/eruda.js`, `https://fonts.googleapis.com/css2?...` | None | **Active when feature used** | Injected into proxied preview HTML; OAuth callback HTML pages | `crates/preview-proxy/src/lib.rs:608,613`, `crates/server/src/routes/oauth.rs:33` |
| Remote Server Loops Email | `https://app.loops.so/api/v1/transactional`, `https://app.loops.so/api/v1/events/send` | `LOOPS_EMAIL_API_KEY` | **Active on remote server if key set** | Invitations, review notifications, digest emails (`DIGEST_ENABLED`) | `crates/remote/src/mail.rs:187,228,269,340` |

---

## 5. Exact Data & Properties Collected

### A. Active Telemetry in `crates/remote` (When Compiled/Configured with PostHog / OTEL)
- **PostHog Envelope (`crates/remote/src/analytics.rs:50-79`):**
  - `distinct_id`: `user_id` (`Uuid`).
  - Automatic properties on non-`$identify` events: `timestamp` (RFC3339), `version` (`CARGO_PKG_VERSION`), `source: "remote"`.
  - **IP / GeoIP Handling:** No `$geoip_disable` or `$ip` suppression is sent (`analytics.rs:56-79`), so PostHog derives GeoIP from the remote server's source IP (server-side ingestion).
- **PostHog Events in `crates/remote`:**
  - `$identify` with `{"email": user.email}` (`crates/remote/src/auth/local.rs:125-129`, `crates/remote/src/routes/oauth.rs:85-91`).
  - `issue_created` (`issue_id`, `project_id`, `organization_id`, `has_description`, `has_parent`, `priority`) & `subtask_created` (`issue_id`, `parent_issue_id`, `project_id`, `organization_id`) (`crates/remote/src/routes/issues.rs:308-332`).
  - `issue_comment_created` (`comment_id`, `issue_id`, `organization_id`, `is_reply`) (`crates/remote/src/routes/issue_comments.rs:123-134`).
  - `organization_created` (`organization_id`) (`crates/remote/src/routes/organizations.rs:65-73`).
  - `invitation_created` (`invitation_id`, `organization_id`, `role`) & `invitation_accepted` (`organization_id`, `role`) (`crates/remote/src/routes/organization_members.rs:163-173,304-312`).
  - `project_created` (`project_id`, `organization_id`) (`crates/remote/src/routes/projects.rs:118-127`).
  - `workspace_created_from_issue` (`workspace_id`, `project_id`, `issue_id`) (`crates/remote/src/routes/workspaces.rs:100-110`).
- **OpenTelemetry / Azure Application Insights (`crates/remote/src/lib.rs:40-117`, `routes/mod.rs:56-101`):**
  - Traces HTTP requests (`method`, `uri`, `request_id`, `user_id`, `status`, `latency_ms`) and structured `tracing` spans/events (including `audit` logs with `audit_user_id`, `audit_session_id`, `audit_action`, `audit_http_path`).
- **IP Address Storage (`crates/remote/src/routes/review.rs:146-177,214,236`):**
  - Stores raw client IP (`CF-Connecting-IP` → `X-Forwarded-For` → `X-Real-IP`) in PostgreSQL `reviews.ip_address` for `/v1/review/init` rate limiting (2/min, 20/hr).

### B. Dead / Dormant Telemetry Payloads in Local App (Never Sent in `3f095f01`)
Although unreachable at runtime because `AnalyticsConfig::new()` returns `None` (`crates/services/src/services/analytics.rs:25-27`), `SentrySource::dsn()` returns `None` (`crates/utils/src/sentry.rs:28-30`), and frontend `posthog.init()` / `Sentry.init()` are absent, the following dormant calls remain in source:
- **Backend PostHog (`crates/services/src/services/analytics.rs:52-75,167-176`):** Previously sent `distinct_id: npm_user_<hex>`, `device` (`os_type`, `os_version`, `architecture`, `bitness`), `version`, `source: "backend"`, without `$geoip_disable` or `$ip` suppression.
- **Dormant Backend Events:**
  - `session_start` (`startup.rs:175`), `login_status_timeout` (`config.rs:134-140`), `onboarding_disclaimer_accepted`, `onboarding_completed` (with `ExecutorProfileId` and `EditorConfig` including `custom_command`/`remote_ssh_host`/`remote_ssh_user`, `config.rs:212-239`), `analytics_session_start` (`config.rs:229`, `oauth.rs:370-374`).
  - `$identify` with `profile.email` and `$merge_dangerously` aliasing `npm_user_<hex>` to `profile.user_id` (`crates/server/src/routes/oauth.rs:400-415`).
  - `relay_host_paired` (`client_id`, `client_browser`, `client_os`, `client_device`, `relay_pairing/server.rs:65-74`).
  - `task_attempt_finished` (`workspace_id`, `session_id`, `execution_success`, `exit_code`, `container.rs:748-753`), `pr_merged` (`pr_monitor.rs:195-201`), `approval_responded` (`tool_name`, `status`, `approvals.rs:31-38`), `file_uploaded` (`size_bytes`, `mime_type`, `attachments.rs:118-126`), `tag_created` / `tag_updated` (`tag_name`, `tags.rs:44-70`), `repo_editor_opened` (`config`/`repo.rs:177-184`), `workspace_*` / `task_attempt_*` / `dev_server_started` / `setup_script_executed` / `cleanup_script_executed` / `archive_script_executed` / `review_started` / `pr_created` / `gh_cli_setup_executed` / `follow_up_queued` / `follow_up_queue_cancelled` / `organization_created` / `invitation_created`.
- **Dormant Sentry Scope (`crates/deployment/src/lib.rs:131-139`, `crates/remote/src/auth/middleware.rs:160`, `handoff.rs:307,382`):** Populates `sentry::User` with `id`, `username` (GitHub username), and `email` (`github.primary_email` or remote user email). `send_default_pii` is never set (defaults to `false` in Sentry Rust SDK), and `init_once` never initializes Sentry.
- **Dormant Frontend Events (`LandingPage.tsx:168-299`, `OnboardingSignInPage.tsx:47-55,105-367`, `actions/index.ts:484`):** `remote_onboarding_ui_stage_viewed`, `stage_submitted`, `stage_completed`, `stage_failed`, `sign_in_provider_clicked`, `sign_in_provider_result`, `sign_in_more_options_opened`, and `posthog.displaySurvey(...)` — all no-ops because `posthog.init()` is never called.
- **Custom / Private Model Names & Prompts/Code:**
  - **Custom model names:** Not included in any telemetry event payloads (events record `executor` / `variant` only, never `model_id`).
  - **Prompts / source code / file paths / repo names:** Never included in telemetry events. (Note: `vibe-kanban-review` CLI intentionally uploads PR code & optional Claude session logs only when explicitly invoked by the user after accepting the CLI disclaimer; `crates/review/src/main.rs:54-66,182-217`.)

---

## 6. Docs / Privacy Policy vs Code Discrepancies

1. **`README.md` Build-Time PostHog Docs Outdated (`README.md:118-119`):**
   - **Docs say:** `POSTHOG_API_KEY` and `POSTHOG_API_ENDPOINT` are build-time environment variables that enable/disable PostHog analytics.
   - **Actual code:** `crates/services/src/services/analytics.rs:25-27` unconditionally returns `None` regardless of `POSTHOG_API_KEY`/`POSTHOG_API_ENDPOINT`, and `scripts/release-local.sh:39-40,135-137` forcefully unsets them. (Only `crates/remote/src/analytics.rs:13-18` still reads those compile-time variables.)
2. **User Guide Docs Reference Removed Telemetry Toggle & Notice (`docs/settings/general.mdx:173-175`, `docs/configuration-customisation/global-settings.mdx:104-106,126-130`):**
   - **Docs say:** Users can `"Enable or disable telemetry data collection"` under Settings and reset a `"Telemetry Notice"` acknowledgment.
   - **Actual code:** `GeneralSettingsSection.tsx:795-800` has no telemetry toggle or telemetry notice reset; it displays a static message `"This version of Vibe Kanban doesn't collect usage analytics or crash reports."` (`settings.json:228`).
3. **Onboarding Privacy/Terms Link vs Zero-Analytics Claim (`LandingPage.tsx:538-558` vs `settings.json:228`):**
   - `LandingPage.tsx:539-557` requires clicking Continue with `"By continuing you agree to the terms and conditions and privacy policy"` (`https://www.vibekanban.com/privacy`), while local analytics & crash reporting are hard-disabled, yet `generate_user_id()` still computes `machine_id` unconditionally and the app makes un-gated startup/UI requests to ACP registry, Discord, and GitHub.
4. **Legacy CI Workflow vs Local Release Script (`.github/workflows/pre-release.yml` vs `RELEASING.md:13,45-49`, `scripts/release-local.sh`):**
   - `.github/workflows/pre-release.yml:189-205,348-352,393-401,915-919` still wires `POSTHOG_*`, `SENTRY_*`, `getsentry/action-release`, and `sentry-cli debug-files upload`, whereas `RELEASING.md:3-4,13,45-49` explains CI is no longer used and `scripts/release-local.sh` enforces zero analytics/Sentry.
5. **Hosted Docs Mintlify Config (`docs/docs.json:158-163`):**
   - Contains an active PostHog integration (`apiKey: "phc_V5XpxUvgOfEWk38iGHr0Kve2oZTmWFjDe3mIwTCXzx0"`, `apiHost: "https://eu.i.posthog.com"`) for the documentation site, separate from the application binaries.
6. **Stale `analytics_enabled` Default & Login Re-Enable Logic (`v8.rs:144`, `oauth.rs:349-379`):**
   - `Config::default().analytics_enabled` remains `true` and `finalize_login` still sets `analytics_enabled = true` on login, which would toggle `posthog.opt_in_capturing()` in `__root.tsx:22-25` if PostHog were ever re-initialized, even though the UI states no analytics are collected.

---

## 7. Zero-Telemetry / Negative-Check Verification

All negative checks below were directly executed via `rg` across `repos/vibe-kanban` (excluding lockfiles) on commit `3f095f0186f6e68d67143d38b02da2238f6524d8`:

| Check | Command Scope | Result | Verified Status |
| :--- | :--- | :--- | :--- |
| `DO_NOT_TRACK` | Full repo (`!Cargo.lock`, `!pnpm-lock.yaml`, `!package-lock.json`) | **0 matches** (exit 1) | `DO_NOT_TRACK` is not checked anywhere |
| `$geoip_disable` / `geoip_disable` | Full repo (`!Cargo.lock`, `!pnpm-lock.yaml`, `!package-lock.json`) | **0 matches** (exit 1) | PostHog GeoIP suppression is absent (matters for `crates/remote`) |
| `$ip` | Full repo (`!Cargo.lock`, `!pnpm-lock.yaml`, `!package-lock.json`) | **0 matches** (exit 1) | PostHog IP stripping is absent |
| `send_default_pii` / `sendDefaultPii` | Full repo (`!Cargo.lock`, `!pnpm-lock.yaml`, `!package-lock.json`) | **0 matches** (exit 1) | Sentry PII auto-capture flag is absent (and Sentry is disabled) |
| `posthog.init` / `Sentry.init` | `packages/` | **0 matches** (exit 1) | Frontend PostHog & Sentry clients are never initialized |
| Other telemetry SDKs (`mixpanel`, `amplitude`, `segment.io`, `datadog`, `bugsnag`, `rollbar`, `honeycomb`, `aptabase`, `plausible`) | Full repo (`!Cargo.lock`, `!pnpm-lock.yaml`, `!package-lock.json`) | **0 matches** (exit 1) | No hidden third-party analytics SDKs present |
| OS hostname / MAC / `whoami` / `sysinfo` reads | `crates/`, `packages/`, `npx-cli/src/` | **0 OS-identity matches** (only URL `.hostname` parsing & `--hostname 127.0.0.1` flags) | OS hostname, `whoami`, and MAC addresses are never read |

---

## Matrix Summary Block

```json
{
  "slug": "vibe-kanban",
  "platform": "Vibe Kanban",
  "group": "A",
  "repo": "BloopAI/vibe-kanban",
  "sha": "3f095f0186f6e68d67143d38b02da2238f6524d8",
  "commit_date": "2026-11-15T17:05:51+01:00",
  "license": "Apache-2.0",
  "status": "Active",
  "default_state": "OFF (hard-disabled in local server, Tauri, MCP, and web UI; remote server PostHog & Application Insights OTEL OFF unless operator-configured)",
  "opt_out_controls": "Hard-disabled in local code (`AnalyticsConfig::new() -> None`, `SentrySource::dsn() -> None`, no frontend `posthog.init`/`Sentry.init`, `scripts/release-local.sh` strips/bans keys); legacy `analytics_enabled` config key remains `true` by default with UI toggle removed; no opt-out for ACP registry fetch, Discord/GitHub UI polling, npm update check, or remote server telemetry if enabled",
  "do_not_track": "Not checked (0 matches in repo)",
  "opt_out_bypasses": "Local PostHog/Sentry cannot fire; however, un-gated outbound calls run on startup/render (ACP registry `cdn.agentclientprotocol.com`, Discord widget, GitHub stars API, npm latest check, Tauri hourly updater in CI builds, preview-proxy jsDelivr Eruda); `generate_user_id()` runs unconditionally on startup; `crates/remote` ignores `analytics_enabled` when operator enables PostHog/OTEL",
  "persistent_ids": "Unconditional startup `generate_user_id()` (`crates/local-deployment/src/lib.rs:128`) produces `npm_user_<16-hex>` from OS machine ID (`/etc/machine-id` / `IOPlatformUUID` / `MachineGuid`) + username + home dir, exposed at `/api/info` (`machine_id`) and sent to Relay on connect (`crates/server/src/runtime/relay_registration.rs:130`); remote server uses user UUID",
  "hashing": "Unsalted deterministic `std::collections::hash_map::DefaultHasher` (SipHash-1-3 with fixed keys) in `generate_user_id()` (`crates/services/src/services/analytics.rs:112-164`)",
  "external_auth_reads": "Functional-only: pre-onboarding startup and `/api/agents/check-availability` check metadata `mtime`/existence (not file contents) on `~/.codex/auth.json`, `~/.claude.json`, `~/.gemini/oauth_creds.json`, `~/.cursor/mcp.json`, `~/.qwen/`, `~/.factory/`, `~/.opencode/`; MCP editor reads/writes agent configs on request; `vibe-kanban-review` CLI optionally reads `~/.claude/projects` on user confirmation. None sent to telemetry",
  "telemetry_endpoints": "Local app: none active for PostHog/Sentry; active non-telemetry endpoints: `https://cdn.agentclientprotocol.com/registry/v1/latest/registry.json`, `https://discord.com/api/guilds/1423630976524877857/widget.json`, `https://api.github.com/repos/BloopAI/vibe-kanban`, `https://registry.npmjs.org/vibe-kanban/latest`, `https://cdn.jsdelivr.net/npm/eruda@3.4.3/eruda.js`, `https://fonts.googleapis.com/css2`, Tauri `__TAURI_UPDATE_ENDPOINT__`, optional `https://api.vibekanban.com` / relay; Remote server (`crates/remote` if operator-configured): `{POSTHOG_API_ENDPOINT}/capture/` & Azure Application Insights; Docs: `https://eu.i.posthog.com`",
  "ip_geo_handling": "No `$geoip_disable` or `$ip` suppression in `AnalyticsService` (`crates/services/src/services/analytics.rs`, `crates/remote/src/analytics.rs`); `send_default_pii` absent; `crates/remote/src/routes/review.rs:214,236` stores raw client IP (`CF-Connecting-IP`/`X-Forwarded-For`/`X-Real-IP`) in PostgreSQL `reviews.ip_address` for rate limiting",
  "custom_model_handling": "Custom/private model IDs (`model_id`) are not included in any local or remote analytics payloads (dormant events only reference `executor` and `variant`)",
  "docs_discrepancies": "`README.md:118-119` still lists `POSTHOG_API_KEY`/`POSTHOG_API_ENDPOINT` for local builds despite hardcoded disablement; `docs/settings/general.mdx:173-175` & `docs/configuration-customisation/global-settings.mdx:104-106,126-130` still document removed telemetry toggle/notice; `LandingPage.tsx:550` links privacy policy while UI claims no collection; `.github/workflows/pre-release.yml` retains PostHog/Sentry build steps vs `RELEASING.md`; `docs/docs.json:158-163` embeds live PostHog key",
  "privacy_rating": "B",
  "one_line_verdict": "Local PostHog and Sentry telemetry are hard-disabled in this sunsetting release, though unconditional startup hardware-ID derivation, un-gated third-party API/CDN/registry fetches, operator-configurable telemetry in `crates/remote`, and stale config/docs remnants remain."
}
```
