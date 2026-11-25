# Emdash Forensic Telemetry & Privacy Audit

- **Platform:** Emdash (`generalaction/emdash`)
- **Group:** A
- **Repository:** https://github.com/generalaction/emdash
- **Audited Branch / Commit SHA:** `main` @ `c8769892fd61f63b5c95ca1fccf727cee4d07809`
- **Commit Date:** `2026-11-23T15:30:26-08:00`
- **Commit Subject:** `Merge pull request #3443 from generalaction/feat/ibm-bob-provider`
- **License:** `Apache-2.0` (`LICENSE.md`)
- **Status:** Active
- **Overall Privacy Rating:** **D — Aggressive / Leaky**

---

## 1. Default State & Opt-Out Mechanism

### Default State
- **Opt-out (ON by default in packaged releases):** `DesktopTelemetryService` initializes `private enabled = true` (`apps/emdash-desktop/src/main/lib/telemetry.ts:37`) and resolves runtime telemetry enablement via:
  ```ts
  this.enabled = !isViteDevBuild && config.telemetryEnabled;
  ```
  (`apps/emdash-desktop/src/main/lib/telemetry.ts:351`).
- In `loadAppConfig()`, `TELEMETRY_ENABLED` defaults to `true` whenever unset (`apps/emdash-desktop/src/main/bootstrap/core/config.ts:35-41`):
  ```ts
  const telemetryEnabled = z
    .string()
    .optional()
    .transform((value) => {
      const normalized = (value ?? 'true').toLowerCase();
      return normalized !== 'false' && normalized !== '0' && normalized !== 'no';
    });
  ```
- **CI/CD secret injection:** Production, Canary, and Linux release workflows pass repository secrets `POSTHOG_PROJECT_API_KEY` and `POSTHOG_HOST` into `.github/actions/setup-build/action.yml`, which appends `VITE_POSTHOG_KEY`, `VITE_POSTHOG_HOST`, and `VITE_BUILD` into `apps/emdash-desktop/.env.production` (`.github/actions/setup-build/action.yml:61-70`, `.github/workflows/release-prod.yml:52-54,64-68,131-135`, `.github/workflows/release-canary.yml:52-54,64-69,135-139`, `.github/workflows/release-linux.yml:57-62`).
- **Dev builds disabled by default:** `isViteDevBuild = import.meta.env.DEV` disables telemetry sending at `telemetry.ts:29,61,351`, though dev fallback keys `POSTHOG_PROJECT_API_KEY` and `POSTHOG_HOST` are parsed from `.env.local` / `process.env` (`apps/emdash-desktop/src/main/lib/env.ts:14-19,36-38`, `telemetry.ts:353-354`).
- **Zero first-run telemetry disclosure:** First-run onboarding (`apps/emdash-desktop/src/renderer/App.tsx:29,54-62`, `apps/emdash-desktop/src/core/features/workbench/browser/onboarding/onboarding.tsx:4-19`, `onboarding-shell.tsx:6-20`, `sign-in-step.tsx:78-110`) renders only `Sign in` and `Import` steps with no notice of telemetry or opt-out toggle. Telemetry begins emitting on first launch (`daily_active_user` during `initialize()`, `app_started` after boot settles, view navigation events, and window focus/blur events).

### Opt-Out Controls
1. **UI Toggle (`Settings -> General -> App -> Privacy & Telemetry`):**
   - Rendered in `apps/emdash-desktop/src/core/features/settings/browser/pages/general-settings-page.tsx:28-31` and `apps/emdash-desktop/src/core/features/settings/browser/components/TelemetryCard.tsx:10-58`.
   - Backed by `useTelemetryConsent()` (`apps/emdash-desktop/src/core/primitives/telemetry/browser/useTelemetryConsent.ts:26-85`) over wire RPC `telemetry.setEnabled` (`apps/emdash-desktop/src/core/features/telemetry/node/wire-controller.ts:11-13`), which persists `'true'` or `'false'` under SQLite KV key `telemetry:enabled` (`apps/emdash-desktop/src/main/lib/telemetry.ts:53,373,387,514-517`).
   - Note: When `TELEMETRY_ENABLED=false` (or in Vite dev builds), `envDisabled` disables the UI switch (`TelemetryCard.tsx:46`, `telemetry.ts:496`).
2. **Environment Variable (`TELEMETRY_ENABLED`):**
   - Setting `TELEMETRY_ENABLED=false`, `0`, or `no` (case-insensitive) sets `config.telemetryEnabled = false` (`apps/emdash-desktop/src/main/bootstrap/core/config.ts:35-41,64-72`).
   - Note: `loadAppConfig` passes `argv: []` (`config.ts:67`), so CLI flags like `--disable-telemetry` do not exist.
3. **Do Not Track (`DO_NOT_TRACK` / `doNotTrack` / `navigator.doNotTrack`):**
   - **Not supported.** Zero matches in source (`apps/` and `packages/`).

### Opt-Out Leaks, Side Effects & Unchecked Channels
1. **Opting out in Settings transmits a final `setting_changed` telemetry event:**
   - In `TelemetryCard.tsx:40-45`:
     ```tsx
     <Switch
       checked={prefEnabled}
       onCheckedChange={(checked) => {
         captureTelemetry('setting_changed', { setting: 'telemetry' });
         void setTelemetryEnabled(checked);
       }}
     ```
   - `captureTelemetry('setting_changed', ...)` runs **before** the async `setEnabled({ enabled: false })` RPC reaches the main process, so turning telemetry **OFF** emits one last `setting_changed` payload to PostHog `/capture/` while `isEnabled()` is still `true` (and conversely, turning telemetry **ON** drops `setting_changed` because `isEnabled()` is still `false`!).
