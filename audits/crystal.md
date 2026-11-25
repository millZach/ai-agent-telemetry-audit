# Forensic Privacy & Telemetry Audit: Crystal (`stravu/crystal`)

- **Slug**: `crystal`
- **Platform**: `Crystal`
- **Group**: `A`
- **Repository**: `https://github.com/stravu/crystal` (`stravu/crystal`)
- **Branch / Tag**: `main` (`0.3.5`)
- **Audited Commit SHA**: `1e18e0bc981225f75b5226f82a300fa741970c6f`
- **Commit Date**: `2026-02-26T16:44:09-05:00`
- **License**: `MIT` (`LICENSE:1-21`, `package.json:10`)
- **Status**: **Deprecated / Rebranded** — Succeeded by **Nimbalyst** (`https://github.com/Nimbalyst/nimbalyst`, `https://nimbalyst.com/`). Declared deprecated as of February 2026 in `README.md:13-16,52-56` and `CHANGELOG.md:4-8`, and enforced in the app via an unconditional startup modal (`showNimbalystMigrationPopup()` at `main/src/index.ts:103-269,851`) and worktree integration (`main/src/ipc/nimbalyst.ts:6-63`).
- **Privacy Rating**: **`D` (Aggressive / Leaky)**

---

## 1. Default State & Opt-Out Mechanism

### Default State
On paper, Crystal configures PostHog analytics as **opt-in (`enabled: false`)**, but in practice it **sends telemetry unconditionally on every launch even when disabled or unconsented**:
- `ConfigManager` initializes `config.analytics` to `{ enabled: false, posthogApiKey: 'phc_uwOqT2KUa4C9Qx5WbEPwQSN9mUCoSGFg1aY0b670ft5', posthogHost: 'https://us.i.posthog.com' }` (`main/src/services/configManager.ts:60-64,202-212`).
- At startup (`main/src/index.ts:716-717`), `AnalyticsManager` runs:
  1. Unconditional UUID creation and persistence in its constructor (`main/src/services/analyticsManager.ts:20-47`).
  2. Unconditional instantiation of a **second** PostHog client (`this.minimalClient`) with `flushAt: 1` and `flushInterval: 1000`, regardless of `settings.enabled` (`main/src/services/analyticsManager.ts:55-67`).
  3. Only `this.client` (full behavioral tracking) is skipped when `!settings.enabled` (`main/src/services/analyticsManager.ts:70-74`).
- Immediately after creating the main window (`main/src/index.ts:868-879`), if `configManager.isAnalyticsEnabled()` is `false` (which is true on first launch *before* the user answers the consent modal, and on every launch after declining/opting out), Crystal calls `analyticsManager.trackMinimalEvent('app_opened', { is_first_launch: isFirstLaunch })`, immediately dispatching an event to `https://us.i.posthog.com` with the persistent `distinctId`.
- Additionally, `versionChecker.checkOnStartup()` (`main/src/index.ts:893`, `main/src/services/versionChecker.ts:80-92`) unconditionally requests `https://api.github.com/repos/stravu/crystal/releases/latest` at startup without checking `config.autoCheckUpdates === false`.

### Opt-Out Controls & Bypasses
1. **First-Run Consent Dialog** (`frontend/src/App.tsx:73-99`, `frontend/src/components/AnalyticsConsentDialog.tsx:14-119`):
   - Triggered once when `user_preferences['analytics_consent_shown'] !== 'true'`.
   - Clicking **"No thanks"** (`AnalyticsConsentDialog.tsx:45-71`) calls `analyticsService.trackAnalyticsOptedOut()`, which routes through `main/src/ipc/analytics.ts:97-99` to `analyticsManager.trackMinimalEvent('analytics_opted_out', {})` — **transmitting an outbound PostHog event upon opting out**. It then sets `analytics.enabled = false`.
   - Clicking **"Enable analytics"** (`AnalyticsConsentDialog.tsx:18-43`) updates `config.analytics.enabled = true`, then calls `analyticsService.trackAnalyticsOptedIn()`. However, `AnalyticsManager` never subscribes to `configManager`'s `'config-updated'` event and never re-runs `initialize()`, so `this.isInitialized` remains `false` and `this.client` remains `null` (`main/src/services/analyticsManager.ts:17,70-73,96`) until the next application restart — meaning `analytics_opted_in` and all full-telemetry events in that first session are silently dropped.
