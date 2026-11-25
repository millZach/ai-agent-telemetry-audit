# Forensic Telemetry & Privacy Audit: Opcode (formerly Claudia) (`winfunc/opcode`)

- **Slug:** `opcode`
- **Platform:** Opcode (Claudia) (`winfunc/opcode`, to which `getAsterisk/claudia` redirects)
- **Group:** `A`
- **Repository:** `https://github.com/winfunc/opcode`
- **Branch / Commit SHA:** `main` @ `c24004d3bbc910bd30b49d402daec254dac26ab2`
- **Commit Date:** `2026-11-06T19:53:42+05:30`
- **Commit Subject:** `Merge pull request #488 from shahuldxb666-crypto/fix/462-tauri-events-esm`
- **License:** `AGPL-3.0` (`package.json:5`, `src-tauri/Cargo.toml:6`, `LICENSE:1-4`)
- **Status:** `Active (getAsterisk/claudia redirects here)`
- **Privacy Rating:** **`F`** (**Critical / Non-Compliant** — default-ON PostHog analytics with silent pre-set consent (`enabled: true, hasConsented: true`), unmounted dead consent UI, missing IP/GeoIP suppression, zero `DO_NOT_TRACK`/env/CLI opt-out, unguarded `setScreen`/`PostHogProvider`/`deleteAllData` edges, persistent `userId` and background timers retained when disabled, sanitizer leaks for `agent_type`/`feature_name`/`tools_used[]`/UUIDs/Windows paths, and a directly false `"No Telemetry"` / `"Local Storage"` security claim in `README.md:383-384`)

---

## 1. Default State & Opt-Out Mechanism

### Default State
- **Opt-out (ON by default on first launch across both desktop and web-server modes):**
  - `src/main.tsx:12-16` unconditionally invokes `analytics.initialize()` and `resourceMonitor.startMonitoring(120000)` before React mounts. Because both the Tauri desktop app (`src-tauri/src/main.rs`) and headless/LAN `opcode-web` server (`src-tauri/src/web_main.rs:22-37`, `src-tauri/src/web_server.rs:110-112`) serve the same React bundle, PostHog runs in both modes.
  - `ConsentManager.initialize()` (`src/lib/analytics/consent.ts:18-55`) checks `localStorage.getItem('opcode-analytics-settings')` (`consent.ts:3, 21`). When absent on fresh install—or if parsing throws (`consent.ts:47-54`, failing open)—it silently sets:
    ```ts
    this.settings = {
      enabled: true,
      hasConsented: true,
    };
    ```
    (`consent.ts:26-29`), generates a persistent UUID v4 `userId` (`consent.ts:33-36, 124-131`), saves it to `localStorage`, and assigns an in-memory `sessionId` (`consent.ts:39-41, 133-136`).
  - `AnalyticsService.initialize()` (`src/lib/analytics/index.ts:25-65`) hardcodes PostHog credentials (`apiKey: 'phc_6seRe1SJkFckJU2qQWeeIy62kaSoaUbCsdVCm1TQZg8'`, `apiHost: 'https://us.i.posthog.com'`, `persistence: 'localStorage'`, `autocapture: false`, `disable_session_recording: true`, `opt_out_capturing_by_default: false` at `index.ts:29-36`). Because `settings.hasConsented && settings.enabled` is `true` by default (`index.ts:54-56`), `initializePostHog()` immediately runs `posthog.init(...)`, calls `ph.identify(settings.userId, ...)`, captures `$screen` (`$screen_name: 'app_start'`), and calls `ph.opt_in_capturing()` (`index.ts:69-96`), followed by `app_started` in `useAppLifecycle()` (`src/App.tsx:67`, `src/hooks/useAnalytics.ts:370-383`).
- **Zero First-Run Telemetry Disclosure (`AnalyticsConsent` is dead code):**
  - `src/components/AnalyticsConsent.tsx:16-144, 150-235` defines `AnalyticsConsent` and `AnalyticsConsentBanner`, but neither component is imported or mounted anywhere in the app (`rg -n "AnalyticsConsent" src` matches only `AnalyticsConsent.tsx`). Even if mounted, both only render when `!settings?.hasConsented` (`AnalyticsConsent.tsx:33, 163`), which is permanently `false` under `ConsentManager`'s `hasConsented: true` default.
  - On first launch, `App` (`src/App.tsx:533-534`) renders `StartupIntro` (`src/components/StartupIntro.tsx:10-95`), a purely visual 2-second splash animation with no telemetry disclosure or opt-out choice.

