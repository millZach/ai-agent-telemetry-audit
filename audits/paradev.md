# Telemetry & Privacy Forensic Audit — Paradev (formerly Superset)

- **Slug:** `paradev`
- **Platform:** `Paradev (Superset)`
- **Group:** `A`
- **Repository:** `ParadevAI/Paradev` (`https://github.com/ParadevAI/Paradev`, to which `superset-sh/superset` redirects)
- **Branch / Commit:** `main` @ `296f25b450ca126aa2df291dbe905ab48ff15294`
- **Commit Date:** `2026-11-24T16:44:41-06:00`
- **Commit Subject:** `feat(web): add Linux downloads to the web home and README (#10207)`
- **License:** `MIT`
- **Status:** `Active (superset-sh/superset redirects here)`

---

## 1. Default State & Opt-Out Mechanism

Paradev runs **analytics and crash reporting ON by default across every client and server surface**, with **all desktop telemetry opt-out controls removed/stubbed**, **no CLI/mobile/host-service opt-outs**, **only a single environment variable opt-out in the JS SDK (`SUPERSET_TELEMETRY=0`/`false`)**, and **zero `DO_NOT_TRACK` support repository-wide**.

### Desktop App (`apps/desktop`)
- **Default state:** `DEFAULT_TELEMETRY_ENABLED = true` (`apps/desktop/src/shared/constants.ts:61`).
- **Removed database toggle & silent no-op tRPC procedures:** Database migration `packages/local-db/drizzle/0016_add_telemetry_enabled.sql:1` previously added `telemetry_enabled` to `settings`, but `packages/local-db/drizzle/0018_add_delete_local_branch_setting.sql:2` executed `ALTER TABLE \`settings\` DROP COLUMN \`telemetry_enabled\`;` (confirmed absent in `packages/local-db/src/schema/schema.ts:187-277`). The desktop tRPC settings router explicitly leaves stubbed procedures behind (`apps/desktop/src/lib/trpc/routers/settings/index.ts:663-672`):
  - `getTelemetryEnabled` unconditionally returns `true` (`settings/index.ts:664-666`).
  - `setTelemetryEnabled` accepts `{ enabled: z.boolean() }` and silently returns `{ success: true }` without persisting anything (`settings/index.ts:668-672`).
- **No UI control & forced opt-in sync:** `BehaviorSettings.tsx` (`apps/desktop/src/renderer/routes/_authenticated/settings/behavior/components/BehaviorSettings/BehaviorSettings.tsx:28-60`) and `SecuritySettings.tsx` (`apps/desktop/src/renderer/routes/_authenticated/settings/security/components/SecuritySettings/SecuritySettings.tsx:13-35`) expose no telemetry toggle (confirmed by searching `setTelemetryEnabled` across the repo: zero callers exist outside `settings/index.ts`). On mount, `PostHogProvider` (`apps/desktop/src/renderer/providers/PostHogProvider/PostHogProvider.tsx:14-23`) initializes `posthog-js` and sends `desktop_opened` immediately, while `TelemetrySync` (`apps/desktop/src/renderer/components/TelemetrySync/TelemetrySync.tsx:6-21`) queries `getTelemetryEnabled` (always `true`) and calls `posthog.opt_in_capturing()`. Meanwhile `PostHogSurfaceTagger` stamps `surface_source: "opted-in"` onto every PostHog event (`apps/desktop/src/renderer/components/PostHogSurfaceTagger/PostHogSurfaceTagger.tsx:10`). Main-process analytics (`apps/desktop/src/main/lib/analytics/index.ts:24-26, 36-50`) hardcodes `isTelemetryEnabled()` to return `DEFAULT_TELEMETRY_ENABLED` (`true`) and captures events once `userId` is set.
- **Desktop Sentry crash reporting:** Main-process Sentry initializes unconditionally in production whenever `SENTRY_DSN_DESKTOP` is bundled (`apps/desktop/src/main/index.ts:591`, `apps/desktop/src/main/lib/sentry.ts:14-36`), renderer Sentry initializes unconditionally (`apps/desktop/src/renderer/index.tsx:3-5`, `apps/desktop/src/renderer/lib/sentry.ts:11-39`), and tRPC `sentryMiddleware` forwards unhandled `INTERNAL_SERVER_ERROR`s to Sentry (`apps/desktop/src/lib/trpc/index.ts:47-73`). Neither checks any telemetry preference.
- **Desktop auto-update opt-out:** Background/startup update checks run every 4 hours (`UPDATE_CHECK_INTERVAL_MS` in `apps/desktop/src/main/lib/auto-updater.ts:59, 702`) on macOS and Linux (`IS_AUTO_UPDATE_PLATFORM` at `:76`). Opt-out via `DISABLE_UPDATE_CHECK` only works on **macOS** (`PLATFORM.IS_MAC && process.env.DISABLE_UPDATE_CHECK !== undefined` at `:145-147`); Linux builds have no env opt-out or UI toggle to disable update checks.

### Host Service (`packages/host-service` & `apps/desktop/src/main/host-service`)
- **Default state:** Always active whenever signed in (`ctx.api` present) or running with `HOST_SERVICE_SENTRY_DSN` injected in packaged builds (`apps/desktop/src/main/lib/host-service-coordinator.ts:1135-1141`).
- **Opt-out controls:** **None.** `trackWorkspaceEvent` (`packages/host-service/src/workspaces/local-workspace-store.ts:62-90`), `trackAgentLaunched` (`packages/host-service/src/agent-launch-analytics/agent-launch-analytics.ts:174-209`), and `trackConfigResolved` (`packages/host-service/src/runtime/setup/track-config-resolved.ts:10-57`) emit events via `api.analytics.captureEvent` without consulting any preference. Sentry (`packages/host-service/src/sentry.ts:23-93`) enables crash/error reporting unconditionally when `HOST_SERVICE_SENTRY_DSN` is present; sandbox log streaming (`enableLogs`) is gated server-side via `SUPERSET_SANDBOX_BOX_LOGS=1` in cloud sandboxes (`sentry.ts:16-21`).