2. **Settings Modal ("Analytics" Tab)** (`frontend/src/components/Settings.tsx:533-614,689-711`):
   - **Broken "Save Changes" button on the Analytics tab**: The Analytics tab renders `<form id="analytics-form" onSubmit={handleSubmit}>` (`Settings.tsx:534`) with no submit button inside it, while the modal footer's "Save Changes" `<Button>` (`Settings.tsx:699-708`, `frontend/src/components/ui/Button.tsx:44-56`) sets:
     - `type={activeTab === 'general' ? 'submit' : 'button'}`
     - `form={activeTab === 'general' ? 'settings-form' : undefined}`
     - `onClick={activeTab === 'notifications' ? (e) => handleSubmit(e as React.FormEvent) : undefined}`
     Consequently, clicking **Save Changes** while on `activeTab === 'analytics'` executes **nothing** (no form submission and no `onClick` handler), silently failing to persist opt-out/opt-in changes unless the user switches to the `general` or `notifications` tab before clicking Save.
   - **Shallow merge overwrites `distinctId` / `posthogApiKey` / `posthogHost`**: When `handleSubmit` is triggered from another tab, `Settings.tsx:146-148` sends `analytics: { enabled: analyticsEnabled }` to `configManager.updateConfig()` (`main/src/services/configManager.ts:124-126`), which performs a shallow spread (`this.config = { ...this.config, ...updates }`), wiping `posthogApiKey`, `posthogHost`, and `distinctId` in memory/`config.json` until restart (on restart, `configManager.initialize()` at `104-107` merges defaults back in and `AnalyticsManager` generates a new `distinctId`).
   - **UI state fallback bug**: `Settings.tsx:55,96-100` initializes `analyticsEnabled` React state to `true` and checks `data.analytics.enabled !== false`, defaulting UI state to enabled if `data.analytics` is absent.
3. **Hard Opt-Out / Unstoppable Channels**:
   - `minimalClient` (`app_opened`, `analytics_opted_out`, plus the open IPC handler `analytics:track-minimal-event` at `main/src/ipc/analytics.ts:115-123`) has **no UI toggle, CLI flag, or environment variable to disable it**. Only manually editing `~/.crystal/config.json` to set `analytics.posthogApiKey` to `""` (not deleting the key, because `configManager.ts:104-107` restores the hardcoded default if omitted) prevents `minimalClient` initialization.
   - `versionChecker.checkOnStartup()` (`main/src/services/versionChecker.ts:80-92`) ignores `autoCheckUpdates: false` (unlike `startPeriodicCheck()` and `performCheck()` at `versionChecker.ts:97,118`).
   - **Do Not Track (`DO_NOT_TRACK` / `DNT`)**: Not checked or honored anywhere in the repository (verified via negative `rg`).

---

## 2. External Auth / Local Non-Workspace File Reads

Crystal accesses several paths outside the active repository/workspace, **none of which are transmitted to PostHog or used for telemetry tracking**:
- **Claude Code MCP config (`~/.claude.json`)**: `ClaudeCodeManager.getBaseProjectMcpServers()` (`main/src/services/panels/claude/claudeCodeManager.ts:570-648,606-636`) reads `path.join(os.homedir(), '.claude.json')` when starting Claude sessions to extract `projects[baseProjectPath].mcpServers` and global `mcpServers` and merge them into a temporary MCP config (`~/.crystal/crystal-base-mcp-<sessionId>.json` or `crystal-mcp-<sessionId>.json` at `claudeCodeManager.ts:732,882-905`) so git worktrees inherit MCP servers configured on the base project. It does **not** read `~/.claude/.credentials.json` or OAuth keys.
- **OpenAI Codex session logs (`~/.codex/sessions/**/*.jsonl`)**: `CodexManager.findAndStoreCodexSessionId()` (`main/src/services/panels/codex/codexManager.ts:655-934,702-765`) recursively scans `path.join(os.homedir(), '.codex', 'sessions')` for `.jsonl` files and reads their first line (`payload.cwd`, `payload.id`) to correlate and persist the Codex UUID for conversation `--resume`. It does **not** read `~/.codex/auth.json` or OpenAI credentials from Codex's home dir.
- **Shell / PATH / Node discovery**:
  - `main/src/utils/shellPath.ts:132-162,248-311,359-390` executes login shells or falls back to reading `~/.zshrc`, `~/.bashrc`, `~/.bash_profile`, `~/.profile`, `~/.zprofile` for `export PATH=...` lines, and inspects `~/.nvm/versions/node`, `~/.yarn/bin`, `~/.local/bin`, etc.
  - `main/src/utils/shellDetector.ts:74-107` runs `dscl . -read /Users/<username> UserShell` (macOS) or reads `/etc/passwd` (Unix) to determine the user's default shell.
  - `main/src/utils/nodeFinder.ts:26-87` and `main/src/utils/claudeCodeTest.ts:13-84` probe standard Node/Claude binary locations in `os.homedir()`.