2. **Unconditional SQLite state persistence & 60-second heartbeat loop even when disabled:**
   - Inside `DesktopTelemetryService.initialize()` (`apps/emdash-desktop/src/main/lib/telemetry.ts:382-421`), regardless of `TELEMETRY_ENABLED=false` or `telemetry:enabled === 'false'`:
     - Generates and writes persistent `telemetry:instanceId` (UUIDv4) if absent (`telemetry.ts:382-385`);
     - Writes `telemetry:lastSessionId` (`telemetry.ts:405`);
     - Unconditionally starts a 60,000 ms `setInterval` writing `telemetry:lastHeartbeatTs` to SQLite KV (`telemetry.ts:419-421`). Only `dispose()` (`telemetry.ts:424-433`) clears the interval and deletes `lastSessionId`/`lastHeartbeatTs`.
3. **Pre-existing perf-vitals timers/observers remain scheduled after mid-session UI opt-out:**
   - If a session won the 5% `perfSampled` roll at boot (`telemetry.ts:34,411`), `startPerfVitalsTelemetry()` (`apps/emdash-desktop/src/main/bootstrap/boot/phases/perf-vitals.ts:16-30`), `desktopWorkers` worker vitals (`apps/emdash-desktop/src/main/gateway/desktop-workers.ts:190-194,204`), and `initRendererPerfVitals()` (`apps/emdash-desktop/src/renderer/utils/perf-vitals.ts:51-125`) stay running even if the user later toggles telemetry OFF via `setTelemetryEnabledViaUser(false)` (`telemetry.ts:514-517`). Outbound PostHog network calls ARE blocked at `isEnabled()` (`telemetry.ts:59-69,236`), so these events drop locally at capture time.
4. **Independent non-telemetry outbound channels unaffected by `TELEMETRY_ENABLED=false`:**
   - Auto-updater (`apps/emdash-desktop/src/main/host/updates/update-service.ts:69-87,164-205`) initializes unconditionally in packaged builds (`!import.meta.env.DEV`), polls GitHub Releases (`generalaction/emdash`) / R2 (`https://releases.emdash.sh`) 30 seconds after startup and every 1 hour thereafter, with no UI or env opt-out (`UpdateCard.tsx`).
   - Emdash Account cold-boot session checks / OAuth (`emdash-account-service.ts:57-87`), GitHub CLI account import (`background.ts:36-57`), Feedback relay (`submit-feedback.ts`), and Waitlist API (`join-waitlist.ts`) also operate independently when triggered or configured.

---

## 2. External Auth / Local Non-Workspace File Reads

### A. Unconditional Startup Harvesting of GitHub CLI (`gh`) Auth Tokens
- On every desktop launch, `bootBackground` spawns `'github-account-reconciliation'` (`apps/emdash-desktop/src/main/bootstrap/boot/phases/background.ts:36-57`), unconditionally calling `services.github.cliImport.importAccounts()` (`background.ts:48`) regardless of `TELEMETRY_ENABLED` or whether the user chose GitHub sign-in/integration.
- `GitHubCliAccountImportService.readCliStatus()` (`apps/emdash-desktop/src/core/features/github/node/accounts/github-cli-account-import.ts:85-96`) executes:
  ```ts
  await this.exec('gh', ['auth', 'status', '--json', 'hosts', '--show-token'], {
    timeout: GITHUB_CLI_AUTH_STATUS_TIMEOUT_MS,
  });
  ```
- For every authenticated host entry in `gh` CLI output (`github-cli-account-import.ts:59-81`), Emdash:
  1. Extracts the **plaintext `gh` CLI OAuth token** (`entry.token.trim()`, line 65);
  2. Sends an outbound `Octokit` `GET /user` request (`githubIdentityClient.getAuthenticatedUser(token, host)`, line 66; `apps/emdash-desktop/src/core/features/github/node/services/github-identity-client.ts:39-55`) to `https://api.github.com` or `https://<host>/api/v3`;
  3. Imports and persists the token into Electron `safeStorage` (`encryptedAppSecretsStore`) via `connectGitHubAccount` -> `IntegrationConnectionService.connectVerified` (`apps/emdash-desktop/src/core/features/github/node/accounts/github-auth-connection.ts:31-56`, `apps/emdash-desktop/src/core/features/integrations/node/integration-connection-service.ts:71-145`);
  4. **Emits `integration_connected` `{ provider: 'github' }` telemetry on every startup** whenever `gh` CLI is signed in and telemetry is enabled, because `connectVerified` calls `this.telemetry.capture('integration_connected', { provider: integrationId })` unconditionally on both insert and update (`integration-connection-service.ts:143`).
- **Critical attribution check:** Emdash does **not** transmit the raw `gh` token or `gh` CLI login username to PostHog (`telemetry.identify()` is wired only to Emdash Account login in `wiring.ts:28-35`). However, silently shelling out to extract another tool's plaintext OAuth tokens on every boot, contacting GitHub APIs without prompt, and emitting `integration_connected` startup telemetry is an aggressive local credential read.