### CLI (`packages/cli`)
- **Default state:** ON by default. CLI middleware (`packages/cli/src/commands/middleware.ts:25-32`) invokes `trackCommandInvoked()` (`packages/cli/src/lib/analytics.ts:29-49`), which sends `cli_command_invoked` via `api.analytics.captureEvent` once per distinct command per UTC day (deduped in `~/.paradev/reported-commands.json` via `resolveAppHomeDir()` at `:7-27`).
- **Opt-out controls:** **None.** Neither `SUPERSET_TELEMETRY` nor `DO_NOT_TRACK` is checked in `packages/cli`.

### SDK (`packages/sdk`)
- **Default state:** ON by default (`packages/sdk/src/lib/telemetry.ts:1-14, 52-55`).
- **Opt-out control:** Honors `SUPERSET_TELEMETRY=0` or `SUPERSET_TELEMETRY=false` (case-insensitive) via `isTelemetryEnabled()` (`packages/sdk/src/lib/telemetry.ts:14, 52-55`). **Only verified telemetry opt-out env var in the codebase** (though CLI/Desktop/Host-Service ignore it). Does **not** check `DO_NOT_TRACK`.

### Cloud API (`apps/api`, `packages/trpc`, `packages/auth`)
- **Default state:** ON by default. `clientTelemetry` tRPC middleware wraps `publicProcedure` (and derived procedures) and captures `api_procedure_called` at a 1% sample rate (`API_CALL_SAMPLE_RATE = 0.01` at `packages/trpc/src/trpc.ts:152-182`). `analytics.captureEvent` proxies CLI/SDK/host-service events straight into `posthog-node` (`packages/trpc/src/router/analytics/analytics.ts:8-34`). Billing webhooks emit Stripe/subscription PostHog events (`packages/auth/src/lib/billing-analytics.ts:67-118`). Sentry runs in production (`apps/api/sentry.server.config.ts:12-37`, `apps/api/sentry.edge.config.ts:5-11`).
- **Opt-out controls:** None server-side.

### Mobile App (`apps/mobile`)
- **Default state:** ON by default, including **100% session replay** (`enableSessionReplay: true`, `sampleRate: 1`, `screenshotModeBackgroundCapture: true` in `apps/mobile/lib/posthog/client.ts:8-13`), touch autocapture (`captureTouches: true` in `apps/mobile/screens/RootLayout/providers/PostHogProvider/PostHogProvider.tsx:13-22`), Sentry crash + Apple MetricKit diagnostics (`enableMetricKit: true` in `apps/mobile/lib/sentry/sentry.ts:5-23`), and background/foreground Expo OTA updates (`useOtaUpdates` in `apps/mobile/screens/RootLayout/hooks/useOtaUpdates/useOtaUpdates.ts:18-30`).
- **Opt-out controls:** **None.** `SettingsScreen.tsx` (`apps/mobile/screens/(authenticated)/settings/SettingsScreen.tsx:52-468`) contains no privacy/telemetry toggle, and `play-store.config.js` declares PostHog interactions, session replay, Sentry crash logs, and PostHog device IDs as `optional: false` (`apps/mobile/play-store.config.js:70-112`).

### Web, Docs & Marketing Sites (`apps/web`, `apps/docs`, `apps/marketing`)
- **Web (`apps/web`) & Docs (`apps/docs`):** PostHog (`api_host: "/ingest"`, `person_profiles: "always"`, `cross_subdomain_cookie: true`, `persistence_name: "superset"`) and Sentry initialize unconditionally on client load (`apps/web/src/instrumentation-client.ts:13-50`, `apps/docs/src/instrumentation-client.ts:12-55`) with **no cookie consent banner or opt-out UI**.
- **Marketing (`apps/marketing`):** PostHog initializes immediately on load/first interaction (`apps/marketing/src/instrumentation-client.ts:19-55`) and only opts out inside `loaded` if `localStorage["superset-analytics-consent"] === "declined"` (`:36-41`; banner in `apps/marketing/src/components/CookieConsent/CookieConsent.tsx:13-71`). **Opt-out bypasses on Marketing:** Google Ads `gtag` (`AW-18209336001`) and Reddit Pixel (`a2_j3wusi5rxx6o`) are unconditionally injected into `<head>` in `apps/marketing/src/app/[lang]/layout.tsx:150-169` and immediately execute `gtag('config', 'AW-18209336001')` and `rdt('init', ...); rdt('track', 'PageVisit')` **regardless of whether the user declined analytics** (`hasDeclinedAnalytics()` is only checked on subsequent conversion helpers in `apps/marketing/src/lib/analytics/google-ads.ts:18-25` and `apps/marketing/src/lib/analytics/reddit.ts:27-37`). LeadConnector (`x.paradev.com/js/external-tracking.js`) respects `hasDeclinedAnalytics()` via `ExternalTracking` (`apps/marketing/src/components/ExternalTracking/ExternalTracking.tsx:14-54`) and fetch wrapper (`apps/marketing/src/lib/analytics/external-tracking.ts:12-60`).

---

## 2. External Auth / Local Non-Workspace File Reads

Paradev reads a remarkably broad set of **third-party AI CLI credentials, macOS Keychain items, local session databases/transcripts, Chromium browser cookie/history stores, and competitor worktree directories** outside workspace trees:

### A. Third-Party AI Agent Credentials & Keychains (Background 5-Min Polling + On-Demand)
Because `useSettings().showRailUsage` defaults to **`true`** (`apps/desktop/src/renderer/stores/settings.ts:31`), `RailUsage` (`apps/desktop/src/renderer/routes/_authenticated/_dashboard/components/NavigationRail/components/RailUsage/RailUsage.tsx:8-14`) mounts `useHostUsageQuota` (`apps/desktop/src/renderer/hooks/host-service/useHostUsageQuota/useHostUsageQuota.ts:21-36`) and polls `usage.quota` every **5 minutes** whenever connected to a local host, invoking host-service credential discovery (`usage.ts` -> `claude.ts`, `codex.ts`, `opencode-quota.ts`, `agy-quota.ts`, `grok-quota.ts`). **Evidence shows OAuth bearer tokens are used locally to query vendor quota APIs and are NOT uploaded to Paradev PostHog/API** (local-only fingerprint hashes are returned to the renderer UI), **except** where explicitly noted below:

- **Claude Code:**
  - macOS Keychain: executes `security find-generic-password` for `"Claude Code-credentials"` and `"Claude Code-credentials-<sha256(configDir)[0..8]>"` across multiple path spellings (`packages/host-service/src/trpc/router/usage/claude.ts:29, 221`; `packages/host-service/src/trpc/router/usage/profiles.ts:82-95`).
  - Filesystem: `~/.claude/.credentials.json`, `~/.config/claude/credentials.json`, `~/.claude.json`, and scans **every dot-directory under `~` and every subdirectory under `~/.config`** looking for `.claude.json` containing `oauthAccount` (`claude.ts:222-242`; `profiles.ts:108-137`; `read-claude-quota-identity.ts:10-12`).
- **OpenAI Codex CLI:**
  - Scans `$CODEX_HOME` / `~/.codex` and **all `~/.codex*` directories**, reading `auth.json` OAuth tokens (skipping files marked `.superset-api-billing`) (`packages/host-service/src/trpc/router/usage/profiles.ts:264-326`; `packages/host-service/src/trpc/router/usage/usage.ts:197-207`).
- **OpenCode:**
  - Reads `<XDG_DATA_HOME | ~/.local/share>/opencode/auth.json` for Anthropic/OpenAI OAuth tokens (`packages/host-service/src/trpc/router/usage/opencode-quota.ts:1-60`).
- **Cursor CLI:**
  - macOS Keychain: executes `security find-generic-password -s cursor-access-token -a cursor-user -w`; falls back to `~/.cursor/auth.json` and `$XDG_CONFIG_HOME/cursor/auth.json`; also reads `~/.cursor/chats/<md5(cwd)>/<conversationId>/store.db` when computing usage history (`packages/host-service/src/trpc/router/usage/history/cursor.ts:1-14, 58-84, 230-250`).
- **Antigravity (`agy`) CLI:**
  - macOS Keychain: executes `security find-generic-password -s gemini -a antigravity -w` (`packages/host-service/src/trpc/router/usage/agy-quota.ts:40-55`).
  - Transcript logs: reads `~/.gemini/antigravity-cli/brain/<sessionId>/.system_generated/logs/transcript_full.jsonl` (`packages/host-service/src/trpc/router/usage/history/agy.ts:141-157`).
- **Grok Build / CLI:**
  - Reads `~/.grok` / `$GROK_HOME` OAuth auth/session metadata (`packages/host-service/src/trpc/router/usage/grok-quota.ts:1-20`; `packages/host-service/src/trpc/router/usage/history/grok.ts:16-59`).
- **Other Agent Local Session Databases/Logs (for Usage History & Leaderboard):**
  - **GitHub Copilot:** `~/.copilot/session-store.db` SQLite read-only (`packages/host-service/src/trpc/router/usage/history/copilot.ts:16-49`).
  - **Devin CLI:** `<XDG_DATA_HOME | ~/.local/share>/devin/cli/sessions.db` SQLite (`packages/host-service/src/trpc/router/usage/history/devin.ts:15-55`).
  - **fx:** `~/.fx/sessions/<id>/events.jsonl` (`packages/host-service/src/trpc/router/usage/history/fx.ts:16-59`).
  - **Hermes Agent:** `~/.hermes/state.db` (`%LOCALAPPDATA%\hermes\state.db` on Windows), profiles under `profiles/<name>`, and `$HERMES_HOME` (`packages/host-service/src/trpc/router/usage/history/hermes-homes.ts:7-39`, `hermes.ts:22-60`).
  - **Muse Code:** `<XDG_DATA_HOME | ~/.local/share>/muse/sessions/.../session.jsonl` (`packages/host-service/src/trpc/router/usage/history/muse.ts:21-25`).
  - **Pi / Oh My Pi (`omp`):** `~/.pi/agent/sessions/` and `~/.omp/agent/sessions/` (`packages/host-service/src/trpc/router/usage/history/pi.ts:25-27`).
  - **Claude / Codex transcripts:** `~/.claude/`, `~/.config/claude/`, `$CLAUDE_CONFIG_DIR` (`packages/host-service/src/trpc/router/usage/history/entries.ts:45-60`).
- **Git / GitHub Identity:**
  - Reads global Git `user.name` (`git config`) and shells out to `gh api user --jq .login` using the user's ambient `gh` CLI authentication (`packages/host-service/src/runtime/git/identity.ts:19-48`).

### B. Writes/Modifications to External Agent Configs
- At desktop/host startup and setup (`packages/agent-setup/src/agent-setup.ts:1-55`, `packages/agent-setup/src/reconcile-mcp-servers.ts:17-55`, `packages/host-service/src/trpc/router/workspace-creation/shared/seed-agent-trust.ts:1-45`), Paradev installs/updates wrapper scripts, notify hooks, trust entries, and managed MCP server entries into global configs for Claude (`~/.claude/settings.json`, `~/.claude.json`), Codex (`~/.codex/config.toml`), OpenCode (`superset-notify.js`, `tui.js`), Cursor, Copilot, Devin, Gemini, Grok, Muse, Pi, Amp, Droid, Mastra, Kimi, Vibe, etc., unless explicitly disabled per agent (`settings/index.ts:637-661`).

### C. Chromium Browser History & Encrypted Cookie Stores + Keychain Prompts
- Exposes tRPC desktop procedures `browserHistory.getImportSources`, `importFromSource`, and `importCookiesFromSource` (`apps/desktop/src/lib/trpc/routers/browser-history/index.ts:109-227`).
- Reads `Local State`, `History` SQLite (`urls` table up to 10,000 rows), and `Cookies` SQLite across installed Chromium-family browsers (`apps/desktop/src/main/lib/browser/chromium-profiles.ts:32-60`; `chrome-history-import.ts:1-45`; `chrome-cookie-import.ts:262-298`).
- On macOS, `hasSafeStorageKey` probes Keychain metadata (`security find-generic-password -s <service>`) and `readSafeStorageKey` retrieves the decryption password with `-w` (`chrome-cookie-import.ts:77-105`), deriving the AES-128-CBC key via PBKDF2 (`salt: "saltysalt"`, 1003 iterations) to decrypt `v10` cookies into Electron's `persist:browser` partition (filtering out Paradev's own domains, `:309-334`).