- **Crystal's own state (`~/.crystal/` or `~/.crystal_dev/`)**: `main/src/utils/crystalDirectory.ts:45-72` stores `config.json`, `sessions.db`, logs (`~/.crystal/logs/` via `main/src/utils/logger.ts:31`), and temporary MCP/artifact files (`~/.crystal/artifacts/`). Overridable via `--crystal-dir` or `CRYSTAL_DIR` (`main/src/index.ts:294-303`, `crystalDirectory.ts:47-55`).
- **Stravu auth store**: `StravuAuthManager` (`main/src/services/stravuAuthManager.ts:50-63,147-151`) constructs `path.join(app.getPath('userData'), 'stravu-auth.json')`, but MCP/OAuth storage is disabled (`// MCP functionality disabled - no credential storage`), `getStoreData()` returns `{}`, `setStoreData()` is a no-op, and `authenticate()`/`loadStoredCredentials()` do not read or persist credentials.
- **Nimbalyst app probe**: `main/src/ipc/nimbalyst.ts:6,10-18,24` checks `existsSync('/Applications/Nimbalyst.app/Contents/MacOS/Nimbalyst')` when the user clicks the Nimbalyst session button.

---

## 3. Persistent Device / Install IDs & Hashing

- **Persistent Install ID (`distinctId`)**:
  - Generated unconditionally in `AnalyticsManager` constructor (`main/src/services/analyticsManager.ts:23,30-47`) using `crypto.randomUUID()` prefixed as `crystal_${uuid}`.
  - Persisted in `~/.crystal/config.json` under `analytics.distinctId` via `configManager.setAnalyticsDistinctId()` (`main/src/services/configManager.ts:214-228`).
  - **Generated, persisted, and transmitted even when `analytics.enabled === false`** (via `minimalClient.capture({ distinctId: this.distinctId, ... })` at `main/src/services/analyticsManager.ts:152-156`).
  - Note: Saving settings from any tab where `Settings.tsx:146-148` submits `analytics: { enabled }` temporarily removes `distinctId` from `config.json` due to shallow merge (`configManager.ts:125`), causing a new `crystal_<uuid>` to be generated on next launch.
- **Session ID Hashing (`hashSessionId`)**:
  - `AnalyticsManager.hashSessionId()` (`main/src/services/analyticsManager.ts:234-236`) computes an **unsalted** SHA-256 hex digest truncated to 16 characters:
    `crypto.createHash('sha256').update(sessionId).digest('hex').substring(0, 16)`.
  - Used for `session_id_hash` on `terminal_panel_created` (`main/src/services/panelManager.ts:122`), `claude_panel_created` / `codex_panel_created` (`main/src/services/panels/ai/AbstractAIPanelManager.ts:267`), and `nimbalyst_button_clicked` / `nimbalyst_install_dialog_shown` / `nimbalyst_opened` (`frontend/src/components/SessionListItem.tsx:346-379`).
  - Because `sessionId` is Crystal's local random session UUID (not an email, username, or third-party account ID), there is **no unsalted cross-provider identity stitching**, though the hash itself lacks a per-install salt.