### B. Other Non-Workspace Local Files & Tool Configs Read/Written
1. **SSH configuration (`~/.ssh/config` and `Include` globs):**
   - `parseSshConfigFile()` (`apps/emdash-desktop/src/core/services/ssh/node/config/sshConfigParser.ts:37-95`) reads `~/.ssh/config` and recursively expands `Include` directives when SSH config host import is invoked in UI.
2. **External Coding Agent Config Directories (Hooks, MCP, Trust Markers):**
   - `AgentHookInstaller` (`packages/core/src/services/agent-plugins/node/hook-installer.ts:31-100`) and provider adapters resolve home/XDG config roots (`packages/core/src/services/agent-plugins/api/plugins/helpers/config-root.ts:20-50`) and read/modify external agent configs:
     - Claude Code: `~/.claude/settings.json` (`CLAUDE_CONFIG_DIR` fallback `.claude`, `packages/plugins/src/agents/impl/claude/hooks.ts:11,50-60`) and `~/.claude.json` for MCP & workspace trust (`packages/plugins/src/agents/impl/claude/index.ts:177`, `packages/plugins/src/agents/impl/claude/trust.ts:32-36`).
     - OpenAI Codex: `~/.codex/config.toml` and legacy `~/.codex/hooks.json` (`CODEX_HOME` fallback `.codex`, `packages/plugins/src/agents/impl/codex/hooks.ts:23-67,134,184,225`).
     - Other agent config roots probed/updated for hooks/MCP/trust when those agents are configured or launched (`packages/plugins/src/agents/impl/`): `~/.cursor` (`cursor/hooks.ts:90`, `cursor/trust.ts:17-20`), `~/.copilot` (`copilot/hooks.ts:24`), `~/.gemini/config` (`antigravity/hooks.ts:69`), `~/.qwen` (`qwen/hooks.ts:37`), `~/.qoder` (`qoder/hooks.ts:45`), `~/.vibe` (`mistral/hooks.ts:34`), `~/.kiro` (`kiro/hooks.ts:67`), `~/.kimi-code` (`kimi/hooks.ts:65`), `~/.grok` (`grok/hooks.ts:81`), `~/.agents` (`goose/hooks.ts:51`), `~/.factory` (`droid/hooks.ts:18`), `~/.augment` (`auggie/hooks.ts:21`), `~/.codebuddy` (`codebuddy/hooks.ts:78`), `~/.commandcode` (`commandcode/hooks.ts:16`), `~/.pi/agent` (`pi/index.ts:61`), `~/.prime/agent` (`prime-agent/index.ts:158`), `~/.omp/agent` (`oh-my-pi/index.ts:130-131`), and XDG `devin` / `muse` (`devin/hooks.ts:18`, `muse/hooks.ts:69`).
3. **External Agent Auth & Usage Probes:**
   - Auth checks in `packages/plugins/src/agents/` invoke provider CLIs rather than parsing raw token files directly (e.g., `claude auth status` + `ANTHROPIC_API_KEY` env in `packages/plugins/src/agents/impl/claude/auth.ts:18-29`, `opencode auth list` + provider env vars in `packages/plugins/src/agents/impl/opencode/auth.ts:16-37`, and `codex` auth status CLI / `OPENAI_API_KEY` in `packages/plugins/src/agents/impl/codex/index.ts:165-171`).
   - Usage probes (`apps/emdash-desktop/src/core/features/usage/node/usage-overview.ts:34-100`) run on demand when observed via `@anthropic-ai/claude-agent-sdk` `query()` (`packages/plugins/src/agents/impl/claude/usage.ts:70-149`, loading `'user'` settings & returning `email`, `organization`, `plan`) and Codex app-server JSON-RPC (`account/read` & `account/rateLimits/read`, `packages/plugins/src/agents/impl/codex/usage.ts:72-105`, returning `email`, `planType`, `accountId`).
4. **Installed App & Font Discovery / Shell Environment / Legacy Emdash Migration:**
   - Installed IDEs/editors/terminals detected via macOS LaunchServices (`NSWorkspace`), PATH lookup, Windows `vswhere` (`apps/emdash-desktop/src/main/core/app/installed-apps.ts:27-90`, exposed to renderer on demand via `appOperations.checkInstalledApps` at `controller.ts:158`).
   - Interactive login shell captured on startup via `$SHELL -ilc 'env'` (`apps/emdash-desktop/src/main/lib/userEnv.ts:68-78`) to populate subprocess `PATH`/environment.
   - Audio file import restricted strictly under `homedir()` via `realpath` prefix check (`apps/emdash-desktop/src/main/core/app/service.ts:642-662`).
   - Legacy DB/token migrations read Emdash's own prior `userData` files (`emdash.db` / `emdash-beta` DB) and Emdash legacy GitHub tokens (`apps/emdash-desktop/src/main/db/legacy-port/legacy-source/path.ts:7-58`, `apps/emdash-desktop/src/core/features/github/node/accounts/github-legacy-token-import-step.ts:32-60`).

---

## 3. Persistent Device / Install IDs & Hashing