### D. Competitor Worktree Scanning (Reported to PostHog During Onboarding)
- `packages/host-service/src/projects/external-worktrees/sources.ts:1-69` scans user home subdirectories for 8 competing tools' worktrees: `~/conductor/workspaces`, `~/.t3/worktrees`, `~/orca/workspaces`, `~/emdash/worktrees`, `~/.claude-squad/worktrees`, `~/.cursor/worktrees`, `~/.herdr/worktrees`, and `~/.paseo/worktrees`.
- **Reported via telemetry:** When onboarding offers import, `useExternalWorktrees` automatically fires PostHog event `external_worktrees_offered` containing `{ sources, repo_count, worktree_count }` (`apps/desktop/src/renderer/routes/_authenticated/onboarding/project/components/ImportFromOtherTools/hooks/useExternalWorktrees/useExternalWorktrees.ts:44-52`).

---

## 3. Persistent Device / Install IDs & Hashing

- **Raw OS Hardware/Machine ID (`getMachineId`):** `packages/shared/src/host-info.ts:14-69` reads:
  - macOS: `/usr/sbin/ioreg -rd1 -c IOPlatformExpertDevice` -> `IOPlatformUUID` (`:19-26`)
  - Linux: `/etc/machine-id`, falling back to `/var/lib/dbus/machine-id` (`:27-32`)
  - Windows: `reg query HKLM\SOFTWARE\Microsoft\Cryptography /v MachineGuid` (`:33-46`)
  - Fallback: `${hostname()}-${homedir()}-superset-fallback` (`:55`)
  - `getMachineId()` stays local and is used as the `scryptSync` password to encrypt/decrypt local `~/.paradev/auth-token.enc` via AES-256-GCM (`packages/shared/src/auth-token-crypto.ts:15-40`; `apps/desktop/src/lib/trpc/routers/auth/utils/auth-functions.ts:12-75`).
- **App-Salted Persistent Device ID (`getHostId`):** `hostIdFromMachineId()` (`packages/shared/src/host-info.ts:74-103`) computes a **fixed-salt HMAC-SHA256**:
  ```ts
  const APP_HOST_SALT = "superset-desktop-device-id-v1";
  createHmac("sha256", APP_HOST_SALT).update(machineId).digest("hex").slice(0, 32)
  ```
  Comment (`:6-7`): *"Salt value preserved verbatim across the rename to keep existing host ids stable for users already registered against the cloud."*
  Transmitted to Paradev cloud in:
  1. `api.host.ensure` (`machineId: getHostId()`, along with `name: getHostName()`, `version`, `platform: ${os.platform()}-${os.arch()}`, `installSource`) on host registration (`packages/host-service/src/tunnel/connect.ts:61-71`).
  2. Host-service PostHog telemetry (`host_id` / `client_machine_id`) on `workspace_created`, `workspace_deleted` (`local-workspace-store.ts:68-84`), and `agent_launched` (`agent-launch-analytics.ts:201`).
  3. Opt-in `leaderboard.publish` (`hostId: machineId`, stored in `leaderboard_daily.hostId`, `publishUsage.ts:91`, `leaderboard.ts:781-814`).
- **Unsalted Hashes Found:**
  - `packages/host-service/src/trpc/router/usage/claude.ts:244`: unsalted SHA-256 of Claude OAuth `accessToken` (local login-change fingerprint returned to desktop UI).
  - `packages/host-service/src/trpc/router/usage/read-claude-quota-identity.ts:14-17`: unsalted SHA-256 of `JSON.stringify([accountUuid, organizationUuid])` from `~/.claude.json` (used as local quota deduplication key).
  - `packages/host-service/src/trpc/router/usage/usage.ts:202-204`: unsalted SHA-256 of raw Codex `auth.json` file contents (local login-change fingerprint).
  - `packages/host-service/src/trpc/router/usage/profiles.ts:75-94`: unsalted SHA-256 truncated to 8 hex chars of literal `CLAUDE_CONFIG_DIR` strings to match Claude Code's own macOS Keychain naming convention.
  - `packages/host-service/src/trpc/router/usage/history/cursor.ts:234`: unsalted MD5 of local workspace `cwd` paths to match Cursor's `~/.cursor/chats/<md5(cwd)>` folder scheme.
  - `packages/auth/src/lib/billing-analytics.ts:38-47`: unsalted SHA-256 of `${event}:${idempotencyKey}` formatted as a UUID for PostHog deduplication.
- **Cross-Surface Identity Stitching:**
  - **Single Shared PostHog Project & Better Auth `user.id`:** CI workflows inject the **same** `secrets.NEXT_PUBLIC_POSTHOG_KEY` across Desktop (`build-desktop.yml:102, 323`), API, Web, Mobile (`EXPO_PUBLIC_POSTHOG_KEY`), Marketing, Docs, and Sandboxes (`deploy-production.yml:155, 370, 493, 542, 942`). Every surface calls `posthog.identify(user.id, { email, name, ... })` or `distinctId: ctx.session.user.id`:
    - Desktop renderer: `apps/desktop/src/renderer/components/PostHogUserIdentifier/PostHogUserIdentifier.tsx:18-32` + `PostHogPlanFlags.tsx:11-12`
    - Desktop main: `apps/desktop/src/main/lib/analytics/index.ts:28-50`
    - Mobile: `apps/mobile/screens/RootLayout/components/PostHogUserIdentifier/PostHogUserIdentifier.tsx:15` (plus local persistent anonymous/device ID in `posthog-react-native` prior to sign-in, `play-store.config.js:110`)
    - Web: `apps/web/src/components/PostHogUserIdentifier/PostHogUserIdentifier.tsx:31-35` (shared cross-subdomain cookie `superset` across web, marketing, and docs via `POSTHOG_COOKIE_NAME = "superset"`, `packages/shared/src/constants.ts:142`)
    - Cloud API / tRPC / Billing / CLI / SDK / Host-Service: `packages/trpc/src/router/analytics/analytics.ts:20-32`, `packages/auth/src/lib/billing-analytics.ts:101-102`

---

## 4. Exact Outbound Telemetry / Analytics / Crash / OTEL / Relay / Update Endpoints