### Opt-Out Controls
1. **In-App Settings UI Toggle (`Settings -> General -> Enable Analytics`):**
   - `src/components/Settings.tsx:673-698` binds a switch to `analytics.enable()` / `analytics.disable()`.
   - Turning the switch OFF awaits `analytics.disable()` (`Settings.tsx:691`) *before* calling `trackEvent.settingsChanged('analytics_enabled', false)` (`Settings.tsx:693`). `analytics.disable()` sets `this.settings.enabled = false` in `localStorage['opcode-analytics-settings']` and calls `posthog.opt_out_capturing()` (`src/lib/analytics/index.ts:115-120`, `src/lib/analytics/consent.ts:69-77`), so the trailing `settings_changed` call is dropped by `AnalyticsService.track()` (`index.ts:142-144`).
   - On subsequent cold starts with `enabled: false`, `AnalyticsService.initialize()` skips `initializePostHog()` (`index.ts:54-56`).
2. **Environment Variables & CLI Flags:** **None.** Neither Vite/React nor Rust exposes any telemetry disable env var or CLI flag (`--disable-telemetry`). Note `Settings.tsx:964` lists `CLAUDE_CODE_ENABLE_TELEMETRY` solely as informational help text for Claude Code CLI subprocess environment variables, not Opcode's own PostHog telemetry.
3. **Do Not Track / Global Privacy Control:** **Not supported.** Zero references to `navigator.doNotTrack`, `globalPrivacyControl`, or `DO_NOT_TRACK` exist anywhere in `src/` or `src-tauri/`.

### Opt-Out Bypasses, Fail-Open Edges & Persistent State When Disabled
1. **Unconditional `<PostHogProvider>` wrapper (`src/main.tsx:47-55`):**
   - `ReactDOM.createRoot(...).render(...)` unconditionally mounts `<PostHogProvider apiKey={import.meta.env.VITE_PUBLIC_POSTHOG_KEY} options={{ api_host: import.meta.env.VITE_PUBLIC_POSTHOG_HOST, defaults: '2025-05-24', capture_exceptions: true, debug: import.meta.env.MODE === "development" }}>` without checking `ConsentManager`.
   - In upstream `.github/workflows/*.yml`, `VITE_PUBLIC_POSTHOG_KEY` / `VITE_PUBLIC_POSTHOG_HOST` are not passed (`apiKey` evaluates to `undefined`), so CI release builds rely solely on the hardcoded `AnalyticsService` singleton. However, any build environment that defines `VITE_PUBLIC_POSTHOG_KEY` activates a second PostHog client with `capture_exceptions: true` that bypasses the Settings opt-out completely.
2. **Unguarded `setScreen()` (`src/lib/analytics/index.ts:129-138`):**
   - Called on every tab switch via `useScreenTracking()` (`src/components/TabContent.tsx:39`, `src/hooks/useAnalytics.ts:419-430`), `setScreen()` invokes `posthog.capture('$screen', { $screen_name: screenName })` **without checking `this.consentManager.isEnabled()`**. When opted out from cold start in standard builds, `initializePostHog()` is skipped so `posthog` is uninitialized; after toggling OFF mid-session (or if `PostHogProvider` initializes `posthog` via build env vars), `setScreen()` relies solely on `posthog.opt_out_capturing()`.
3. **`deleteAllData()` silently re-enables telemetry (`src/lib/analytics/consent.ts:79-92`; `src/lib/analytics/index.ts:122-127`):**
   - Although currently uncalled in the UI (`rg deleteAllData src` hits only `consent.ts:79` and `index.ts:122`), `ConsentManager.deleteAllData()` removes `opcode-analytics-settings` and immediately resets `{ enabled: true, hasConsented: true, userId: this.generateAnonymousId(), sessionId: this.generateSessionId() }`, silently re-enabling tracking without calling `posthog.opt_out_capturing()`.
4. **`shutdown()` omits `isEnabled()` check (`src/lib/analytics/index.ts:252-260`):**
   - Invoked on `beforeunload` (`src/hooks/useAnalytics.ts:376-379`), `shutdown()` calls `this.flushEvents()` without checking `isEnabled()` (harmless only while `eventQueue` remains empty because `track()` checks `isEnabled()`).
5. **Persistent ID, `sessionId`, and periodic timers remain active when disabled:**
   - Even when `enabled: false`, `ConsentManager.initialize()` preserves the stored `userId` in `localStorage['opcode-analytics-settings']` and generates a new `sessionId` in memory on every launch (`consent.ts:33-41`), `AnalyticsService.initialize()` starts the 5-second `setInterval` flush loop (`index.ts:59, 243-250`), and `src/main.tsx:16` starts `resourceMonitor.startMonitoring(120000)` (sampling heap/resource timing every 2 minutes, dropped at `analytics.track()`).