| Identifier | Storage Location | Lifecycle & Generation | Transmitted Where | Hashing / Obfuscation |
|---|---|---|---|---|
| `telemetry:instanceId` (UUIDv4) | SQLite KV table (`kv` key `telemetry:instanceId`, `apps/emdash-desktop/src/main/db/kv.ts:65-75`, `telemetry.ts:53,372,382-385`) | Generated on first boot via `randomUUID()`, **even when `TELEMETRY_ENABLED=false` or user opted out**. Never deleted on opt-out. Also copied via `Help -> Copy Installation ID` (`apps/emdash-desktop/src/main/host/menu.ts:40-49`). | PostHog `distinct_id` on `/capture/` (`telemetry.ts:243,267`) and `/decide/?v=3` (`telemetry.ts:291`); also exposed over RPC `telemetry.getStatus().instance_id` (`telemetry.ts:501`). | **None (raw UUIDv4)** |
| `sessionId` & `telemetry:lastSessionId` (UUIDv4) | In-memory (`telemetry.ts:43,356`) + SQLite KV (`telemetry:lastSessionId`, `telemetry.ts:375,405`) | Generated per app launch via `randomUUID()`, persisted to KV even when telemetry is disabled; deleted on clean quit (`telemetry.ts:432`), used on next startup to emit synthetic `app_closed { was_crash: true }` (`telemetry.ts:393-401`). | Attached as `session_id` to every captured event (`telemetry.ts:465-474`, `telemetry-client.ts:21-52`). | **None (raw UUIDv4)** |
| `desktop-client-id` (UUIDv4) | SQLite KV (`kv` key `desktop-client-id`, `apps/emdash-desktop/src/main/core/runtime/desktop-client-id.ts:4-22`) | Generated once at `bootInfrastructure()` (`apps/emdash-desktop/src/main/bootstrap/boot/phases/infrastructure.ts:35`). | Passed only to SSH `workspace-server` connection management (`infrastructure.ts:43`), **not** sent to PostHog/Emdash telemetry. | **None (raw UUIDv4)** |
| Emdash Account PII (`email`, `github_username`, `account_id`) | Electron `safeStorage` + KV (`AccountSessionStore`) + cached in memory on `DesktopTelemetryService` (`telemetry.ts:45-47,441-448`) | Emitted via `accountChanged` hook both on OAuth sign-in (`emdash-account-service.ts:137-142`) and **on every cold boot** when a session is stored (`emdash-account-service.ts:61-68`) -> `wireAccountTelemetry()` (`apps/emdash-desktop/src/main/bootstrap/boot/wiring.ts:28-35`) -> `telemetryService.identify()`. | Sent in PostHog `$identify` (`$set: { email, ... }`, `telemetry.ts:265-272`), `/decide/?v=3` (`person_properties: { github_username, account_id, email }`, `telemetry.ts:292-296`), and appended to **every subsequent event** via `getBaseProps()` (`telemetry.ts:83-84`). | **None — plaintext PII.** Stitches anonymous pre-login `instanceId` events to real user email & GitHub handle! |
| Entity IDs (`project_id`, `task_id`, `conversation_id`, `automation_id`, `terminal_id`) | Local SQLite DB (`tasks`, `projects`, `conversations`, `automations`) | Emitted in lifecycle/focus events (`task_created`, `task_deleted`, `project_deleted`, `agent_run_started`, `automation_*`, `terminal_*`, `telemetry-scope.ts:11-35`). | PostHog `/capture/` event `properties` (`telemetry.ts:111-113,151,155`). | **None (raw internal IDs)** |
| Hardware / OS Machine IDs | — | Checked `node-machine-id`, `/etc/machine-id`, `IOPlatformUUID`, `MachineGuid`. | **None found.** | N/A |

- **No cross-provider email/ID hashing exists anywhere in telemetry:** Identity linkage uses **plaintext** `email`, `github_username`, and `account_id` stitched directly to the persistent `instanceId` (`distinct_id`).

---

## 4. Exact Outbound Telemetry / Analytics / Crash / OTEL / Relay / Update Endpoints