- **Local SQLite Launch Counter**:
  - `app_opens` table (`main/src/database/database.ts:720-742,2522-2565`) logs every startup timestamp (`opened_at`) and `app_version` locally in `~/.crystal/sessions.db` to power `is_first_launch` and `app_updated` detection (`main/src/index.ts:856-882`) plus Discord popup gating (`frontend/src/App.tsx:219-252`).
- **No Hardware / OS Machine Fingerprinting**: Negative `rg` confirmed zero usage of `os.hostname()`, MAC addresses, `machine-id`, IOPlatformUUID, or Electron `machineId`.

---

## 4. Exact Outbound Telemetry / Analytics / Crash / OTEL / Relay / Update Endpoints

| Category | Destination / Endpoint | Trigger & Consent Gate | Source Reference |
|---|---|---|---|
| **PostHog Analytics (Minimal Client)** | `https://us.i.posthog.com` (fallback `https://app.posthog.com`), project key `phc_uwOqT2KUa4C9Qx5WbEPwQSN9mUCoSGFg1aY0b670ft5` | **ALWAYS ON** (when `posthogApiKey` non-empty). Fires `app_opened` on startup when `enabled === false`, and `analytics_opted_out` when declining/opting out. | `main/src/services/configManager.ts:62-63`; `main/src/services/analyticsManager.ts:55-67,132-162`; `main/src/index.ts:874-879`; `main/src/ipc/analytics.ts:97-99,115-123` |
| **PostHog Analytics (Full Client)** | `https://us.i.posthog.com` (fallback `https://app.posthog.com`), same project key | Opt-in (`analytics.enabled === true`), requires app restart after enabling from first-run dialog. | `main/src/services/analyticsManager.ts:70-89,94-126`; `main/src/services/configManager.ts:60-64` |
| **GitHub Releases Version Check** | `https://api.github.com/repos/stravu/crystal/releases/latest` | **Unconditional at startup** (`checkOnStartup()` ignores `autoCheckUpdates === false`); periodic 24h checks & manual checks respect `autoCheckUpdates`. | `main/src/services/versionChecker.ts:40,80-133`; `main/src/index.ts:891-894`; `main/src/ipc/updater.ts:12-20` |
| **Electron Auto-Updater** | GitHub Releases (`owner: "stravu"`, `repo: "crystal"`) via `electron-updater` (or `process.env.UPDATE_SERVER_URL` / `http://localhost:8080` when `TEST_UPDATES=true`) | User-triggered only via `UpdateDialog` (`updater:check-and-download`, `updater:download-update`, `updater:install-update`); `autoDownload = false`. | `package.json:194-198`; `main/src/autoUpdater.ts:12-23`; `main/src/ipc/updater.ts:124-170`; `main/src/test-updater.ts:5-11`; `frontend/src/components/UpdateDialog.tsx:100-140` |
| **Stravu API (Disabled Integration)** | `https://api.stravu.com` (prod) / `http://localhost:9100` (dev) | Auth initiation is stubbed to throw `'Functionality coming soon'`; `pollForCompletion` (`GET /mcp/auth/status/:sessionId`), `disconnect` (`POST /mcp/auth/revoke`), and `StravuNotebookService` (`/mcp/v1/notebooks*`) exist but cannot connect in normal flow. | `main/src/services/stravuAuthManager.ts:7-9,65-120,166-205`; `main/src/services/stravuNotebookService.ts:56,94,125`; `main/src/services/stravuMcpService.ts:33-63` |
| **Anthropic SDK (Optional User-Key Feature)** | Anthropic API (`@anthropic-ai/sdk`) | Only initialized & called (`claude-3-haiku-20240307`) if user configures an `anthropicApiKey` in Settings for session naming. | `main/src/services/worktreeNameGenerator.ts:20-66` |
| **External Browser Links (`shell.openExternal` / `<a href>`)** | `https://nimbalyst.com/`, `https://nimbalyst.com/download/`, `https://discord.gg/XrVa6q7DPY`, `https://stravu.com/?utm_source=Crystal&utm_medium=OS&utm_campaign=Crystal&utm_id=1`, `https://github.com/stravu/crystal`, `https://docs.anthropic.com/en/docs/claude-code` | User click only (migration popup, Nimbalyst install dialog, Discord popup, About dialog, Settings Stravu tab). Note `stravu.com` links carry static UTM query params (`utm_source=Crystal`). | `main/src/index.ts:100-101,136-138,247-252`; `frontend/src/components/NimbalystInstallDialog.tsx:36`; `frontend/src/components/DiscordPopup.tsx:78`; `frontend/src/components/AboutDialog.tsx:152,305,320,328`; `frontend/src/components/Settings.tsx:658` |
| **Crash Reporting / OTEL / Sentry / Relay** | **None** | Verified absent via negative search. | — |