6. **Unused telemetry modules present in tree:**
   - `src/lib/api-tracker.ts:1-115` (wraps every `api.*` method to emit `network_performance`, `api_error`, `performance_bottleneck`, and `memory_warning`), `src/hooks/usePerformanceMonitor.ts:17-124`, and `PerformanceTracker` (`src/lib/analytics/index.ts:285-351`) are defined in source but have **zero imports or callers** across `src/` (`rg -n "api-tracker|PerformanceTracker|usePerformanceMonitor" src`). Similarly, `useWorkflowTracking` in `ClaudeCodeSession.tsx:141, 628` only calls `trackStep()` and never calls `startWorkflow()` or `completeWorkflow()` (`useAnalytics.ts:517-570`).

---

## 2. External Auth / Local Non-Workspace File Reads

- **No third-party credential or non-Claude auth harvesting:** Negative checks across `src/` and `src-tauri/src/` confirm zero reads of `~/.ssh`, `~/.aws`, `~/.codex`, `~/.cursor`, `~/.vscode`, OS Keychains, or `gh auth` commands.
- **Claude Code files (core application scope):** As a desktop GUI and session browser for Claude Code, the Rust backend reads/writes `~/.claude/` and invokes the `claude` CLI:
  - `src-tauri/src/main.rs:127-140` resolves `~/.claude` at startup for `CheckpointState`.
  - `src-tauri/src/claude_binary.rs:35-95, 355-475` scans standard install paths (`PATH`, `/usr/local/bin`, `/opt/homebrew/bin`, `~/.claude/local`, `~/.local/bin`, `~/.nvm/versions/node/*/bin`, `%USERPROFILE%\.claude\local\claude.exe`, `%USERPROFILE%\AppData\Roaming\npm\claude.cmd`, etc.) and `app_data_dir()/agents.db` (`claude_binary.rs:39-40`).
  - `src-tauri/src/commands/claude.rs:137-144, 622-700, 862-884, 928-942, 2385-2470`, `src-tauri/src/commands/usage.rs:291-310, 621-656`, `src-tauri/src/commands/slash_commands.rs:258-307, 335-368`, and `src-tauri/src/commands/agents.rs:170-191, 1461-1485, 1931-1994` read/write `~/.claude/projects/*.jsonl`, `~/.claude/settings.json`, `~/.claude/CLAUDE.md`, and `~/.claude/commands/` to power project/session browsing, usage dashboards, hooks/slash commands, and agent execution.
  - `src-tauri/src/commands/mcp.rs:213-227` shells out to `claude mcp ...`.
  - `src-tauri/src/commands/agents.rs:216-284` initializes SQLite `agents.db` under `app.path().app_data_dir()`. None of these files are transmitted to telemetry.
- **Claude Desktop MCP import (explicit user click only):** `mcp_add_from_claude_desktop` (`src-tauri/src/commands/mcp.rs:462-525`) reads `~/Library/Application Support/Claude/claude_desktop_config.json` (macOS) or `dirs::config_dir()/Claude/claude_desktop_config.json` (Linux) **only** when triggered via the "Import from Claude Desktop" button (`src/components/MCPImportExport.tsx:34-38`).
- **Tauri FS capability scope:** `src-tauri/tauri.conf.json:37-52` and `src-tauri/capabilities/default.json:34-45` grant `$HOME/**` (`fs:scope-home-recursive`) to the WebView context, though no startup or telemetry code reads outside `.claude` or user-selected folders (`FilePicker`).

---

## 3. Persistent Device / Install IDs & Hashing

- **Persistent `userId` (`localStorage['opcode-analytics-settings']`):**
  - Random UUID v4 generated with `Math.random()` (`src/lib/analytics/consent.ts:3, 33-36, 124-131`), persisted in browser/WebView `localStorage`, and passed to PostHog as `bootstrap.distinctID` and `ph.identify(settings.userId, ...)` (`src/lib/analytics/index.ts:73-87`). PostHog also stores its own persistent state via `persistence: 'localStorage'` (`index.ts:32, 76`).
  - Remains persisted across opt-outs (`consent.ts:69-77`).
- **Per-launch `sessionId`:**
  - Generated on each `ConsentManager.initialize()` call as `${Date.now()}-${Math.random().toString(36).substr(2, 9)}` (`consent.ts:39-41, 133-136`) and attached to events as `$session_id` (`index.ts:161, 235`).