| Subsystem | Exact Endpoint URL | Implementation & Trigger | Controlled by `TELEMETRY_ENABLED`? |
|---|---|---|---|
| **Product Analytics & Feature Flags (PostHog)** | `POST ${VITE_POSTHOG_HOST}/capture/` and `POST ${VITE_POSTHOG_HOST}/decide/?v=3` (build-time CI secrets `POSTHOG_HOST` / `POSTHOG_PROJECT_API_KEY` injected in `.github/actions/setup-build/action.yml:64-70`) | Direct `fetch()` in `apps/emdash-desktop/src/main/lib/telemetry.ts:238-253,262-279,288-302`. No `posthog-js`/`posthog-node` SDK. `/capture/` sends behavioral, boot, focus, and sampled perf vitals + `$identify`; `/decide/?v=3` runs inside `identify()`. | **Yes** (except final `setting_changed` emitted right before UI opt-out RPC completes; KV ID/heartbeat state still written locally). |
| **Crash / Exception Tracking** | Same PostHog endpoint (`POST ${VITE_POSTHOG_HOST}/capture/` with `event: '$exception'`) | `DesktopTelemetryService.captureException()` (`telemetry.ts:480-491`). **0 call sites** in production code (`rg` verified). No Sentry, Bugsnag, Rollbar, or OpenTelemetry SDKs exist. Crash exit inference uses synthetic `app_closed { was_crash: true }` via KV heartbeat (`telemetry.ts:393-401`). | **Yes** |
| **Auto-Updater (Desktop)** | `https://github.com/generalaction/emdash/releases` (and GitHub Atom feed / release YAML + release notes `https://api.github.com/repos/generalaction/emdash/releases/tags/v${version}`) & Cloudflare R2 `https://releases.emdash.sh` (`UPDATE_CHANNEL` = `v1-stable` / `v1-canary`) | `electron-updater` initialized unconditionally in packaged builds (`apps/emdash-desktop/src/main/host/updates/update-service.ts:69-87,164-205,355-357`, `apps/emdash-desktop/electron-builder.config.ts:19-31`, `electron-builder.canary.config.ts:20-37`, `app-identity.ts:12-14`, `app-identity.canary.ts:5-11`). Polls 30s after launch and every 1 hour (`update-service.ts:22-23,86,164-174`). | **No** (no opt-out setting or env toggle). |
| **Remote SSH Agent Provisioning** | `https://releases.emdash.sh/workspace-server` (overridable via `EMDASH_WORKSPACE_SERVER_ARTIFACTS_URL` or settings `installBaseUrl`) | Fetches version pointer and installs remote `workspace-server` artifact over SSH when using remote SSH hosts (`apps/emdash-desktop/src/core/services/hosts/node/workspace-server/provision/installer.ts:13-14`, `settings.ts:11-13`, `apps/workspace-server/install.sh:4`). | **No** (user-triggered SSH provisioning). |
| **Emdash Account & OAuth Relay** | `https://auth.emdash.sh` (`/sign-in`, `/api/v1/auth/electron/exchange`, `/api/v1/auth/electron/account-link/start`, `/api/v1/auth/electron/account-link/authorize`, `/api/v1/auth/electron/provider-token`, `/api/auth/get-session`, `/health`) | PKCE browser OAuth & token broker (`apps/emdash-desktop/src/core/features/account/node/config.ts:1,15-17`, `account-oauth-client.ts:20-47`, `account-auth-server-client.ts:17-164`, `apps/emdash-desktop/src/main/core/shared/oauth-flow.ts:25-52`). Production builds ignore `EMDASH_AUTH_SERVER_URL` (`config.ts:17`). Auth server retains OAuth refresh tokens (`packages/plugins/src/integrations/capabilities/auth.ts:111`). | **No** (only contacted when signing in, linking integrations, validating stored session, or opening Settings Account tab `/health`). |
| **Feedback Relay (Cloudflare Worker)** | `https://emdash-feedback-relay.real-general-action.workers.dev` (overridable via `VITE_FEEDBACK_RELAY_URL` / dev `FEEDBACK_RELAY_URL`) | `POST` multipart `FormData` when user submits in-app Feedback modal (`apps/emdash-desktop/src/main/core/app/submit-feedback.ts:3-31`, `apps/emdash-desktop/src/main/core/app/controller.ts:202-212`). | **No** (user-initiated modal submission). |
| **Waitlist Signup API** | `https://emdash.com/api/waitlist` (overridable via `VITE_WAITLIST_API_URL` / dev `WAITLIST_API_URL`) | `POST` JSON `{ email, note?, source }` (`"desktop app v<version>"`) when user submits sidebar Emdash 2.0 waitlist form (`apps/emdash-desktop/src/main/host/waitlist/controller-operations.ts:5-15`, `apps/emdash-desktop/src/core/features/waitlist/node/join-waitlist.ts:21-69`). | **No** (user-initiated waitlist modal). |
| **MCP & Skills Catalogs** | `https://registry.modelcontextprotocol.io`, `https://skills.sh/api/search`, `https://skills.sh/<repo>`, `https://api.github.com/repos/openai/skills/...`, `https://api.github.com/repos/anthropics/skills/...`, `https://raw.githubusercontent.com/...` | `CatalogService` (`apps/emdash-desktop/src/core/features/catalog/node/catalog-service.ts:32,188,457,500-660`). | **No** (user-initiated catalog browsing/search/install). |
| **Git / Issue Provider APIs & Device OAuth** | `https://api.github.com`, `https://github.com/login/device/code`, `https://github.com/login/oauth/access_token`, `https://gitlab.com/oauth/...`, `https://api.atlassian.com/oauth/token/accessible-resources`, plus configured Linear/Jira/Asana/Forgejo/Plane/Plain/Notion/Monday/Trello/YouTrack/Featurebase/GHE hosts | `github-identity-client.ts` (also triggered **unconditionally at startup** via `gh` CLI import), `packages/plugins/src/integrations/impl/{github,gitlab}/index.ts`, etc. | **No** (`gh` CLI startup check triggers `api.github.com` / GHE unconditionally). |

---

## 5. Exact Data & Properties Collected

### A. IP Address & Geolocation (`$geoip_disable`, `$ip`, `sendDefaultPii`)
- **Zero IP suppression on PostHog requests:** Raw `fetch()` POST payloads to `/capture/` and `/decide/?v=3` (`apps/emdash-desktop/src/main/lib/telemetry.ts:238-302`) do **not** include `$ip: null` or `$geoip_disable: true` (`rg` confirmed zero matches across the entire repo). PostHog ingests the caller's public IP address and applies server-side GeoIP enrichment by default. (`sendDefaultPii` is N/A because Sentry is absent.)
- **Explicit timezone capture:** `daily_active_user` transmits the system IANA timezone via `Intl.DateTimeFormat().resolvedOptions().timeZone` (`telemetry.ts:331-334`).