| Channel / Purpose | Default Endpoint / Domain | Source Surface(s) | Citation |
| :--- | :--- | :--- | :--- |
| **PostHog (Direct)** | `https://us.i.posthog.com` (and `https://us.posthog.com`) | Desktop main & renderer, API/tRPC (incl. CLI/SDK/Host-Service relay), Mobile | `apps/desktop/src/main/env.main.ts:26`; `apps/desktop/src/renderer/env.renderer.ts:24`; `packages/trpc/src/env.ts:47, 53`; `apps/mobile/lib/env.ts:12` |
| **PostHog (First-Party Rewrite Proxy)** | `/ingest/*` -> `https://us.i.posthog.com/*` & `https://us-assets.i.posthog.com/static/*` | Web, Marketing, Docs (Next.js rewrites) | `apps/web/next.config.ts:169-180`; `apps/marketing/next.config.ts:143-154`; `apps/docs/next.config.mjs:129-140` |
| **Sentry (Direct & Tunnel)** | `*.ingest.sentry.io` (`SENTRY_DSN_*`); Web/Marketing/Docs browser events tunneled via `/monitoring` | Desktop (`desktop` + `host-service` DSNs), API, Web, Marketing, Docs, Mobile, Relay, Realtime, Usercontent | `apps/desktop/src/main/env.main.ts:27-28`; `apps/desktop/electron.vite.config.ts` (`org: "superset-sh"`); `apps/web/next.config.ts:211`; `apps/marketing/next.config.ts:292`; `apps/docs/next.config.mjs:152`; `apps/mobile/lib/sentry/sentry.ts:10` |
| **Google Ads Conversion & Tag** | `https://www.googletagmanager.com/gtag/js?id=AW-18209336001`, `googleadservices.com`, `doubleclick.net`, `google.com` | Marketing | `apps/marketing/src/app/[lang]/layout.tsx:150-161`; `apps/marketing/src/lib/analytics/google-ads.ts:10-35`; `apps/marketing/next.config.ts:39-44, 57-60` |
| **Reddit Pixel** | `https://www.redditstatic.com/ads/pixel.js?pixel_id=a2_j3wusi5rxx6o`, `https://alb.reddit.com` | Marketing | `apps/marketing/src/lib/constants.ts:4`; `apps/marketing/src/app/[lang]/layout.tsx:163-169`; `apps/marketing/next.config.ts:61, 86` |
| **LeadConnector / GoHighLevel Tracking** | `https://x.paradev.com/js/external-tracking.js` (`tk_7cd75fb5e34042539263f0d7e1a98b57`) -> `https://backend.leadconnectorhq.com/external-tracking/events` | Marketing | `apps/marketing/src/components/ExternalTracking/ExternalTracking.tsx:48-50`; `apps/marketing/src/lib/analytics/external-tracking.ts:23-24` |
| **Cloudflare Web Analytics** | `https://static.cloudflareinsights.com` -> `https://cloudflareinsights.com` (edge-injected CSP allowlist) | Marketing | `apps/marketing/next.config.ts:34, 56, 87` |
| **Desktop Auto-Updater** | `https://github.com/paradevai/paradev/releases/latest/download` (or `/releases/download/desktop-canary`) | Desktop (macOS & Linux every 4h) | `apps/desktop/src/main/lib/auto-updater.ts:59, 82-84, 702`; `apps/desktop/electron-builder.ts:68-71` |
| **CLI Self-Updater** | `https://github.com/paradevai/paradev/releases/download/cli-latest/version.txt` & `.tar.gz` | CLI (`superset update` command) | `packages/cli/src/commands/update/command.ts:26-62` |
| **Mobile OTA Updates** | Expo Updates (`expo-updates` on launch & foreground >=15m) | Mobile | `apps/mobile/package.json:116`; `apps/mobile/screens/RootLayout/hooks/useOtaUpdates/useOtaUpdates.ts:1-30` |
| **Paradev Cloud API / Relay / Realtime / Streams** | `https://api.paradev.com`, `https://app.paradev.com`, `https://relay.superset.sh`, `https://realtime.superset.sh`, `https://streams.superset.sh`, `https://superset-stream.fly.dev` | Desktop, Host-Service, CLI, SDK, Mobile | `apps/desktop/src/main/env.main.ts:21-31`; `packages/cli/src/lib/env.ts:12-13, 36-37`; `apps/mobile/lib/env.ts:9-10` |
| **Third-Party AI Quota Endpoints (Local Token Replay)** | `https://api.anthropic.com/api/oauth/usage`, `https://chatgpt.com/backend-api/wham/usage`, `https://api2.cursor.sh/aiserver.v1.DashboardService/GetFilteredUsageEvents`, `https://daily-cloudcode-pa.googleapis.com` / `cloudcode-pa.googleapis.com`, `https://grok.com/grok_api_v2.GrokBuildBilling/GetGrokCreditsConfig` | Host-Service (polled every 5m when `showRailUsage: true`) | `claude.ts:30`; `codex.ts:17`; `cursor.ts:31, 95`; `agy-quota.ts:8-11`; `grok-quota.ts:7-8` |

---

## 5. Exact Data & Properties Collected

### A. IP Address & Geolocation (`$geoip_disable`, `$ip`, `sendDefaultPii`)
- **Zero `$geoip_disable` or `$ip` suppression anywhere in the repository** (verified via negative `rg` across `apps` and `packages`). Client IP addresses are exposed directly to `us.i.posthog.com` (Desktop, Mobile) and arrive at Vercel/Next.js edge rewrites (`/ingest` on Web, Marketing, Docs) and `api.paradev.com` (for CLI/SDK/Host-Service `captureEvent` calls).
- **Sentry `sendDefaultPii`:**
  - `true` in **4 Next.js apps**: `apps/web/src/instrumentation-client.ts:37`, `apps/api/sentry.server.config.ts:34`, `apps/api/sentry.edge.config.ts:9`, `apps/marketing/src/instrumentation-client.ts:70`, `apps/marketing/sentry.server.config.ts:9`, `apps/marketing/sentry.edge.config.ts:9`, `apps/docs/src/instrumentation-client.ts:44`, `apps/docs/sentry.server.config.ts:9`, `apps/docs/sentry.edge.config.ts:9`.
  - `false` in **Desktop Main, Mobile, Relay, Realtime, Usercontent**: `apps/desktop/src/main/lib/sentry.ts:25`, `apps/mobile/lib/sentry/sentry.ts:14`, `apps/relay/src/index.ts:462`, `apps/realtime/src/index.ts:699`, `apps/usercontent/src/index.ts:597` (and omitted/default in `desktop/renderer` and `host-service`, though `host-service` attaches `organization_id` tags at `packages/host-service/src/sentry.ts:39, 88-90`).