- **Hardware / Machine IDs & Hashing:** **None.** No OS machine-ID, hostname, MAC address, or user-account email/username is collected or hashed for telemetry (`sha2` in `src-tauri/Cargo.toml:55` has zero usages in `src-tauri/src`; `.github/workflows/release.yml:83-87` only generates `.sha256` artifact checksums).
- **Session / Checkpoint UUIDs leaked in payloads:** `ClaudeCodeSession.tsx:857` (`session_resumed`) and `TimelineNavigator.tsx:180-183` (`checkpoint_restored`) pass raw hyphenated UUIDs in `checkpoint_id` (see Section 5).

---

## 4. Exact Outbound Telemetry / Analytics / Crash / OTEL / Relay / Update Endpoints

1. **`https://us.i.posthog.com` (Active Default-ON PostHog Ingestion):**
   - Hardcoded in `src/lib/analytics/index.ts:30-31` with project API key `phc_6seRe1SJkFckJU2qQWeeIy62kaSoaUbCsdVCm1TQZg8`.
   - Permitted in Tauri's CSP (`src-tauri/tauri.conf.json:27`: `script-src ... https://app.posthog.com https://*.posthog.com https://*.i.posthog.com https://*.assets.i.posthog.com; connect-src ... https://app.posthog.com https://*.posthog.com https://*.i.posthog.com`).
   - Invoked directly from the frontend in both Tauri desktop mode and `opcode-web` browser mode.
2. **`VITE_PUBLIC_POSTHOG_HOST` / `VITE_PUBLIC_POSTHOG_KEY` (`PostHogProvider`, build-time optional):**
   - `src/main.tsx:47-55` configures `<PostHogProvider apiKey={import.meta.env.VITE_PUBLIC_POSTHOG_KEY} options={{ api_host: import.meta.env.VITE_PUBLIC_POSTHOG_HOST, defaults: '2025-05-24', capture_exceptions: true, ... }}>`.
   - Unset in upstream `.github/workflows/*.yml` (`release.yml`, `build-linux.yml`, `build-macos.yml`, `build-test.yml`, `pr-check.yml`) and absent from the repository (zero `.env*` files); activates only if supplied in a custom build environment.
3. **Auto-Updater (Unused / Inert):**
   - `src-tauri/Cargo.toml:32` declares `tauri-plugin-updater = "2"` and `src-tauri/capabilities/default.json:52` includes `"updater:default"`, **but** `src-tauri/src/main.rs:58-60` does not register the plugin, `src-tauri/tauri.conf.json` configures no updater endpoints or public keys, and `src/` makes zero updater calls.
4. **Non-Telemetry User-Initiated / Local Endpoints:**
   - **GitHub Agents Browser (user-triggered):** Opening `GitHubAgentBrowser` (`src/components/GitHubAgentBrowser.tsx:49-90`) calls `fetch_github_agents` -> `GET https://api.github.com/repos/getAsterisk/opcode/contents/cc_agents` (`src-tauri/src/commands/agents.rs:1823-1866`, `User-Agent: opcode-App`) and `fetch_github_agent_content(download_url)` (`agents.rs:1870-1907`).
   - **External Browser Links (user click only):** `GitHubAgentBrowser.tsx:155`, `NFOCredits.tsx:159, 200`, `ClaudeBinaryDialog.tsx:117`, `Topbar.tsx:143`.
   - **Local `opcode-web` Server (opt-in CLI mode):** Binds `0.0.0.0:8080` (configurable via `--port`) in `src-tauri/src/web_server.rs:832-833`; frontend `src/lib/apiAdapter.ts:90-128, 296` only makes same-origin REST/WebSocket requests.
   - **Claude Code Subprocess & Optional User Proxy:** `proxy.rs:101-162` and `claude_binary.rs:641-659` propagate user-configured `HTTP_PROXY`/`HTTPS_PROXY`/`ALL_PROXY`/`NO_PROXY` env vars.

---

## 5. Exact Data & Properties Collected

### Identity, Context & IP/GeoIP Handling
- **`identify()` traits (`src/lib/analytics/index.ts:82-87`):** `distinctID` (UUID v4), `anonymous: true`, `consent_date`, `app_type: 'desktop'`, `app_name: 'opcode'`.
- **Automatic envelope on `track()` (`index.ts:150-163, 233-238`):** `screen_name`, `app_context: 'opcode_desktop'`, `$session_id`, `timestamp`, `$current_url: "opcode://${event.properties?.screen_name || 'unknown'}"`.
- **IP & GeoIP Suppression:** **Absent.** Neither `posthog.init()` (`index.ts:69-101`) nor `<PostHogProvider>` (`main.tsx:47-55`) sets `ip: false`, `$ip: null`, or `$geoip_disable: true`, exposing client IP addresses directly to `https://us.i.posthog.com` for standard PostHog GeoIP processing.