### B. Custom / Private Model Names
- **Not collected.** Neither `TelemetryEventProperties` (`apps/emdash-desktop/src/core/primitives/telemetry/api/telemetry.ts:41-212`) nor `sanitizeEventAndProps()` (`telemetry.ts:98-191`) contains any model-name property. Events record only `provider` ID (`claude`, `codex`, `opencode`, etc.) in `task_created`, `conversation_created`, `agent_run_started`, and `automation_created`. `rg` verified zero occurrences of `customModel` / `modelName` in source.

### C. Prompts, Code, File Paths & Stack Traces
- **Prompts & Code:** **Not collected** by PostHog telemetry. `task_created` and `automation_created` record only boolean `has_initial_prompt` (`task-telemetry.ts:26-29`, `automation-telemetry.ts:69`). No file contents, diffs, repo URLs, branch names, commit messages, or workspace paths appear in `allowedProps`.
- **Stack Traces (Dormant Unredacted Sink):** `sanitizeEventAndProps()` allows `$exception_message`, `$exception_type`, `$exception_stack_trace_raw`, and `$exception_fingerprint` up to **2,000 characters** via `passthroughProps` without running `redactAll()` (`telemetry.ts:186-200`), and `captureException()` passes `errorObj.stack || ''` verbatim (`telemetry.ts:480-491`). Because `captureException` currently has **0 call sites** in the codebase, stack traces are not emitted in practice today—though the sink itself lacks path/user-directory redaction.
- **Feedback Modal Logs (Opt-In Manual Submission):** When users submit feedback (`apps/emdash-desktop/src/core/features/workbench/browser/feedback-modal/feedback-modal.tsx:55-100`, `use-feedback-submit.ts:26-59,139-165`), Emdash attaches `feedbackDetails`, optional `contactEmail`, default `githubLogin` (`GitHub: @<login>`), `appVersion`, and OS display name/kernel version (`macOS <ver>`, `Windows <release>`, `Linux <release>`, `controller.ts:29-52`). If the user checks *"Include diagnostic logs"*, `getDiagnosticLogAttachment()` (`file-logger.ts:66-86`) sends the last 500 KB of `emdash.log` processed through `redactAll()` (`packages/shared/src/logger/redact.ts:14-78`), which masks `/Users/<user>`, `/home/<user>`, `C:\Users\<user>`, IPv4/IPv6, MACs, emails, PEM blocks, URL credentials, and common token prefixes (`ghp_`, `glpat-`, `sk-ant-`, `sk-`, `AKIA`, JWTs, etc.).

### D. Behavioral Profile, Identity & Device Context Fields
- **Base properties on every event (`getBaseProps()`, `telemetry.ts:71-86`):**
  `schema_version` (`1`), `app_version`, `build_variant` (`prod` | `canary`), `source` (`'desktop_app'`), `electron_version`, `platform`, `arch`, `is_dev`, `install_source` (defaults to `'dmg'` for all packaged builds unless `INSTALL_SOURCE` env is set — `services.ts:687`, `telemetry.ts:81`), `$lib` (`'emdash'`), plus **`github_username`** and **`account_id`** whenever signed into an Emdash Account.
- **Identity & feature flag payloads (`telemetry.ts:259-319`):**
  `$identify` sets `email` + all base properties on `distinct_id: instanceId`; `/decide/?v=3` transmits `github_username`, `account_id`, and `email` in `person_properties`.
- **Event inventory (`TelemetryEventProperties`, `core/primitives/telemetry/api/telemetry.ts:41-212`, `sanitizeEventAndProps`, `telemetry.ts:98-216`):**
  Every `capture()` call merges `{ event_ts_ms, session_id }` (`telemetry.ts:467-474`), and renderer calls merge `{ active_view, active_main_panel, focused_region }` (`telemetry-client.ts:62-65`, `focus-tracker.ts:80-86`) plus optional `{ project_id, task_id, conversation_id }` (`telemetry-scope.ts:3-35`). Tracked events cover:
  - **Lifecycle & activity:** `app_started` (`window_visible_ms`, `usable_workspace_ms`), `boot_watchdog_triggered` (`stuck_phase`, `backend_completed`, `window_loaded`), `app_closed` (`was_crash`), `app_window_focused`, `app_window_unfocused`, `daily_active_user` (`date`, `timezone`).
  - **Fine-grained UI navigation & focus (`navigation-telemetry.ts:8-18`, `focus-tracker.ts:48-72`):** `focus_changed` (`view`, `main_panel`, `focused_region`, `trigger`), `home_viewed`, `project_viewed`, `task_viewed`, `settings_viewed`, `automations_viewed` (`from_view`), `sidebar_toggled`, `setting_changed`.
  - **Workflow & agent actions:** `project_added` (`type`, `strategy`, `success`), `project_deleted`, `task_created` (`strategy`, `has_initial_prompt`, `has_issue`, `provider`), `task_provisioned`, `task_provision_timing` (`duration_ms`), `task_archived`, `task_status_changed`, `task_deleted`, `conversation_created` (`provider`, `is_first_in_task`), `conversation_deleted`, `agent_run_started` (`provider`), `terminal_created`, `terminal_deleted`, `automation_created` / `enabled_changed` / `run_started` / `run_completed` (`automation_id`, `trigger_kind`, `duration_ms`, `error_step`, `error_code`), `pr_created`, `pr_creation_failed`, `pr_merged`, `vcs_branch_published` / `fetch` / `push` / `pull` / `files_staged` / `unstaged` / `discarded`, `user_signed_in`, `user_signed_out`, `integration_connected`, `integration_disconnected`, `issue_linked_to_task`, `open_in_external`, `ssh_connection_attempted`, `mcp_server_added` / `removed`, `skill_installed` / `uninstalled` / `created`.
  - **Sampled performance vitals (5% of enabled sessions at startup, every 5 min, `telemetry.ts:34,411`, `perf-vitals.ts:7,16-30`, `packages/shared/src/perf/node/vitals.ts:15-84`, `renderer/utils/perf-vitals.ts:51-125`):** `perf_vitals` (`process_name`, `rss_mb`, `heap_used_mb`, `heap_total_mb`, `detached_contexts`, `cpu_percent`, `elu_percent`, `loop_delay_p95_ms`, `loop_delay_max_ms`, `interval_ms`, `spawns_<purpose>`, `app_process_count`, `app_total_rss_mb`, `renderer_rss_mb`, `gpu_rss_mb`) and `perf_renderer_vitals` (`long_tasks`, `long_task_total_ms`, `inp_ms`, `interval_ms`).