### B. Custom / Private Model Names
- **Mobile (`apps/mobile`):**
  - `new_session_model_selected` captures `{ agent: launchPresetId, model: id }` (`apps/mobile/screens/(authenticated)/components/AgentOptionsScreen/AgentOptionsScreen.tsx:62-65`, `ModelGroupScreen.tsx:50-53`).
  - `chat_message_sent` captures `{ agent, model, effort, message_length, attachment_count, destination }` (`NewChatWidget.tsx:228-237`).
  - `workspace_create_requested` captures `{ workspace_id, project_id, base_branch, agent, model, effort, surface }` (`useCreateTerminalWorkspace.ts:256-266`).
- **Slack Integration (`apps/api`):**
  - `slack_model_changed` captures verbatim `{ model: selectedModel }` (`apps/api/src/app/api/integrations/slack/interactions/route.ts:143-147`).
  - `slack_message_sent` captures verbatim `{ model: slackUserLink.modelPreference, tools_used, actions }` (`process-agent-message.ts:487-496`).
- **Opt-In Leaderboard (`leaderboard.publish`):** Sends verbatim `provider` (max 64 chars) and `model` (max 128 chars) per day to Postgres (`publishDaySchema` in `packages/trpc/src/router/leaderboard/schema.ts:67-80`; `leaderboard-days.ts:56-63`; `leaderboard.ts:776-781`).
- **Desktop/Host-Service (`agent_launched`):** Does **not** send model names; maps built-in presets to catalog ID or `"custom"` + `commandBaseName(launch.command)` (`agent-launch-analytics.ts:149-158`).

### C. Prompts, Code, Branches, Paths & Crash Context
- **Session Replay & Touch Autocapture (Mobile):** `sampleRate: 1`, `screenshotModeBackgroundCapture: true` (`apps/mobile/lib/posthog/client.ts:8-13`), plus `captureTouches: true` with `propsToCapture` constrained away from raw text `"children"` to avoid serializing workspace names/branches (`PostHogProvider.tsx:13-22`). Still captures 100% of screen visuals via PostHog Session Replay unless explicitly masked at view level.
- **Prompts / Transcripts (`chatSync`):** When enabled for a user's org by the server-side PostHog feature flag `workspace-moves` (`assertMovesEnabled.ts:6-19`, polled hourly by `chatHistory.syncEnabled` at `packages/trpc/src/router/chat-history/chat-history.ts:116-119`, `packages/host-service/src/chat-sync/syncWhenEnabled/syncWhenEnabled.ts:34-38`), cloud-box chat sessions push chat turns/items and transcripts to `api.paradev.com` / R2. Local non-cloud desktop workspaces do **not** run `chatSync` unless opted into cloud sync/workspace moves.
- **Workspace Metadata & Commands in Telemetry:**
  - `workspace_created` / `workspace_deleted` (`local-workspace-store.ts:70-85`) sends `workspace_id`, `project_id`, `organization_id`, `host_id`, **`branch` (git branch name)**, `type`, `host_kind`, `client_machine_id`, `host_service_version`.
  - `workspace_config_resolved` (`track-config-resolved.ts:40-52`) sends `format`, `legacy_sources`, `phase`, `workspace_id`, `project_id`, `organization_id` (explicitly omits commands/contents).
  - `agent_launched` (`agent-launch-analytics.ts:149-204`) sends `agent_id` or `"custom"` + **`command_name` (basename of custom command)**, `surface`, `session_kind`, `host_kind`, `command_found`, `resumed`, `workspace_id`, `host_id`.
  - `cli_command_invoked` (`packages/cli/src/lib/analytics.ts:36-45`) sends `command` path, `flags` (flag names), `cli_version`.
  - `sdk_method_called` (`packages/sdk/src/lib/telemetry.ts:38-50`) sends `method`, `sdk_version`, `sdk_lang`, `runtime`, `target`, `success`, `duration_ms` (explicitly no request/response payloads).
  - `external_worktrees_offered` (`useExternalWorktrees.ts:47-51`) sends competitor tool `sources` (`conductor`, `t3code`, `orca`, `emdash`, `claude-squad`, `cursor`, `herdr`, `paseo`), `repo_count`, `worktree_count`.
  - `command_run` (`apps/desktop/src/renderer/commandPalette/core/execute.ts:12`) sends `commandId` and `section`.
  - `api_procedure_called` (`packages/trpc/src/trpc.ts:161-170`) samples 1% of ALL tRPC calls (`procedure`, `client_product`, `client_version`).
- **Crash Stack Traces & Logs:**
  - Host-service crash reporting in Electron (`host-service-coordinator.ts:1230-1247`) sends **`outputTail`** (process stdout/stderr tail) to Sentry, scrubbed by `redactCrashTail(current.outputTail, { secrets: current.redactions, homeDir: os.homedir() })`.
  - Auto-updater error reporting (`auto-updater.ts:620`) scrubs home account names and `update.XXXX` dirs via `redactUpdateError()` (`update-error-redaction.ts:43-70`) before sending to Sentry.
  - General Electron/Host/Mobile errors send full JS/native stack traces to Sentry (note: OS paths in arbitrary main/renderer exceptions outside `auto-updater.ts` are not uniformly passed through `redactUpdateError`), and PostHog `capture_exceptions: true` is enabled on Desktop renderer, Web, Marketing, and Docs (`apps/desktop/src/renderer/lib/posthog.ts:19`, `apps/web/src/instrumentation-client.ts:19`, `apps/marketing/src/instrumentation-client.ts:28`, `apps/docs/src/instrumentation-client.ts:19`).
- **Local-Only Resource Journal:** `startResourceJournal()` writes host CPU/memory/PID/daemon diagnostics locally to `~/.paradev/resources.jsonl` every 60s (`apps/desktop/src/main/lib/resource-metrics/resource-journal.ts:18-24`, `apps/desktop/src/main/index.ts:749`) and does not auto-upload it.