### Live Emitted Events & Properties
- **App Lifecycle & Navigation:**
  - `app_started`, `app_closed` (`src/App.tsx:67`, `src/hooks/useAnalytics.ts:370-383`).
  - `$screen`: `$screen_name: 'app_start'` on init (`index.ts:90-92`) and `${screenName}/${tabId.substring(0, 8)}` on active tab changes (`src/components/TabContent.tsx:39`, `src/hooks/useAnalytics.ts:419-430`).
  - `tab_created` (`tab_type`), `tab_closed` (`tab_type`), `feature_used` (`feature: 'tab_reorder'`, `subfeature: 'drag_drop'`, `from_index`, `to_index`) (`src/components/TabManager.tsx:175, 182, 266-269, 276, 284`).
  - `journey_milestone` (`category: 'user_journey'`, `journey_stage: 'onboarding'`, `milestone_reached: 'projects_created'`, `time_to_milestone_ms`) (`src/App.tsx:78-82`).
- **Claude Code Sessions & Model Handling (`src/components/ClaudeCodeSession.tsx`):**
  - `session_created` (`model`, `source: 'prompt_input'`, `:863`) and `model_selected` (`new_model`, `:858, 864`).
  - `session_resumed` (`resumed: true`, `checkpoint_id: effectiveSession.id`, `:857`) and `session_completed` (`:1165`).
  - `prompt_submitted` (`prompt_length`, `model`, `has_attachments`, `source: 'keyboard'`, `word_count`, `conversation_depth`, `prompt_complexity`, `contains_code`, `language_detected` [markdown code fence language tag], `session_age_ms`, `:841-852`). Does **not** transmit raw prompt text.
  - `session_stopped` (`duration_ms`, `messages_count`, `reason`, `time_to_first_message_ms`, `average_response_time_ms`, `idle_time_ms`, `prompts_sent`, `tools_executed`, `tools_failed`, `files_created`, `files_modified`, `files_deleted`, `total_tokens_used`, `code_blocks_generated`, `errors_encountered`, `model`, `has_checkpoints`, `checkpoint_count`, `was_resumed`, `agent_success`, `stop_source`, `final_state`, `has_pending_prompts`, `pending_prompts_count`, `:701-743, 989-1020`).
  - `session_engagement` (`session_duration_ms`, `messages_sent`, `tools_used: Array.from(toolsUsed)`, `files_modified`, `engagement_score`, `:1185-1191`).
  - `error_occurred` (`error_type: 'tool_execution'`, `error_code: 'tool_failed'`, `error_message: sanitizers.sanitizeErrorMessage(result.content)`, `context`, `user_action_before_error`, `recovery_attempted`, `recovery_successful`, `error_frequency`, `:642-652`).
  - **Custom model handling:** Chat UI pickers (`FloatingPromptInput.tsx:166-191`, `CreateAgent.tsx:242-289`, `AgentExecution.tsx:603-645`) restrict selection to `"sonnet" | "opus"`, sent verbatim as `model`/`new_model`. While imported `.opcode.json` agents (`src-tauri/src/commands/agents.rs:1702-1745`) can store an arbitrary `model` string loaded into `AgentExecution.tsx:93`, `AgentExecution` passes that value only to local `api.executeAgent` (`AgentExecution.tsx:307`) and does not emit `modelSelected`.
- **Checkpoints (`src/components/TimelineNavigator.tsx`):**
  - `checkpoint_created` (`checkpoint_number`, `session_duration_at_checkpoint`, `:134-137`); `checkpoint_restored` (`checkpoint_id: checkpoint.id`, `time_since_checkpoint_ms`, `:180-183`).
- **Custom Agents (`src/components/AgentExecution.tsx`):**
  - `agent_started` (`agent_type: agent.name || 'custom'`, `agent_name: sanitizeAgentName(agent.name)`, `has_custom_prompt`, `:312-316`).
  - `agent_executed` (`agent_type: agent.name || 'custom'`, `success`, `agent_name: sanitizeAgentName(agent.name)` via `sanitizeProperties`, `duration_ms`, `:359, 371`).
  - `agent_error` (`error_type`, `error_stage`, `retry_count`, `agent_type: agent.name || 'custom'`, `:340-345, 360-365`).
  - `feature_discovered` (`feature_name: "agent_${agent.name || 'custom'}"`, `discovery_method`, `time_to_first_use_ms`, `initial_success`) and `feature_adopted` (`feature`, `adoption_stage`, `usage_count`, `days_since_first_use`, `usage_trend`) via `useFeatureAdoptionTracking` (`AgentExecution.tsx:106, 319`, `useAnalytics.ts:475-507`).