### IP & Geolocation Handling
- Neither `minimalClient` nor `client` sets PostHog's `disableGeoip: true`, `$ip: null`, or `$geoip_disable: true` (`main/src/services/analyticsManager.ts:58-62,77-81,114-118,152-156,173-180`). PostHog (`https://us.i.posthog.com`) therefore receives client IP addresses on every minimal and full event and performs default server-side GeoIP enrichment.

---

## 5. Exact Data & Properties Collected

### A. Minimal Telemetry (`minimalClient` -> `trackMinimalEvent`, active even when opted out / disabled)
Common properties attached by `trackMinimalEvent()` (`main/src/services/analyticsManager.ts:132-156`):
- `distinctId`: `crystal_<uuid>`
- `app_version`: `app.getVersion()` (e.g., `"0.3.5"`)
- `platform`: `os.platform()` (`darwin` | `linux` | `win32`)
- `analytics_enabled`: `boolean` (`false` when opted out / pre-consent)

| Event Name | Additional Properties | Source |
|---|---|---|
| `app_opened` (when disabled) | `is_first_launch` (`boolean`) | `main/src/index.ts:874-879` |
| `analytics_opted_out` | *(none)* | `frontend/src/components/AnalyticsConsentDialog.tsx:50`; `frontend/src/components/Settings.tsx:128-131`; `main/src/ipc/analytics.ts:97-99` |

*(Note: `analytics:track-minimal-event` at `main/src/ipc/analytics.ts:115-123` is also registered on `ipcMain` without an enabled check, though unused by the renderer.)*

### B. Full Telemetry (`client` -> `track`, requires `analytics.enabled === true` + app restart)
Common properties attached by `track()` (`main/src/services/analyticsManager.ts:94-118`):
- `distinctId`: `crystal_<uuid>`
- `app_version`: `app.getVersion()`
- `platform`: `os.platform()`
- `electron_version`: `process.versions.electron`