### D. Behavioral & User Profile Fields
- Identified via Better Auth `user.id` with `$set` / person properties: `email`, `name`, `desktop_version`, `created_at` (`PostHogUserIdentifier.tsx:18-24`), `plan_tier` (`PostHogPlanFlags.tsx:11-12`), `plan` & `active_organization_id` (`analytics.ts:26-27`), `app_locale` & `locale_source` (`PostHogLocaleTagger.tsx:30-33`), `surface: "v2"`, `onboarded_surface`, `surface_first_v2_at`, `surface_ever_v2` (`PostHogSurfaceTagger.tsx:10-19`), `domain` (`apps/web/src/lib/posthog-client.ts:10-15`), billing funnel attribution & Stripe events (`checkout_started`, `paywall_opened`, `captureBillingEvent` in `billing-analytics.ts:67-113`), and acquisition UTMs / `auth_started` (`sign-in/page.tsx:88`).

---

## 6. Docs / Privacy Policy vs Code Discrepancies

1. **False Claim of Desktop Settings Controls (`apps/marketing/content/compare/superset-vs-herdr.mdx:57`):**
   - **Doc says:** Paradev row for **Telemetry** states `"Product analytics, with controls in settings"`.
   - **Code does:** Migration `packages/local-db/drizzle/0018_add_delete_local_branch_setting.sql:2` dropped `telemetry_enabled`, Settings UI has no telemetry toggle, `setTelemetryEnabled` is a no-op stub (`apps/desktop/src/lib/trpc/routers/settings/index.ts:668-672`), and `getTelemetryEnabled` always returns `true` (`:664-666`).
2. **Contradiction on Diagnostic Opt-In & Local Data Isolation (`apps/marketing/content/legal/privacy.mdx:31`):**
   - **Doc says:** *"our desktop application runs locally and we do not have access to your source code, terminal commands, or file contents unless you explicitly choose to share diagnostic information with us."*
   - **Code does:** Diagnostic/error telemetry (`SENTRY_DSN_DESKTOP` and `HOST_SERVICE_SENTRY_DSN`, including `outputTail` crash output in `host-service-coordinator.ts:1242-1245`, renderer `capture_exceptions: true`, custom agent `command_name`, and git `branch` names in PostHog) is **unconditionally enabled with no user choice or opt-out control**.
3. **Undisclosed Third-Party Trackers & Endpoints in `subprocessors.mdx` & `privacy.mdx`:**
   - `apps/marketing/content/legal/subprocessors.mdx:7-18` lists 10 subprocessors (Anthropic, Autumn, Cloudflare, Neon, PostHog, Resend, Sentry, Stripe, Upstash, Vercel) and omits **Google Ads** (`googletagmanager.com` / `doubleclick.net`), **Reddit Pixel** (`redditstatic.com` / `alb.reddit.com`), **LeadConnector / GoHighLevel** (`x.paradev.com/js/external-tracking.js` -> `backend.leadconnectorhq.com`), **Expo Updates**, and direct quota API calls using local third-party CLI tokens (OpenAI/ChatGPT, Google `cloudcode-pa`, Cursor `api2.cursor.sh`, xAI `grok.com`). Furthermore, `privacy.mdx` makes no mention of `Google Ads`, `Reddit`, or `LeadConnector` while `<head>` on `paradev.com` fires Google Ads `gtag` and Reddit Pixel `PageVisit` immediately even if the user declines cookies (`apps/marketing/src/app/[lang]/layout.tsx:150-169`).
4. **SDK Docs Accurate (`apps/docs/content/docs/sdk/reference.mdx:29`), CLI Undocumented:**
   - `reference.mdx:29` accurately documents `sdk_method_called` and `SUPERSET_TELEMETRY=0`. However, no comparable CLI opt-out exists in `packages/cli` despite `packages/cli/src/lib/analytics.ts` collecting `cli_command_invoked`.
5. **Misleading Telemetry Tag (`surface_source: "opted-in"`):**
   - `apps/desktop/src/renderer/components/PostHogSurfaceTagger/PostHogSurfaceTagger.tsx:10` tags every desktop user with `surface_source: "opted-in"`, even though desktop users have no mechanism to opt out.

---

## 7. Zero-Telemetry / Negative-Check Verification

Clean static `rg` checks performed across `apps/` and `packages/` (excluding `node_modules`, `*.po`, `*.svg`, `*.jsonl`, `locales/`, and marketing/plugin docs):
- **Clean sub-apps with zero telemetry/Sentry/PostHog SDKs:** `apps/gate` and `apps/agent-worker` returned **0 matches** for `posthog|sentry|telemetry|analytics|otel|opentelemetry` (`rg -n -i --glob '!**/node_modules/**' 'posthog|sentry|telemetry|analytics|otel|opentelemetry' apps/gate apps/agent-worker`).
- **Absent third-party analytics/crash/ID SDKs:** Searching all `package.json` manifests and source code confirmed **0 matches** for `amplitude`, `mixpanel`, `segment`, `datadog`, `opentelemetry`, `bugsnag`, `rollbar`, `raygun`, `plausible`, `umami`, `fathom`, `clarity`, `hotjar`, `aptabase`, `countly`, `rudderstack`, `snowplow`, `statsig`, `launchdarkly`, and `node-machine-id`.
- **Absent privacy signals/configs:** `DO_NOT_TRACK` / `doNotTrack` / `navigator.doNotTrack` returned **0 matches** repository-wide; `$geoip_disable`, `$ip`, `mask_all_text`, and `maskAllInputs` returned **0 matches** across `apps` and `packages`.

---