- **MCP & Slash Commands:**
  - `mcp_server_added` (`server_type`, `configuration_method: 'manual'`, `src/components/MCPAddServer.tsx:139-142, 199-202`).
  - `mcp_server_removed` (`server_name: sanitizeServerName(name)`, `was_connected`), `mcp_server_connected` (`server_name`, `success`, `server_type`), `mcp_connection_error` (`server_name: sanitizeServerName(name)`, `error_type`, `retry_attempt`) (`src/components/MCPServerList.tsx:111-114, 134, 141-145`).
  - `slash_command_selected` (`command_name: sanitizeCommandName(name)`, `selection_method`, plus `feature_discovered`/`feature_adopted` for `'slash_commands'`) (`src/components/SlashCommandPicker.tsx:94, 179-184, 232-237`); `slash_command_created` (`command_type`, `has_parameters`) (`src/components/SlashCommandsManager.tsx:184-187`).
- **Settings, Component Lifetimes, Resources & UI Errors:**
  - `settings_changed` (`analytics_enabled`, `tab_persistence_enabled`, `startup_intro_enabled`, `Settings.tsx:688, 693, 731, 757`).
  - `component_metrics` (`component`, `lifetime_ms`, `render_count` for `ClaudeCodeSession` and `AgentExecution`, `useAnalytics.ts:387-406`).
  - `resource_usage_sampled` (every 10th 2-minute tick = ~20 min) & `resource_usage_high` (`memory_usage_mb`, `network_requests_count`, `active_connections` in `src/lib/analytics/resourceMonitor.ts:67-112`, `main.tsx:16`).
  - `ui_error` (`component_name: errorInfo.componentStack?.split('\n')[0] || 'Unknown'`, `error_type: error.name`, `src/components/AnalyticsErrorBoundary.tsx:29-35`).

### Sanitization Gaps (`src/lib/analytics/index.ts:190-223`, `src/lib/analytics/events.ts:648-700`)
1. **Array properties bypass sanitization:** `sanitizeProperties()` (`index.ts:208-218`) only regex-cleans `typeof value === 'string'`; `ClaudeCodeSession.tsx:1188` sends `tools_used: Array.from(toolsUsed)`, passing raw MCP server/tool names (e.g., `mcp__<server>__<tool>`) unredacted.
2. **Custom `agent.name` leaks past `sanitizeAgentName`:** `sanitizeProperties()` (`index.ts:204`) only sanitizes keys containing `'agent_name'`. Raw `agent.name` passes verbatim in `agent_type` (`AgentExecution.tsx:313, 344, 359, 364, 371`), `feature_name` (`AgentExecution.tsx:106` -> `"agent_${agent.name}"`), and `feature` (`useAnalytics.ts:497`). Similarly, `mcpServerConnected` (`events.ts:187-195`) sends raw `server_name` without calling `sanitizeServerName`.
3. **Hyphenated UUIDs evade API-key regex:** `session_resumed` (`ClaudeCodeSession.tsx:857`) and `checkpoint_restored` (`TimelineNavigator.tsx:181`) send full hyphenated UUIDs in `checkpoint_id` (max 12 contiguous alphanumerics, evading `[a-zA-Z0-9]{32,}` at `index.ts:212`).
4. **Windows backslash paths unredacted:** Both `index.ts:210` and `events.ts:663` only match POSIX `/[\w\-\/\.]+`, leaving `C:\Users\...` paths unstripped in `error_message` and generic string fields.

---

## 6. Docs / Privacy Policy vs Code Discrepancies

- **No Privacy Policy:** The repository has no `PRIVACY.md`, `SECURITY.md`, or `docs/` directory (`git ls-files`).
- **Outright False `"No Telemetry"` and `"Local Storage"` Claims (`README.md:377-386`):**
  - `README.md:383-384` explicitly states:
    > `3. **Local Storage**: All data stays on your machine`
    > `4. **No Telemetry**: No data collection or tracking`
  - **Code Reality:** Opcode bundles `posthog-js` (`package.json:52`), hardcodes a live PostHog API key (`phc_6seRe1SJkFckJU2qQWeeIy62kaSoaUbCsdVCm1TQZg8` -> `https://us.i.posthog.com`) in `src/lib/analytics/index.ts:30-31`, silently defaults `{ enabled: true, hasConsented: true }` in `src/lib/analytics/consent.ts:26-29`, permits PostHog domains in Tauri's CSP (`src-tauri/tauri.conf.json:27`), and emits telemetry by default from first boot across desktop and `opcode-web` browser modes.