---

## 6. Docs / Privacy Policy vs Code Discrepancies

1. **"Anonymous usage data" claim vs. real PII `$identify` & IP ingestion:**
   - `TelemetryCard.tsx:19,47` presents the toggle as *"Help improve Emdash by sending anonymous usage data"* (`aria-label="Enable anonymous telemetry"`).
   - In code, whenever a user signs into an Emdash Account (including silently on every cold boot for existing sessions via `emdash-account-service.ts:61-68`), `telemetryService.identify()` sends PostHog `$identify` with **plaintext `email`, `github_username`, and `account_id`** bound to the persistent `instanceId`, attaches `github_username` & `account_id` to **every subsequent telemetry event** (`telemetry.ts:83-84`), omits IP/GeoIP suppression (`$ip: null` / `$geoip_disable: true`), and records system `timezone`.
2. **Conflicting documentation domains & missing in-repo Privacy Policy:**
   - `README.md:7,55,64,80` points to `https://emdash.com/docs/telemetry` (and `emdash.com/docs/...`), whereas `TelemetryCard.tsx:27` and `urls.ts:2` point to `https://docs.emdash.sh/telemetry` (`https://docs.emdash.sh`).
   - There is no `PRIVACY.md` or local telemetry documentation file inside the repository (`rg --files --max-depth 2` confirms only `README.md`, `AGENTS.md`, `CONTRIBUTING.md`, `CLAUDE.md`, `CONTEXT.md`, `LICENSE.md`, and internal `agents/` engineering docs).
3. **"Disable telemetry with `TELEMETRY_ENABLED=false`" vs. continued local state tracking:**
   - `README.md:74-78`, `AGENTS.md:239`, and `agents/workflows/flows.md:193` state that `TELEMETRY_ENABLED=false` (or Settings) disables telemetry. While outbound PostHog HTTP traffic is indeed stopped once disabled, `initialize()` still generates and persists `telemetry:instanceId`, `telemetry:lastSessionId`, and runs the 60-second `telemetry:lastHeartbeatTs` SQLite KV loop unconditionally (`telemetry.ts:382-421`), and UI opt-out emits one final `setting_changed` event (`TelemetryCard.tsx:43`).
4. **Hardcoded `install_source: 'dmg'` fallback on non-macOS packaged builds:**
   - `apps/emdash-desktop/src/main/bootstrap/boot/phases/services.ts:687` passes `installSource: app.isPackaged ? 'dmg' : 'dev'`, causing packaged Windows and Linux builds without an `INSTALL_SOURCE` env var to misreport `install_source: 'dmg'`.
5. **Updater & R2 migration wording in internal docs:**
   - `agents/risky-areas/updater.md:94` notes *"R2 uploads continue until telemetry confirms all clients have migrated to the GitHub-backed feed"*; auto-updating itself (`update-service.ts`) cannot be disabled in settings.

---

## 7. Zero-Telemetry / Negative-Check Verification

Static repository checks performed against `repos/emdash`:
- **Package manifests (`rg -n --glob "package.json"` across all `apps/` and `packages/`):** Zero third-party analytics, crash-reporting, or OpenTelemetry packages (`posthog-js`, `posthog-node`, `@sentry/*`, `sentry-electron`, `@opentelemetry/*`, `mixpanel`, `amplitude`, `bugsnag`, `rollbar`, `segment`, `datadog`, `honeycomb`, `statsig`, `launchdarkly`, `flagsmith`). (Only `electron-updater: ^6.8.11` at `apps/emdash-desktop/package.json:72` matched; `amplitude` appeared only as an MCP catalog entry URL `https://mcp.amplitude.com/mcp` in `apps/emdash-desktop/src/core/primitives/mcp/api/catalog.ts:326`.)
- **Negative code searches (`rg -n --glob '!*.test.*' --glob '!pnpm-lock.yaml'` across `apps/`, `packages/`, `package.json`):**
  - `DO_NOT_TRACK`, `doNotTrack`, `navigator.doNotTrack`: **0 matches** (DNT is ignored).
  - `$geoip_disable`, `geoip_disable`, `$ip`, `sendDefaultPii`: **0 matches** (IP suppression is missing).
  - `node-machine-id`, `/etc/machine-id`, `IOPlatformUUID`, `MachineGuid`: **0 matches** (no OS hardware fingerprinting).
  - `@sentry`, `sentry-electron`, `@opentelemetry`, `mixpanel`, `bugsnag`, `rollbar`: **0 matches**.
  - `customModel`, `modelName`: **0 matches** (custom model names are not captured).
  - `captureException`: Only **2 matches** — interface declaration (`apps/emdash-desktop/src/core/primitives/telemetry/api/telemetry.ts:238`) and method definition (`apps/emdash-desktop/src/main/lib/telemetry.ts:480`); **0 production call sites**.