| Category | Event Name | Exact Properties | Source |
|---|---|---|---|
| **App Lifecycle** | `app_updated` | `previous_version`, `new_version` | `main/src/index.ts:862-865` |
| **App Lifecycle** | `app_opened` | `is_first_launch` | `main/src/index.ts:871-873` |
| **App Lifecycle** | `app_closed` | `session_duration_seconds` | `main/src/index.ts:996-998` |
| **Sessions** | `session_created` | `tool_type`, `template_type` (`main`\|`worktree`), `session_count`, `has_folder`, `used_claude_code`, `used_codex`, `used_auto_name`, `auto_name_available`, `git_mode`, `existing_active_sessions_count`, `existing_total_sessions_count`, `existing_archived_sessions_count`, `existing_sessions_initializing`, `existing_sessions_running`, `existing_sessions_waiting`, `existing_sessions_stopped`, `existing_sessions_error`, `existing_sessions_completed`, `existing_sessions_completed_unviewed`, `existing_projects_count` | `main/src/services/sessionManager.ts:415-436` |
| **Sessions** | `session_started` | `tool_type`, `is_continuation` | `main/src/services/sessionManager.ts:554-557` |
| **Sessions** | `session_completed` | `duration_seconds`, `duration_category`, `prompt_count` | `main/src/services/sessionManager.ts:629-633` |
| **Sessions** | `session_archived` | `session_age_days` | `main/src/services/sessionManager.ts:749-751` |
| **Sessions** | `session_stopped` | `duration_seconds`, `duration_category`, `had_errors` | `main/src/services/sessionManager.ts:820-824` |
| **Sessions** | `session_continued` | `time_since_last_message_hours`, `message_count` | `main/src/services/sessionManager.ts:1055-1058` |
| **AI Panels** | `claude_panel_created` / `codex_panel_created` | `session_id_hash` (16-char unsalted SHA-256), `panel_count` | `main/src/services/panels/ai/AbstractAIPanelManager.ts:252-270` |
| **AI Panels** | `ai_prompt_submitted` | `prompt_length_category` (`short`\|`medium`\|`long`\|`very_long`), `panel_type` | `main/src/services/panels/ai/AbstractAIPanelManager.ts:475-478` |
| **AI Panels** | `ai_conversation_turn` | `turn_number`, `panel_type` | `main/src/services/panels/ai/AbstractAIPanelManager.ts:494-497` |
| **AI Panels** | `ai_response_received` | `duration_seconds`, `had_tool_calls`, `panel_type` | `main/src/services/panels/ai/AbstractAIPanelManager.ts:517-521` |
| **Panels** | `terminal_panel_created` | `session_id_hash`, `terminal_count` | `main/src/services/panelManager.ts:121-124` |
| **Panels** | `panel_closed` | `panel_type`, `panel_lifetime_seconds` | `main/src/services/panelManager.ts:194-197` |
| **Panels** | `panel_switched` | `from_panel_type`, `to_panel_type` | `main/src/services/panelManager.ts:274-277` |
| **Panels** | `panel_renamed` | `panel_type` | `main/src/ipc/panels.ts:101-103` |
| **Projects** | `project_created` | `was_auto_initialized`, `project_count` | `main/src/ipc/project.ts:158-161` |
| **Projects** | `project_switched` | `has_active_sessions` | `main/src/ipc/project.ts:212-214` |
| **Projects** | `project_settings_updated` | `setting_category` (`system_prompt`\|`scripts`\|`commit`\|`ide`\|`name`\|`other`) | `main/src/ipc/project.ts:272-274` |
| **Projects** | `project_deleted` | `session_count`, `project_age_days` | `main/src/ipc/project.ts:346-349` |
| **Folders** | `folder_created` | `nesting_level` | `main/src/ipc/folders.ts:42-44` |
| **Folders** | `folder_renamed` | *(none)* | `main/src/ipc/folders.ts:59` |
| **Folders** | `folder_deleted` | `contained_session_count` | `main/src/ipc/folders.ts:95-97` |
| **Git & Scripts** | `git_diff_viewed` | `file_count_category`, `has_uncommitted` | `main/src/services/gitDiffManager.ts:335-338` |
| **Git & Scripts** | `git_worktree_created` | `branch_existed` | `main/src/services/worktreeManager.ts:165-167` |
| **Git & Scripts** | `git_worktree_cleaned` | `session_age_days` | `main/src/services/worktreeManager.ts:189-191` |
| **Git & Scripts** | `git_rebase_executed` | `success`, `duration_seconds`, `conflict_occurred` | `main/src/services/worktreeManager.ts:489-493,506-510` |
| **Git & Scripts** | `git_squash_executed` | `success`, `duration_seconds`, `commit_count_category` | `main/src/services/worktreeManager.ts:667-671,685-689` |
| **Git & Scripts** | `git_operation_failed` | `operation_type`, `error_category` | `main/src/services/worktreeManager.ts:514-517,691-694` |
| **Git & Scripts** | `run_script_executed` | `success`, `duration_seconds` (categorized string via `categorizeDuration`) | `main/src/services/panels/logPanel/logsManager.ts:436-439` |
| **UI & Nimbalyst** | `analytics_opted_in` | *(none; dropped on initial consent due to uninitialized `client`)* | `frontend/src/components/AnalyticsConsentDialog.tsx:30`; `frontend/src/components/Settings.tsx:122-125` |
| **UI & Nimbalyst** | `notification_shown` | `notification_type`, `trigger_event` | `frontend/src/hooks/useNotifications.ts:115-118` |
| **UI & Nimbalyst** | `nimbalyst_button_clicked` | `session_id_hash` | `frontend/src/components/SessionListItem.tsx:347-349` |
| **UI & Nimbalyst** | `nimbalyst_install_dialog_shown` | `session_id_hash` | `frontend/src/components/SessionListItem.tsx:360-362` |
| **UI & Nimbalyst** | `nimbalyst_download_clicked` | *(none)* | `frontend/src/components/NimbalystInstallDialog.tsx:34` |
| **UI & Nimbalyst** | `nimbalyst_opened` | `session_id_hash` | `frontend/src/components/SessionListItem.tsx:377-379` |