- **Misleading Settings & Dead Consent Copy (`Settings.tsx:706-711`, `AnalyticsConsent.tsx:102-109, 203`):**
  - Claims `"No personal information or file contents collected"` and `"All data is anonymous with random IDs"`, omitting that PostHog receives unmasked client IPs for GeoIP and that sanitization misses `tools_used[]`, custom agent/server names, Claude session/checkpoint UUIDs, and Windows paths in tool errors.
- **Misleading Config Comment & Dead Consent UI:**
  - `src/lib/analytics/index.ts:28` states `"// Default configuration - pulled from Vite environment variables"`, yet lines 30–31 hardcode the production key and host directly in source.
  - `src/components/AnalyticsConsent.tsx` renders consent prompts only when `!settings?.hasConsented`, which can never trigger under `consent.ts:28`, and is not imported anywhere.

---

## 7. Zero-Telemetry / Negative-Check Verification

**Not zero-telemetry** — Opcode transmits default-ON PostHog analytics to `https://us.i.posthog.com`. The following negative checks document the absence of *other* SDKs, updaters, CI key injections, and external credential harvesting:

1. **No Sentry, OpenTelemetry, Mixpanel, Amplitude, Segment, Datadog, Statsig, Plausible, or IP/GeoIP lookup APIs, and no DNT/GPC handlers:**
   ```powershell
   rg -n "\b(sentry|opentelemetry|otel|statsig|mixpanel|amplitude|segment|datadog|bugsnag|rollbar|logrocket|plausible|umami|clarity|heap|fullstory|hotjar|matomo|ipify|geoip|ip-api|ipinfo|doNotTrack|DO_NOT_TRACK|globalPrivacyControl|tauri_plugin_updater|plugin-updater|machine_uid|machine-id|getmac)\b" src src-tauri/src package.json src-tauri/Cargo.toml
   ```
   - **Result:** Zero telemetry matches (only `src-tauri/Cargo.toml:32` unused `tauri-plugin-updater = "2"` and a comment word `"clarity"` in `src/components/FloatingPromptInput.tsx:399`).
2. **No Active Auto-Updater:**
   ```powershell
   rg -n "tauri_plugin_updater|plugin-updater|updater" src src-tauri package.json
   ```
   - **Result:** Matches only `src-tauri/Cargo.toml:32`, `src-tauri/Cargo.lock:3586,5685`, and `src-tauri/capabilities/default.json:52`; zero registration in `src-tauri/src/main.rs`, zero endpoints in `src-tauri/tauri.conf.json`, and zero JS calls in `src/`.
3. **No Third-Party Auth Harvesting or Machine Fingerprinting in Rust:**
   ```powershell
   rg -n "\b(token|auth|credentials|password|secret|machine|hostname|whoami|uuid|Uuid)\b|home_dir|config_dir|data_dir|cache_dir|app_data_dir|\.claude|\.codex|\.cursor|\.vscode|\.ssh|\.aws|Application Support|AppData|api_key|apiKeyHelper|reqwest|https?://" src-tauri/src
   ```
   - **Result:** Zero matches for `~/.ssh`, `~/.aws`, `~/.codex`, `~/.cursor`, `~/.vscode`, `gh auth`, hostnames, MAC addresses, or OS Keychains. Disk/HTTP reads outside workspace folders are restricted to `~/.claude`, `app_data_dir/agents.db`, explicit user-triggered Claude Desktop MCP import (`src-tauri/src/commands/mcp.rs:462-525`), and explicit user-triggered GitHub CC agent fetching (`src-tauri/src/commands/agents.rs:1823-1926`).
4. **No CI Secret Injection or `.env` Files:**
   ```powershell
   rg -n --hidden "VITE_PUBLIC_POSTHOG|phc_6seRe1SJkFckJU2qQWeeIy62kaSoaUbCsdVCm1TQZg8" .
   ```
   - **Result:** Matches only `src/lib/analytics/index.ts:30` and `src/main.tsx:48, 50`. Confirmed zero matches across `.github/workflows/*.yml` and zero `.env*` files in the repository.
5. **Unreferenced Telemetry Wrappers Confirmed Unused:**
   ```powershell
   rg -n "api-tracker|PerformanceTracker" src
   rg -n "AnalyticsConsent" src
   ```
   - **Result:** `src/lib/api-tracker.ts`, `src/hooks/usePerformanceMonitor.ts`, `PerformanceTracker`, and `src/components/AnalyticsConsent.tsx` have zero external callers.

---