- **`apps/workspace-server` audit (`rg -n --glob '!*.test.*' 'telemetry|posthog|sentry|analytics|fetch\(|https://' apps/workspace-server`):** **Clean** — zero telemetry code or analytics beacons in the remote workspace server; outbound URLs exist solely in `install.sh` (`https://releases.emdash.sh/workspace-server`) and build/upload scripts.
- **Prompts, workspace files, and git diffs:** Zero prompt text, source code, file paths, or git commit/diff contents are included in `TelemetryEventProperties` or `allowedProps`.

```json
{
  "slug": "emdash",
  "platform": "Emdash",
  "group": "A",
  "repo": "generalaction/emdash",
  "sha": "c8769892fd61f63b5c95ca1fccf727cee4d07809",
  "commit_date": "2026-11-23T15:30:26-08:00",
  "license": "Apache-2.0",
  "status": "Active",
  "default_state": "Opt-out (ON by default in packaged canary/prod builds when CI VITE_POSTHOG_KEY/HOST are injected; disabled in Vite dev builds; zero first-run onboarding notice)",
  "opt_out_controls": "Settings -> General -> Privacy & Telemetry switch (KV telemetry:enabled) and TELEMETRY_ENABLED=false/0/no env var (no CLI flag)",
  "do_not_track": false,
  "opt_out_bypasses": "Settings switch captures setting_changed BEFORE setTelemetryEnabled(false) RPC finishes (emits final event on opt-out); telemetry:instanceId, lastSessionId & 60s lastHeartbeatTs KV loop still run when disabled; pre-sampled perf-vitals timers not stopped mid-session (though outbound fetch drops at isEnabled()); UpdateService (GitHub/R2) has no disable toggle; gh CLI boot probe runs unconditionally",
  "persistent_ids": "SQLite KV telemetry:instanceId (UUIDv4, persisted even when disabled, exposed in Help -> Copy Installation ID), per-launch session_id + KV telemetry:lastSessionId, KV desktop-client-id (SSH workspace-server only), raw entity IDs (project_id, task_id, conversation_id, automation_id, terminal_id), and signed-in plaintext email + github_username + account_id",
  "hashing": "None (raw UUIDs; signed-in Emdash Account users send plaintext email, github_username, and account_id to PostHog $identify, /decide/?v=3, and getBaseProps(), stitching anonymous pre-login instanceId to real identity)",
  "external_auth_reads": "CRITICAL: bootBackground unconditionally runs `gh auth status --json hosts --show-token` on every launch to extract plaintext GitHub CLI OAuth tokens, query api.github.com/GHE, save to safeStorage, and emit integration_connected telemetry (token not sent to PostHog); also reads ~/.ssh/config (+Includes) for UI import, manages agent hooks/MCP/trust in ~/.claude/settings.json, ~/.claude.json, ~/.codex/config.toml, ~/.cursor, ~/.copilot, ~/.gemini/config, etc., and checks agent auth/usage via CLI/SDK",
  "telemetry_endpoints": [
    "https://<VITE_POSTHOG_HOST>/capture/",
    "https://<VITE_POSTHOG_HOST>/decide/?v=3",
    "https://github.com/generalaction/emdash/releases & https://api.github.com/repos/generalaction/emdash/releases/tags/v<ver> (electron-updater)",
    "https://releases.emdash.sh (electron-updater generic channel & workspace-server artifacts)",
    "https://auth.emdash.sh (Emdash Account PKCE OAuth & provider token broker)",
    "https://emdash-feedback-relay.real-general-action.workers.dev (opt-in feedback modal relay)",
    "https://emdash.com/api/waitlist (opt-in waitlist modal)"
  ],
  "ip_geo_handling": "No IP suppression ($ip: null and $geoip_disable: true absent on raw PostHog fetch() calls); additionally daily_active_user explicitly collects Intl.DateTimeFormat().resolvedOptions().timeZone",
  "custom_model_handling": "Not collected (only agent provider ID e.g. claude/codex sent; customModel/modelName absent; zero prompt text or code sent)",
  "docs_discrepancies": "UI calls telemetry 'anonymous usage data' while $identify/getBaseProps() attach plaintext email, github_username, and account_id + raw IP + timezone; conflicting doc URLs (README.md links emdash.com/docs/telemetry vs TelemetryCard.tsx links docs.emdash.sh/telemetry); no PRIVACY.md in repo; TELEMETRY_ENABLED=false still writes instanceId/lastSessionId/60s heartbeat to SQLite; install_source falls back to 'dmg' on packaged Windows/Linux builds",
  "privacy_rating": "D",
  "one_line_verdict": "Opt-out PostHog telemetry with no onboarding notice, missing IP suppression, plaintext PII identity stitching ($identify + base props) contradicting 'anonymous' UI copy, final setting_changed leak on opt-out, persistent ID/heartbeat writes when disabled, and unconditional startup `gh auth status --show-token` token harvesting."
}
```