*(Note: `view_switched`, `help_dialog_opened`, `settings_opened`, `settings_saved`, `sidebar_toggled`, and `search_used` are defined in `frontend/src/services/analyticsService.ts:12-198` and `main/src/ipc/analytics.ts:59-66`, but have zero call sites in `frontend/src/`. `AnalyticsManager.identify()` at `main/src/services/analyticsManager.ts:167-184` also has zero callers. No prompt text, AI responses, code, file paths, project names, or model names are transmitted to PostHog.)*

---

## 6. Docs / Privacy Policy vs Code Discrepancies

1. **False Zero-Collection Claim in Settings UI (`frontend/src/components/Settings.tsx:573,597`)**:
   - The Settings Analytics tab tells users *"When disabled, no analytics data will be collected or sent"* (`Settings.tsx:573`) and *"Analytics is disabled. No data will be collected or sent"* (`Settings.tsx:597`).
   - **Reality**: `minimalClient` is unconditionally initialized (`main/src/services/analyticsManager.ts:55-67`) and transmits `app_opened` on every startup when disabled (`main/src/index.ts:874-879`) as well as `analytics_opted_out` when declining (`AnalyticsConsentDialog.tsx:50`, `main/src/ipc/analytics.ts:97-99`), accompanied by a persistent `crystal_<uuid>` `distinctId` and client IP to PostHog.
2. **Missing Privacy Policy / Telemetry Disclosure in `README.md`**:
   - `README.md:1-56` contains only the February 2026 deprecation/Nimbalyst migration notice and has no privacy policy or disclosure of PostHog telemetry or hardcoded PostHog keys.
3. **Internal Docs Omit `minimalClient` Bypass & Misstate Configuration (`docs/ANALYTICS_UI_EVENTS_INTEGRATION.md`, `docs/FEATURE_USAGE_TRACKING_INTEGRATION.md`)**:
   - `docs/ANALYTICS_UI_EVENTS_INTEGRATION.md:304` and `docs/FEATURE_USAGE_TRACKING_INTEGRATION.md:326-327` state *"The tracking is opt-in (users can disable analytics in settings) and respects user privacy by design"* without mentioning `minimalClient` or persistent ID creation prior to consent.
   - `docs/FEATURE_USAGE_TRACKING_INTEGRATION.md:314` instructs developers to *"Configure PostHog API key in settings"*, whereas the PostHog key and host are hardcoded in `main/src/services/configManager.ts:62-63` and not exposed in `Settings.tsx`.
   - `docs/ANALYTICS_UI_EVENTS_INTEGRATION.md:35-209` documents 6 UI events (`view_switched`, `help_dialog_opened`, `settings_opened`, `settings_saved`, `sidebar_toggled`, `search_used`) that are wired into IPC/helpers but never invoked anywhere in the frontend.
4. **Update Check Frequency vs Startup Check (`frontend/src/components/Settings.tsx:408`)**:
   - Settings states *"We check GitHub for new releases every 24 hours"*, while `main/src/index.ts:893` and `main/src/services/versionChecker.ts:80-92` also execute an unconditional startup check against `api.github.com` even when `autoCheckUpdates` is unchecked (`false`).
5. **Broken Analytics Save Button (`frontend/src/components/Settings.tsx:534,699-708`)**:
   - The UI presents an interactive checkbox (`Enable anonymous analytics tracking`) and a `Save Changes` button on the Analytics tab, but clicking `Save Changes` on that tab is a dead no-op because `type`, `form`, and `onClick` only wire up the `general` and `notifications` tabs.

---

## 7. Zero-Telemetry / Negative-Check Verification