```json
{
  "slug": "opcode",
  "platform": "Opcode (Claudia)",
  "group": "A",
  "repo": "winfunc/opcode",
  "sha": "c24004d3bbc910bd30b49d402daec254dac26ab2",
  "commit_date": "2026-11-06T19:53:42+05:30",
  "license": "AGPL-3.0",
  "status": "Active (getAsterisk/claudia redirects here)",
  "default_state": "Opt-out (ON by default on first launch across Tauri desktop and opcode-web mode via hardcoded PostHog key and silent enabled: true, hasConsented: true defaults; AnalyticsConsent/Banner are unmounted dead code and check !hasConsented)",
  "opt_out_controls": "Settings -> General -> Enable Analytics switch (sets localStorage['opcode-analytics-settings'].enabled = false and calls posthog.opt_out_capturing() before trailing settings_changed check); no env var or CLI flag",
  "do_not_track": false,
  "opt_out_bypasses": "Unconditional <PostHogProvider capture_exceptions: true> ignores ConsentManager if VITE_PUBLIC_POSTHOG_* env vars are supplied at build time (unset in CI workflows); AnalyticsService.setScreen() calls posthog.capture('$screen') without checking isEnabled(); ConsentManager.deleteAllData() silently resets enabled: true and hasConsented: true (currently uncalled in UI); persistent userId, per-launch sessionId, 5s flush timer, and 2m ResourceMonitor interval stay active when disabled (dropped at track()); shutdown() flushes without isEnabled() check",
  "persistent_ids": "Random UUID v4 userId in localStorage['opcode-analytics-settings'] (retained across opt-out) + PostHog persistence: 'localStorage' + per-launch in-memory sessionId; plus raw hyphenated session/checkpoint UUIDs leaked in checkpoint_id on session_resumed and checkpoint_restored",
  "hashing": "None (random Math.random() UUID v4 only; zero hardware/machine IDs, emails, or account identifiers)",
  "external_auth_reads": "None (reads ~/.claude/* and invokes claude CLI as a Claude Code GUI, and reads Claude Desktop claude_desktop_config.json only on explicit user click of 'Import from Claude Desktop'; zero ~/.ssh, ~/.aws, ~/.codex, ~/.cursor, ~/.vscode, gh auth, or keychain reads)",
  "telemetry_endpoints": [
    "https://us.i.posthog.com (hardcoded project key phc_6seRe1SJkFckJU2qQWeeIy62kaSoaUbCsdVCm1TQZg8; CSP also allows app.posthog.com, *.posthog.com, *.i.posthog.com, *.assets.i.posthog.com)",
    "VITE_PUBLIC_POSTHOG_HOST (optional build-time PostHogProvider env var; unset in CI release workflows)",
    "https://api.github.com/repos/getAsterisk/opcode/contents/cc_agents (non-telemetry; explicit user click in GitHubAgentBrowser)",
    "tauri-plugin-updater declared in Cargo.toml and capabilities/default.json, but unregistered in main.rs with zero endpoints/calls"
  ],
  "ip_geo_handling": "No IP suppression ($ip: null, ip: false, and $geoip_disable: true absent on both PostHog initializers; https://us.i.posthog.com receives raw client IPs)",
  "custom_model_handling": "Chat UI restricts model picker to 'sonnet' | 'opus' (sent verbatim in session_created, model_selected, prompt_submitted, session_stopped); AgentExecution initializes model from SQLite agent.model (which can store arbitrary strings from imported .opcode.json) for local execution only and does not emit modelSelected",
  "docs_discrepancies": "Material contradiction: README.md:383-384 explicitly asserts 'Local Storage: All data stays on your machine' and 'No Telemetry: No data collection or tracking' despite active default-ON PostHog analytics with silent pre-set consent; no PRIVACY.md in repo; Settings and dead AnalyticsConsent UI claim 'No personal information' and 'random IDs' while exposing client IPs and leaking tools_used[], raw custom agent.name in agent_type/feature_name, raw server_name in mcp_server_connected, hyphenated UUIDs in checkpoint_id, and Windows backslash paths in error strings; AnalyticsService comment falsely claims config is pulled from Vite env vars",
  "privacy_rating": "F",
  "one_line_verdict": "Default-ON PostHog telemetry meriting F due to silent pre-set consent (`hasConsented: true`), dead unmounted consent UI, missing IP/GeoIP suppression, zero DNT/env/CLI opt-out, unguarded setScreen/PostHogProvider/deleteAllData edges, retained UUID/timers when disabled, sanitizer leaks, and an outright false 'No Telemetry' / 'Local Storage' claim in README.md."
}
```