```json
{
  "slug": "paradev",
  "platform": "Paradev (Superset)",
  "group": "A",
  "repo": "ParadevAI/Paradev",
  "sha": "296f25b450ca126aa2df291dbe905ab48ff15294",
  "commit_date": "2026-11-24T16:44:41-06:00",
  "license": "MIT",
  "status": "Active (superset-sh/superset redirects here)",
  "default_state": "ON by default across Desktop (main + renderer + host-service), Mobile (100% session replay), CLI, SDK, Cloud API (1% tRPC procedure sampling), Web, Docs, and Marketing",
  "opt_out_controls": "JS SDK only: SUPERSET_TELEMETRY=0 or false (packages/sdk/src/lib/telemetry.ts:52-55). Desktop UI/DB toggle was dropped (0018_add_delete_local_branch_setting.sql:2) and replaced with hardcoded true / no-op stubs (settings/index.ts:663-672). No opt-out in Desktop, CLI, Host-Service, Mobile, Web, or Docs. Marketing banner sets localStorage opt-out for PostHog/LeadConnector only; macOS-only DISABLE_UPDATE_CHECK disables desktop auto-updater.",
  "do_not_track": "Ignored (0 matches across repository)",
  "opt_out_bypasses": "1) Desktop getTelemetryEnabled hardcoded to true and setTelemetryEnabled is a silent no-op (apps/desktop/src/lib/trpc/routers/settings/index.ts:663-672), while TelemetrySync forces posthog.opt_in_capturing(); 2) Marketing unconditionally loads Google Ads gtag and Reddit Pixel PageVisit in <head> before/despite cookie opt-out (apps/marketing/src/app/[lang]/layout.tsx:150-169); 3) CLI/Host-Service/Desktop ignore SUPERSET_TELEMETRY=0; 4) Server-side clientTelemetry captures 1% of API calls independently of client settings.",
  "persistent_ids": "1) Raw OS machine ID (IOPlatformUUID / /etc/machine-id / Registry MachineGuid) used locally for AES-256-GCM auth-token encryption (packages/shared/src/host-info.ts:14-69); 2) Fixed-salt HMAC-SHA256 hostId (salt 'superset-desktop-device-id-v1', truncated to 32 hex chars) transmitted to cloud host.ensure, PostHog (host_id, client_machine_id), and leaderboard (host-info.ts:8, 74-103); 3) Shared Better Auth user.id + email + name + cross-subdomain cookie ('superset') stitching Desktop, Mobile, Web, Marketing, Docs, CLI, SDK, and Billing into one PostHog project.",
  "hashing": "HMAC-SHA256 with fixed static salt 'superset-desktop-device-id-v1' for machine/host ID; unsalted SHA-256 for Claude OAuth access tokens (claude.ts:244), Claude [accountUuid, organizationUuid] (read-claude-quota-identity.ts:17), Codex auth.json (usage.ts:202-204), CLAUDE_CONFIG_DIR Keychain suffixes (profiles.ts:75-94), and Stripe PostHog UUIDs (billing-analytics.ts:38-47); unsalted MD5 of cwd for Cursor chat dirs (cursor.ts:234).",
  "external_auth_reads": "Automatically every 5m via RailUsage (showRailUsage defaults true) and on-demand: reads macOS Keychains ('Claude Code-credentials*', 'cursor-access-token'/'cursor-user', 'gemini'/'antigravity') and files (~/.claude/.credentials.json, ~/.config/claude/credentials.json, ~/.claude.json, ~/.codex*/auth.json, ~/.grok/auth.json, <data>/opencode/auth.json, ~/.copilot/session-store.db, <data>/devin/cli/sessions.db, ~/.hermes/state.db, ~/.fx/sessions, ~/.pi, ~/.omp, ~/.gemini/antigravity-cli/brain, gh api user, git user.name) plus on-demand Chromium Cookies/History & Safe Storage Keychains. Scans competitor worktree dirs (Conductor, T3Code, Orca, Emdash, Claude Squad, Cursor, Herdr, Paseo) and reports competitor source names to PostHog (external_worktrees_offered).",
  "telemetry_endpoints": "https://us.i.posthog.com & https://us-assets.i.posthog.com (direct + /ingest proxy on web/marketing/docs), *.ingest.sentry.io (+ /monitoring tunnel on web/marketing/docs), https://www.googletagmanager.com / googleadservices.com / doubleclick.net (Marketing), https://www.redditstatic.com / alb.reddit.com (Marketing), https://x.paradev.com/js/external-tracking.js -> https://backend.leadconnectorhq.com/external-tracking/events (Marketing), https://cloudflareinsights.com (Marketing), https://github.com/paradevai/paradev/releases (Desktop/CLI updaters), Expo Updates (Mobile), https://api.paradev.com / relay.superset.sh / realtime.superset.sh / streams.superset.sh, plus third-party AI quota APIs (Anthropic, ChatGPT, Cursor, Google cloudcode-pa, Grok).",
  "ip_geo_handling": "Zero $geoip_disable or $ip suppression across PostHog; client IP exposed directly or via API/Next.js rewrites. Sentry sendDefaultPii is true in Web, API, Marketing, and Docs, and false in Desktop Main, Mobile, Relay, Realtime, and Usercontent.",
  "custom_model_handling": "Verbatim custom model IDs sent to PostHog in Mobile (new_session_model_selected, chat_message_sent, workspace_create_requested) and Slack API (slack_model_changed, slack_message_sent), and to Postgres in opt-in leaderboard.publish; Desktop/Host-Service agent_launched masks custom preset to 'custom' + command_name basename.",
  "docs_discrepancies": "1) Marketing comparison table (apps/marketing/content/compare/superset-vs-herdr.mdx:57) falsely claims Paradev has 'Product analytics, with controls in settings' after the setting and DB column were deleted; 2) Privacy Policy (privacy.mdx:31) claims no access to terminal commands/diagnostic data 'unless you explicitly choose to share diagnostic information' while Sentry (including host crash outputTail) and PostHog (git branch, custom command_name) run unconditionally without opt-out; 3) subprocessors.mdx omits Google Ads, Reddit Pixel, LeadConnector, Cloudflare Web Analytics, and Expo Updates, and marketing <head> executes ad tags even after opt-out.",
  "privacy_rating": "F",
  "one_line_verdict": "Severe red flags: Paradev explicitly deleted its desktop telemetry opt-out setting and replaced it with hardcoded true/no-op stubs while marketing docs still claim settings controls, runs 100% mobile session replay and unconditional CLI/desktop/host telemetry with zero DNT support, leaks git branches/custom commands/verbatim models across a unified PostHog identity, and continuously polls third-party AI CLI credentials and Keychains."
}
```