All non-PostHog telemetry/crash/OTEL channels and fingerprinting vectors were audited via word-bounded `rg` across the entire repository (excluding lockfiles and `NOTICES`):
- **Third-party telemetry / crash / OTEL SDKs**: Zero occurrences of `sentry`, `opentelemetry`, `otel`, `mixpanel`, `amplitude`, `segment`, `datadog`, `bugsnag`, `rollbar`, `statsig`, `launchdarkly`, `growthbook`, `logrocket`, `heap`, `fullstory`, or Electron `crashReporter`.
- **Do Not Track / privacy env vars**: Zero occurrences of `DO_NOT_TRACK`, `doNotTrack`, or `DNT`.
- **Hardware / machine identifiers**: Zero occurrences of `machineId`, `getMachineId`, `macaddress`, or `os.hostname()`. (Only `url.hostname` in `frontend/src/components/panels/ai/components/ToolCallView.tsx:231` matched `hostname`.)
- **Prompt / code / custom-model exfiltration**: Verified that `analyticsManager.track()` never transmits prompt contents, AI responses, code diffs, file paths, project names, or model strings. (Custom models or provider strings configured for Claude/Codex are kept strictly local and passed only to the user's local `claude` / `codex` CLI processes.)
- **Pre-consent / opt-out cleanliness check**: **Failed** due to `AnalyticsManager` generating/persisting `crystal_<uuid>` at construction, instantiating `minimalClient` unconditionally, and sending `app_opened` / `analytics_opted_out` to `https://us.i.posthog.com` with GeoIP enabled even when `analytics.enabled === false`.

---

```json
{
  "slug": "crystal",
  "platform": "Crystal",
  "group": "A",
  "repo": "stravu/crystal",
  "sha": "1e18e0bc981225f75b5226f82a300fa741970c6f",
  "commit_date": "2026-02-26T16:44:09-05:00",
  "license": "MIT",
  "status": "Deprecated / Rebranded (succeeded by nimbalyst/nimbalyst)",
  "default_state": "Mixed: full behavioral analytics defaults to false (opt-in), but minimal PostHog client (app_opened) and startup GitHub release check run unconditionally before consent and when disabled",
  "opt_out_controls": "Consent modal ('No thanks') and Settings -> Analytics checkbox (however, the Settings footer 'Save Changes' button is a broken no-op on the Analytics tab; no CLI flag or env var disables minimalClient)",
  "do_not_track": false,
  "opt_out_bypasses": "AnalyticsManager unconditionally creates minimalClient and sends 'app_opened' on every launch when enabled===false plus 'analytics_opted_out' on decline; VersionChecker.checkOnStartup() ignores autoCheckUpdates===false",
  "persistent_ids": "crystal_<crypto.randomUUID()> stored in ~/.crystal/config.json (analytics.distinctId), generated at startup and transmitted even when analytics is disabled",
  "hashing": "Unsalted 16-char truncated SHA-256 of local session UUIDs (session_id_hash); no cross-provider user identity hashing",
  "external_auth_reads": "None for telemetry; reads ~/.claude.json (MCP server config) and ~/.codex/sessions/*.jsonl (Codex UUID resume) strictly for local CLI orchestration",
  "telemetry_endpoints": [
    "https://us.i.posthog.com (PostHog SDK minimal + full clients, fallback https://app.posthog.com)",
    "https://api.github.com/repos/stravu/crystal/releases/latest (startup & periodic update checks)"
  ],
  "ip_geo_handling": "No IP suppression or disableGeoip option configured on PostHog clients; PostHog ingests client IP and performs default GeoIP enrichment",
  "custom_model_handling": "Not collected or transmitted to telemetry endpoints; model selections remain local to CLI arguments and SQLite panel settings",
  "docs_discrepancies": "Settings UI claims 'When disabled, no analytics data will be collected or sent' while minimalClient still sends app_opened/analytics_opted_out with distinctId; README lacks any privacy/telemetry disclosure; integration docs omit minimalClient bypass and falsely state PostHog API key is configured in Settings; Analytics tab Save Changes button is non-functional",
  "privacy_rating": "D",
  "one_line_verdict": "Deprecated in favor of Nimbalyst; gates full PostHog analytics behind opt-in, but leaks persistent UUIDs, IP/GeoIP, and app_opened/opt-out pings via an unstoppable minimalClient bypass, breaks the Settings Analytics Save button, and misstates zero collection when disabled."
}
```
