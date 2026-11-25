# Forensic Privacy & Telemetry Audit: OpenAI Codex CLI (`openai/codex`)

- **Slug:** `codex`
- **Platform:** OpenAI Codex CLI
- **Group:** B
- **Repository:** `openai/codex` (`https://github.com/openai/codex`)
- **License:** `Apache-2.0`
- **Status:** Active
- **Branch:** `main`
- **SHA:** `3f65e9f13165d85d1b1fbe20ce06c55758f927b9`
- **Commit Date:** `2026-11-25T04:07:29Z`
- **Commit Subject:** `Enable fixed launch directories for shared MCP REPL servers (#63821)`

---

## 1. Default State & Opt-Out Mechanism

### Default Telemetry State (Split by Entrypoint & Build Mode)
OpenAI Codex CLI operates multiple outbound telemetry, update, and sync channels whose default state depends on the binary entrypoint and build configuration (`debug_assertions`):

1. **Statsig OpenTelemetry Metrics (`ON` by default in release builds for TUI, `exec`, `migrate-rollouts`; `OFF` by default in `app-server` unless `--analytics-default-enabled`)**
   - `codex-rs/core/src/config/otel.rs:22` defaults `otel.metrics_exporter` to `OtelExporterKind::Statsig` (also `codex-rs/config/src/types.rs:736`).
   - `codex-rs/otel/src/config.rs:11-35` resolves `OtelExporter::Statsig` to `OtelExporter::None` when `cfg!(debug_assertions)` is true, and in **release builds** routes OTLP/HTTP JSON metrics to `https://ab.chatgpt.com/otlp/v1/metrics` using a hardcoded `statsig-api-key` (`client-MkRuleRQBd6qakfnDYqJVR9JuXcY57Ljly3vi5JVUIO`).
   - `codex-rs/core/src/otel_init.rs:76-84` disables `OtelExporter::Statsig` only when `!config.analytics_enabled.unwrap_or(default_analytics_enabled)`.
   - `default_analytics_enabled` is **`true`** in the interactive TUI (`codex-rs/tui/src/startup_orchestration.rs:706`), headless `codex exec` (`codex-rs/exec/src/lib.rs:189,730`, including TypeScript SDK `sdk/typescript/src/exec.ts:95`), and `codex migrate-rollouts` (`codex-rs/cli/src/migrate_rollouts.rs:59-64`).
   - `default_analytics_enabled` is **`false`** in `codex app-server` (`codex-rs/app-server/src/main.rs:135`, `codex-rs/cli/src/main.rs:621-638,4290`) unless `--analytics-default-enabled` is supplied (`codex-rs/cli/src/main.rs:4305-4308`). The Python SDK spawns `codex app-server --listen stdio://` without `--analytics-default-enabled` (`sdk/python/src/openai_codex/client.py:256`).

2. **First-Party Codex Analytics Events (`ON` by default across ALL entrypoints, including `app-server`)**
   - `codex-rs/analytics/src/client.rs:405,434` enables the background event queue whenever `analytics_enabled != Some(false)` — meaning **`None` (unset) enables the queue**.
   - **Opt-out / Default Bypass in `app-server`**: Although `codex app-server` sets `default_analytics_enabled = false` for OTEL initialization (`codex-rs/app-server/src/lib.rs:458,660-665,966`), `codex-rs/app-server/src/analytics_utils.rs:8-31` (called by `codex-rs/app-server/src/lib.rs:732-736` and `codex-rs/app-server/src/in_process.rs:544-548`) constructs `AnalyticsEventsClient` using raw `config.analytics_enabled` **without consulting `default_analytics_enabled`**. Likewise, `codex-rs/core/src/session/session.rs:1753-1776` falls back to creating `AnalyticsEventsClient` unless `config.analytics_enabled == Some(false)`. Consequently, standalone `codex app-server` (and Python SDK sessions) still emit `/codex/analytics-events/events` whenever `analytics.enabled` is omitted.
   - Delivery at flush time (`codex-rs/analytics/src/client.rs:1124-1135`) requires local credentials: ChatGPT auth sends all queued event types, OpenAI API key auth sends a 6-event plugin/skill subset (`codex-rs/analytics/src/events.rs:212-222`), and non-OpenAI auth/unauthenticated sessions drop the batch.

3. **OTEL Logs & Traces (`OFF` by default; opt-in self-hosted/custom OTLP only)**
   - `codex-rs/core/src/config/otel.rs:14-43` defaults `exporter = None`, `trace_exporter = None`, `log_user_prompt = false`, `log_agent_responses = false`, and `log_guardian_assessments = false`.

4. **User-Initiated Feedback / Sentry Reports (`ON` by default, manual trigger only)**
   - `config.feedback_enabled` defaults to `true` (`codex-rs/core/src/config/mod.rs:5000-5004`). There is **no automatic crash or panic reporting** to Sentry; uploads occur only when a user invokes `/feedback` or the `feedback/upload` JSON-RPC (`codex-rs/feedback/src/lib.rs:626-638`, `codex-rs/app-server/src/request_processors/feedback_processor.rs:59,361`).

5. **Startup Update Checks, Announcement Prewarm, & Curated Plugin Sync (`ON` by default)**
   - `check_for_update_on_startup` defaults to `true` (`codex-rs/core/src/config/mod.rs:4549`).
   - `tooltips::announcement::prewarm` runs **unconditionally** on every TUI startup (`codex-rs/tui/src/lib.rs:1207`), ignoring both `analytics.enabled = false` and `check_for_update_on_startup = false`.
   - `Feature::Plugins` defaults to `true` (`codex-rs/features/src/lib.rs:2016-2020`), causing `maybe_start_plugin_startup_tasks_for_config` (`codex-rs/app-server/src/message_processor.rs:608`, `codex-rs/core-plugins/src/manager.rs:977-993,3173-3184`) to trigger background sync of `openai/plugins` regardless of `analytics.enabled`.

6. **Statsig Client Feature Gates (`OFF` / dormant in public OSS builds)**
   - `Feature::StatsigGates` is `Stage::UnderDevelopment` with `default_enabled: false` (`codex-rs/features/src/lib.rs:1219-1223`), and `Config::statsig_key` requires compile-time `option_env!("CODEX_BUNDLED_STATSIG_CLIENT_KEY")` (`codex-rs/core/src/statsig.rs:45-52`), which is unset in `.github/workflows/rust-release*.yml`. `start_statsig` also has **zero non-test callers** in `codex-rs/`.

### Opt-Out Controls & Friction
- **`DO_NOT_TRACK` / `NO_TELEMETRY`:** **Not supported** (`0` matches across the repository).
- **In-App TUI Settings Toggle:** **None.** The `/usage` menu's `"View analytics"` (`codex-rs/tui/src/chatwidget/usage.rs:50`) opens a local/account token-usage dashboard (`codex-rs/tui/src/analytics/client.rs`), not a telemetry toggle.
- **Required Configuration to Suppress Outbound Channels:** Users must manually edit `~/.codex/config.toml` (or pass `--config` overrides):
  ```toml
  check_for_update_on_startup = false
  [analytics]
  enabled = false
  [feedback]
  enabled = false
  [otel]
  metrics_exporter = "none" # also disables any custom OTLP metrics exporter
  [features]
  plugins = false           # stops startup curated plugin sync
  ```
- **Live-Reload / Residual Bypasses Even When Opted Out:**
  - `codex-rs/app-server/src/otel_reloader.rs:58-107` only reloads OTEL exporters on `auth_changes.changed()`, not when a user edits `config.toml` mid-session.
  - Setting `analytics.enabled = false` does **not** stop `tooltips::announcement::prewarm` in TUI (`codex-rs/tui/src/lib.rs:1207`).
  - Setting `analytics.enabled = false` does **not** stop transmission of the persistent `installation_id` or `x-codex-turn-metadata` (containing workspace paths, git remotes, commit SHAs, and `analytics_enabled: false`) on Responses API inference calls (`codex-rs/core/src/responses_metadata.rs:361-482`, `codex-rs/core/src/session/session.rs:785`).
  - `codex-rs/core/src/otel_init.rs:76-84` explicitly preserves custom `otel.metrics_exporter` collectors when `analytics.enabled = false` (also `codex-rs/core/config.schema.json:120`).

---

## 2. External Auth / Local Non-Workspace File Reads

- **No Stealth Cross-Tool Auth Harvesting:** Codex never reads `~/.claude`, `~/.cursor`, `~/.gemini`, `~/.copilot`, or other tools' credential/config files during startup or background telemetry collection.
- **Auth Storage Scope:** Codex reads and writes only its own credentials under `$CODEX_HOME/auth.json` and the OS keyring (`codex-rs/login/src/auth/storage.rs:168,343-387`), plus optional workload/gateway auth explicitly configured by the user. `codex-rs/login/src/auth_env_telemetry.rs:31-45` records **boolean presence flags only** (`openai_api_key_env_present`, `codex_api_key_env_present`, `provider_env_key_present`, `refresh_token_url_override_present`) — never secret values.
- **Explicit User-Initiated External Agent Migration (`/import` & JSON-RPC):**
  - `codex-rs/external-agent-migration/src/source/cla.rs:19-74` and `cur.rs:18-26` read `.claude/`, `CLAUDE.md`, `.claude.json`, OS `Claude` desktop config directories (`~/Library/Application Support/Claude`, `%APPDATA%/Claude`, `~/.config/Claude`), `.cursor/`, `.cursorrules`, `cli-config.json`, `mcp.json`, and `hooks.json`.
  - Verified callers (`rg -n "external_agent_migration|ExternalAgentConfig"`): triggered **solely** by user-initiated `/import` in the TUI (`codex-rs/tui/src/slash_command.rs:29,120`, `codex-rs/tui/src/chatwidget/slash_dispatch.rs:588`, `codex-rs/tui/src/app_server_session/external_agent_config.rs:11-51`) or explicit `externalAgentConfig/detect` / `externalAgentConfig/import` JSON-RPC calls (`codex-rs/app-server/src/external_agent_migration/processor.rs`).
  - When an import runs, summary counts (`import_id`, `source`, `provider_id`, `item_type`, `success_count`, `failed_count`, or `failure_stage`/`error_type`) are emitted to `AnalyticsEventsClient` (`codex_onboarding_external_agent_import_complete` / `_failure`, `codex-rs/analytics/src/reducer.rs:1646-1687`).
- **Workspace Plugin Manifest Compatibility:** `codex-rs/core-plugins/src/marketplace.rs:23-24` and `manifest.rs:685` recognize `.claude-plugin/plugin.json` and `.cursor-plugin/plugin.json` formats inside loaded plugin/marketplace directories, not external user-home credential stores.

---

## 3. Persistent Device / Install IDs & Hashing

### Persistent Install Identifier (`installation_id`)
- **Generation & Storage:** `codex-rs/runtime-agent-instance/src/installation_id.rs:17-73` creates and persists a random UUIDv4 at `<codex_home>/installation_id` (mode `0o644` on Unix).
- **Unconditional Transmission on Inference Requests (Even When Analytics Are Disabled):**
  - `codex-rs/core/src/responses_metadata.rs:361-398` populates `client_metadata` on every Responses API request (`codex-rs/core/src/client.rs:1368,1386,2173,2572-2579`, `codex-rs/codex-api/src/common.rs:432,459,505-524`), **ignoring its `include_internal: bool` parameter**.
  - Both `client_metadata["x-codex-installation-id"]` and `x-codex-turn-metadata.installation_id` (alongside `session_id`, `thread_id`, `turn_id`, `window_id`, `workspaces`, and `analytics_enabled`) are attached on HTTP and WebSocket Responses requests (`codex-rs/core/src/client.rs:1160-1174,1710,1780-1783`), **even when `analytics.enabled = false`** (`codex-rs/core/src/session/session.rs:785` explicitly sets `analytics_enabled: Some(false)` inside the metadata blob) and **even to custom non-OpenAI Responses API providers**.
  - When Remote Control is enabled (`codex-rs/app-server-transport/src/transport/remote_control/controller.rs:81,88-89`, `websocket.rs:1376-1409`), `installation_id` and `gethostname()` (`x-codex-name`) are sent as headers.
  - If `StatsigGates` were activated, `installation_id` is passed as `stable_id` to `StatsigBootstrapContext` (`codex-rs/core/src/statsig.rs:64`, `codex-rs/backend-client/src/client/statsig.rs:19`).

### Hashing Practices (Unsalted SHA-1)
- **Repository Remote Hash (`repo_hash`):** `codex-rs/analytics/src/accepted_lines.rs:63-70,106-116` computes `fingerprint_hash("repo", &canonical_remote_url)` using **unsalted deterministic SHA-1** over `b"file-line-v1\0repo\0" + canonical_remote_url`. Because there is no per-user salt, identical repositories hash to the exact same `repo_hash` across all users/organizations. (Note: `line_fingerprints` itself is statically empty `[(); 0]` — `codex-rs/analytics/src/events.rs:236-238`.)
- **Skill Identifier Hash (`skill_id`):**
  - `codex-rs/analytics/src/reducer.rs:4172-4188` computes `skill_id_for_local_skill` via **unsalted SHA-1** over `"{prefix}_{path}_{skill_name}"` (where `prefix` is `"repo_{url}"` or `"personal"`).
  - `codex-rs/analytics/src/reducer.rs:1504-1505` hashes missing resource skill IDs via unsalted `sha1::Sha1::digest(id.as_bytes())`.

---

## 4. Exact Outbound Telemetry / Analytics / Crash / OTEL / Relay / Update Endpoints

| Channel / Purpose | Exact Endpoint(s) | Trigger & Auth Gate | Default State (Release) | Source Citation |
|---|---|---|---|---|
| **Built-in Statsig OTLP Metrics** | `https://ab.chatgpt.com/otlp/v1/metrics` (`statsig-api-key: client-MkRuleRQBd6qakfnDYqJVR9JuXcY57Ljly3vi5JVUIO`) | OTLP/HTTP JSON metrics from `codex-otel` | **ON** (`tui`, `exec`, `migrate-rollouts`); **OFF** (`app-server` unless `--analytics-default-enabled`); **OFF** in debug builds | `codex-rs/otel/src/config.rs:11-35`, `codex-rs/core/src/config/otel.rs:22`, `codex-rs/core/src/otel_init.rs:76-84` |
| **Codex Analytics Events** | `{chatgpt_base_url}/codex/analytics-events/events` (default `https://chatgpt.com/backend-api/codex/analytics-events/events`) | Batched session/turn/tool/skill/plugin/accepted-line events; ChatGPT auth (all events) or OpenAI API key auth (6 plugin/skill events) | **ON** across **all** surfaces whenever `analytics.enabled != false` (including `app-server`!) | `codex-rs/analytics/src/client.rs:183-188,434,1124-1135`, `codex-rs/model-provider/src/chatgpt_base_url.rs:6`, `codex-rs/app-server/src/analytics_utils.rs:8-31` |
| **Custom OTEL Logs / Traces / Metrics** | User-configured `[otel.exporter]`, `[otel.trace_exporter]`, `[otel.metrics_exporter]` (OTLP HTTP or gRPC) | Only when explicitly configured in `config.toml`; custom metrics remain enabled even if `analytics.enabled = false` | **OFF** (`exporter = "none"`, `trace_exporter = "none"`) | `codex-rs/core/src/config/otel.rs:18-22`, `codex-rs/core/src/otel_init.rs:73-84` |
| **Manual User Feedback (Sentry)** | `https://o33249.ingest.us.sentry.io/api/4510195390611458/envelope/` (from DSN `https://ae32ed50620d7a7792c1ce5df38b3e3e@o33249.ingest.us.sentry.io/4510195390611458`) | Strictly **manual** `/feedback` command or `feedback/upload` RPC (`feedback.enabled`, default `true`). Zero automatic crash capture | **Dormant** until user submits `/feedback` | `codex-rs/feedback/src/lib.rs:62-63,626-638`, `codex-rs/feedback/src/upload.rs:32-51`, `codex-rs/core/src/config/mod.rs:5000-5004` |
| **TUI Announcement Banner Prewarm** | `https://raw.githubusercontent.com/openai/codex/main/announcement_tip.toml` | Unconditional HTTP GET on TUI startup | **ON** (no config opt-out) | `codex-rs/tui/src/lib.rs:1207`, `codex-rs/tui/src/tooltips.rs:16-17,188-198,258-267` |
| **Startup Update Checks (TUI & Doctor)** | `https://api.github.com/repos/openai/codex/releases/latest`, `https://formulae.brew.sh/api/cask/codex.json`, `https://registry.npmjs.org/@openai%2fcodex` (`codex doctor` also checks `persistent.oaistatic.com/codex-app-prod/{appcast.xml,appcast-x64.xml,windows-store-update.json}` & `chatgpt.com/backend-api/wham/app/appcast`) | TUI startup every 20h (`check_for_update_on_startup`, default `true`, release builds) and `codex doctor` | **ON** (`check_for_update_on_startup = true`) | `codex-rs/tui/src/updates.rs:1,27-63,86-147`, `codex-rs/tui/src/npm_registry.rs:5`, `codex-rs/cli/src/doctor/updates.rs:36-46` |
| **Curated Plugins Startup Sync** | `https://github.com/openai/plugins.git`, `https://api.github.com/repos/openai/plugins`, `https://chatgpt.com/backend-api/plugins/export/curated` | Background sync on startup/account change when `Feature::Plugins` enabled and remote catalog inactive | **ON** (`Feature::Plugins` default `true`, ignores `analytics.enabled`) | `codex-rs/core-plugins/src/startup_sync.rs:23-31`, `codex-rs/core-plugins/src/manager.rs:977-993,3173-3184`, `codex-rs/app-server/src/message_processor.rs:608` |
| **API-Key Turn Cost Query** | `https://api.chatgpt.com/v1/analytics/codex/turn-costs` or `{provider}/analytics/codex/turn-costs` | Queries turn billing cost only when explicit `OtlpHttp`/`OtlpGrpc` OTEL exporter is active | **OFF** by default (not triggered by `Statsig` or `None`) | `codex-rs/app-server/src/turn_cost_worker.rs:108-120,474`, `codex-rs/backend-client/src/client/turn_usage.rs:48-78` |
| **Statsig Feature Flag Evaluations (Dormant)** | `https://ab.chatgpt.com/v1/initialize`, `.../v1/rgstr`, `.../v1/sdk_exception`; `{base}/api/codex/statsig/bootstrap` or `/wham/statsig/bootstrap` | Requires `Feature::StatsigGates` (`false`), compile-time `CODEX_BUNDLED_STATSIG_CLIENT_KEY`, and a `start_statsig` caller | **OFF / Unused** (0 non-test callers) | `codex-rs/core/src/statsig.rs:45-66`, `codex-rs/statsig/src/evaluations.rs:139-144`, `codex-rs/backend-client/src/client/statsig.rs:34-39` |
| **Install / SDK Runtime Downloaders** | `https://chatgpt.com/codex/install.{sh,ps1}` (daemon updater), `https://api.github.com/repos/openai/codex/releases/tags/...` (Python SDK setup) | Only invoked during explicit install/upgrade or Python SDK runtime setup; `codex-cli/bin/codex.js` makes zero network calls | User/installer action | `codex-rs/app-server-daemon/src/update_loop.rs:75-77`, `sdk/python/_runtime_setup.py:146-148`, `codex-cli/bin/codex.js:1-297` |

---

## 5. Exact Data & Properties Collected

### A. IP Address & Geolocation Handling
- **No IP Scrubbing / Leaves Provenance to Server:** Codex does not strip source IP headers or proxy analytics through an IP-anonymizing relay; direct requests to `ab.chatgpt.com`, `chatgpt.com`, `o33249.ingest.us.sentry.io`, `api.github.com`, `raw.githubusercontent.com`, `formulae.brew.sh`, and `registry.npmjs.org` expose the client's IP address to each endpoint.
- **`RequestCountry` Defaults to Omitted:** `codex-rs/protocol/src/request_country.rs:6-16` defaults `RequestCountry` to `Country::Omitted` (*"The default leaves provenance to the server"*), only populating a 2-letter ISO code or `"XX"` (`Unknown`) if explicitly supplied by a trusted host.
- **Infrastructure Cookies:** `codex-rs/http-client/src/chatgpt_cloudflare_cookies.rs:17-20,197-214` restricts global cookie storage on ChatGPT domains strictly to Cloudflare infrastructure cookies (`__cf_bm`, `_cfuvid`, `__cflb`, etc.) and `__oailb`.

### B. Custom / Private Model Names & Provider IDs
- **Sent Verbatim (No Redaction or Allowlist):**
  - `codex-rs/core/src/session/turn.rs:1719-1720` populates `TurnResolvedConfigFact` with raw `turn_context.model_info().slug` and `turn_context.config.model_provider_id`.
  - `codex-rs/analytics/src/reducer.rs:2514,3935,3978-3979,4027-4028` transmits `model` and `model_provider` verbatim in `codex_thread_initialized`, `codex_turn_event` (`codex-rs/analytics/src/events.rs:293,1190-1191`), `codex.accepted_line_fingerprints` (`events.rs:231`), and `skill_invocation`. Because `AnalyticsEventsClient` checks `auth_manager` credentials (`codex-rs/analytics/src/client.rs:1124-1135`), a user logged into ChatGPT who runs a turn against a custom local Ollama/vLLM/private endpoint still leaks the raw custom model slug and provider ID to `chatgpt.com/backend-api/codex/analytics-events/events` unless `analytics.enabled = false`.
  - `codex-rs/otel/src/events/session_telemetry.rs:589-597,639` and `codex-rs/otel/src/metrics/tags.rs:43,54` attach raw `model` as an OTLP metric attribute to `https://ab.chatgpt.com/otlp/v1/metrics`; `sanitize_metric_tag_value` (`codex-rs/utils/string/src/lib.rs:35-56`) only normalizes non-alphanumeric characters and truncates at 256 chars—it never redacts private model names.

### C. Prompts, Code, Local Paths & Stack Traces
- **Analytics Events (`/codex/analytics-events/events`):**
  - **Prompts/Code:** Not included; `line_fingerprints` is hardcoded to `[]` (`codex-rs/analytics/src/events.rs:236-238`), keeping only `accepted_added_lines` and `accepted_deleted_lines` counts plus unsalted SHA-1 `repo_hash`.
  - **Local Paths:** Plugin command-execution telemetry strips absolute paths via `safe_plugin_relative_script_path` (`codex-rs/analytics/src/reducer.rs:3440-3446,4427-4440`). However, `skill_id_for_local_skill` hashes personal skill paths into unsalted SHA-1 (`reducer.rs:4172-4188`).
- **Responses API Metadata (`client_metadata` & `x-codex-turn-metadata`):**
  - Sent on every inference turn (even with `analytics.enabled = false`): `workspaces` embeds **raw local repository root paths** (`codex-rs/core/src/turn_metadata.rs:661`), `associated_remote_urls` (sanitized git remote URLs), `latest_git_commit_hash`, and `has_changes` (`turn_metadata.rs:664-750`, `codex-rs/core/src/responses_metadata.rs:471,690`).
- **OTEL Logs & Traces (Only When User Configures `otel.exporter` / `otel.trace_exporter`):**
  - Prompts are replaced with `"[REDACTED]"` unless `otel.log_user_prompt = true` (`codex-rs/otel/src/events/session_telemetry.rs:1230-1241`).
  - Final agent responses and Guardian rationales are suppressed unless `otel.log_agent_responses = true` or `otel.log_guardian_assessments = true` (`codex-rs/core/src/config/otel.rs:35-36`).
  - `log_event!` (`codex-rs/otel/src/events/shared.rs:14-31`) includes `user.account_id` and `user.email` on `OTEL_LOG_ONLY_TARGET`; `trace_event!` (`shared.rs:34-49`) excludes them.
  - `ResourceKind::Logs` attaches local machine hostname (`host.name` via `gethostname()`, `codex-rs/otel/src/provider.rs:66,471-482`), whereas metrics resources do not.
- **Manual `/feedback` Sentry Uploads:**
  - Only when user confirms upload (`codex-rs/tui/src/bottom_pane/feedback_view.rs:206-310`): uploads user note/reason, `thread_id`, `classification`, `cli_version`, `session_source`, `account_id` / `chatgpt_user_id` tags (`codex-rs/feedback/src/lib.rs:817-860`, `codex-rs/app-server/src/request_processors/feedback_processor.rs:154-164`), plus consent-selected attachments (`codex-logs.log`, session rollout JSONL containing prompts/commands/tool outputs, Windows sandbox log, MCP/app directory caches, and `codex-connectivity-diagnostics.txt` containing redacted proxy env vars `http_proxy`/`https_proxy`/`all_proxy`, `codex-rs/feedback/src/feedback_diagnostics.rs:3-10,43-61`).

### D. Behavioral / System Profile Fields
- **Statsig OTLP Metrics (`ab.chatgpt.com/otlp/v1/metrics`):**
  - Resource attributes: `service.version`, `env` (`dev` default), `os` and `os_version` via `os_info::get` (`codex-rs/otel/src/metrics/client.rs:380,617-633`, `codex-rs/otel/src/provider.rs:464-470`).
  - Metric tags via `SessionMetricTagValues` (`codex-rs/otel/src/metrics/tags.rs:38-57`): `auth_mode`, `session_source`, `originator`, `service_name`, `model`, `app.version`. High-volume metrics in `STATSIG_DISABLED_METRICS` (`codex-rs/otel/src/metrics/config.rs:25-38`, e.g., `codex.api_request`, `codex.tool.call`, `codex.turn.token_usage`, `codex.turn.cost_microusd`) are filtered out of the Statsig route and reserved for custom OTLP collectors.
- **Codex Analytics Events (`chatgpt.com/backend-api/codex/analytics-events/events`):**
  - `CodexRuntimeMetadata` (`codex-rs/analytics/src/events.rs:280-285,1620-1630`): `codex_rs_version`, `runtime_os`, `runtime_os_version` (`os_info`), `runtime_arch`.
  - `CodexTurnEventParams` (`codex-rs/analytics/src/events.rs:1168-1213`, `codex-rs/analytics/src/reducer.rs:4009-4050`): `thread_id`, `session_id`, `turn_id`, `app_server_client` (`product_client_id`, `client_name`, `client_version`, `rpc_transport`), `model`, `model_provider`, `sandbox_policy`, `reasoning_effort`, `reasoning_summary`, `service_tier`, `approval_policy`, `approvals_reviewer`, `guardian_v2_enabled`, `multi_agent_version`, `sandbox_network_access`, `collaboration_mode`, `personality`, `workspace_kind`, `is_worktree`, `num_input_images`, `status`, error codes, critical-path latencies, token usage, and tool/command action counts (never raw command strings).

---

## 6. Docs / Privacy Policy vs Code Discrepancies

1. **App-Server Default Analytics Contradiction (`codex-rs/cli/src/main.rs:621-638` vs `codex-rs/app-server/src/analytics_utils.rs:25-31`):**
   - `codex app-server --help` / docstring states: *"Analytics are disabled by default for app-server. Users have to explicitly opt in via the `analytics` section in the config.toml file."* Similarly `codex-rs/core/config.schema.json:8044` states for `analytics`: *"When unset, the default depends on the client."*
   - In code, `default_analytics_enabled = false` in `app-server` disables only Statsig OTLP metrics (`codex-rs/core/src/otel_init.rs:76-84`), while `analytics_events_client_from_config` (`codex-rs/app-server/src/analytics_utils.rs:28`) and `Session` (`codex-rs/core/src/session/session.rs:1753-1776`) enable `AnalyticsEventsClient` whenever `analytics_enabled != Some(false)` — emitting `/codex/analytics-events/events` by default when unset.
2. **Off-Repo Telemetry Documentation:**
   - `README.md`, `docs/config.md:1-7`, `docs/authentication.md:1-3`, and `codex-rs/config.md:1-6` contain **zero** inline documentation about default-enabled Statsig OTLP metrics, `/codex/analytics-events/events`, unconditional `announcement_tip.toml` fetching, or how to opt out (`config.md` redirects to external `developers.openai.com/codex/config-*`).
   - Only `codex-rs/core/config.schema.json:120,4154,8044` and `codex-rs/otel/README.md` document telemetry fields in-repo (and `codex-rs/otel/README.md` covers `codex-otel` log/trace options without documenting the default-enabled Statsig metrics or `AnalyticsEventsClient` pipeline).
3. **Unconditional Startup Requests Undocumented in Config:**
   - `codex-rs/tui/src/lib.rs:1207` hits `raw.githubusercontent.com/openai/codex/main/announcement_tip.toml` unconditionally at TUI startup with no config toggle or documentation reference.
   - `installation_id` and `x-codex-turn-metadata` (including local workspace path + git remotes + `analytics_enabled: false`) continue sending on Responses API requests even when `analytics.enabled = false`, ignoring `include_internal` in `codex-rs/core/src/responses_metadata.rs:361-398`.

---

## 7. Zero-Telemetry / Negative-Check Verification

Codex CLI is **not** a zero-telemetry repository; it actively emits default-on OTEL Statsig metrics and analytics events. However, targeted negative checks confirm the absence of `DO_NOT_TRACK` support and third-party commercial analytics SDKs:

- **`DO_NOT_TRACK` / `NO_TELEMETRY` Check (Negative — 0 Matches):**
  - Command: `rg -ni "do_not_track|donottrack|no_telemetry|disable_telemetry" codex-rs/ codex-cli/ sdk/ docs/ .github/ README.md`
  - Result: **0 matches** (exit code 1). Codex ignores standard `DO_NOT_TRACK=1` environment variables.
- **Third-Party Analytics / Crash SDKs Check (Negative — 0 Telemetry SDK Matches):**
  - Command: `rg -ni "posthog|mixpanel|amplitude|bugsnag|rollbar|raygun|crashlytics|honeycomb|fullstory|logrocket|heap\.io|datadog" codex-rs/ codex-cli/ sdk/ docs/ .github/ README.md -g "!uv.lock" -g "!Cargo.lock" -g "!*.snap"` and `rg -n "\bsegmentio\b|\bsegment\.com\b|analytics\.js" codex-rs/ codex-cli/ sdk/ docs/ .github/ README.md -g "!uv.lock" -g "!Cargo.lock"`
  - Result: **0 analytics/telemetry SDK usages** (only false-positive hits for TUI CSS `scrollbar`, audio `VoiceAmplitudeHistory`, skill-creator `"datadog": "DataDog"` label map, and test-client `[Datadog trace]` print helper). The only third-party crash/reporting SDK dependency is `sentry` (`codex-rs/feedback/`), used solely for explicit manual `/feedback` submissions.
- **Stealth Non-Workspace Credential Reads (Negative):**
  - Verified all matches of `.claude` / `.cursor` / `.gemini` / `.copilot` / `.windsurf` / `.opencode` and `external_agent_migration` across `codex-rs/`: outside workspace/plugin manifest parsing, `.claude`/`.cursor` reads exist strictly inside `codex-external-agent-migration` and execute only upon explicit `/import` or `externalAgentConfig/{detect,import}`.

---

```json
{
  "slug": "codex",
  "platform": "OpenAI Codex CLI",
  "group": "B",
  "repo": "openai/codex",
  "sha": "3f65e9f13165d85d1b1fbe20ce06c55758f927b9",
  "commit_date": "2026-11-25T04:07:29Z",
  "license": "Apache-2.0",
  "status": "Active",
  "default_state": "ON by default in release builds (Statsig OTLP metrics + Codex analytics events for TUI/exec/migrate-rollouts/TS-SDK; update checks + curated plugin sync + TUI announcement prewarm also ON). Standalone app-server disables Statsig OTEL metrics by default but leaves AnalyticsEventsClient ON.",
  "opt_out_controls": "Config file / CLI override only ([analytics] enabled = false, check_for_update_on_startup = false, [feedback] enabled = false, [otel] metrics_exporter = \"none\", [features] plugins = false); no TUI settings toggle and no master switch.",
  "do_not_track": false,
  "opt_out_bypasses": [
    "app-server AnalyticsEventsClient ignores default_analytics_enabled=false and stays ON when analytics.enabled is unset (analytics_utils.rs:28)",
    "TUI tooltips::announcement::prewarm unconditionally fetches raw.githubusercontent.com/openai/codex/main/announcement_tip.toml (tui/src/lib.rs:1207)",
    "Responses API client_metadata (ignoring include_internal) and x-codex-turn-metadata still transmit installation_id, workspace paths, git remotes, and analytics_enabled=false when opted out (responses_metadata.rs:361-482)",
    "curated plugins startup sync and custom otel.metrics_exporter collectors are independent of analytics.enabled (otel_init.rs:76-84, manager.rs:977-993)",
    "app-server otel_reloader only reloads on auth changes, not mid-session config.toml edits (otel_reloader.rs:58-107)"
  ],
  "persistent_ids": [
    "<codex_home>/installation_id (UUIDv4 persisted and sent in Responses API client_metadata/headers even when analytics.enabled=false, and in Remote Control)",
    "session_id / thread_id / turn_id / window_id",
    "account_id / chatgpt_user_id / email (when ChatGPT-authenticated; user.account_id and user.email in OTEL log_event!; account_id/chatgpt_user_id in manual /feedback Sentry tags)"
  ],
  "hashing": "Unsalted deterministic SHA-1 for repo_hash (\"file-line-v1\\0repo\\0\" + canonical git remote URL in accepted_lines.rs:63-116) and skill_id (reducer.rs:1505,4184-4187).",
  "external_auth_reads": "None unconditionally; .claude/.claude.json/.cursor/.cursorrules/Claude desktop dirs are read strictly on user-initiated /import or externalAgentConfig/{detect,import} JSON-RPC.",
  "telemetry_endpoints": [
    "https://ab.chatgpt.com/otlp/v1/metrics (Statsig OTLP metrics, hardcoded statsig-api-key, release default ON)",
    "https://chatgpt.com/backend-api/codex/analytics-events/events (Codex analytics events, default ON whenever analytics.enabled != false)",
    "https://o33249.ingest.us.sentry.io/api/4510195390611458/envelope/ (Sentry feedback upload, manual /feedback only)",
    "https://raw.githubusercontent.com/openai/codex/main/announcement_tip.toml (unconditional TUI startup prewarm)",
    "https://api.github.com/repos/openai/codex/releases/latest, https://formulae.brew.sh/api/cask/codex.json, https://registry.npmjs.org/@openai%2fcodex (startup update checks)",
    "https://github.com/openai/plugins.git, https://api.github.com/repos/openai/plugins, https://chatgpt.com/backend-api/plugins/export/curated (startup curated plugins sync)",
    "https://api.chatgpt.com/v1/analytics/codex/turn-costs (turn cost query only when custom OTLP collector configured)",
    "https://ab.chatgpt.com/v1/{initialize,rgstr,sdk_exception} & /wham/statsig/bootstrap (dormant; Feature::StatsigGates=false and 0 non-test callers)"
  ],
  "ip_geo_handling": "No IP suppression or anonymizing proxy; direct connections expose client IP to OpenAI/Statsig/Sentry/GitHub/npm/Homebrew. RequestCountry defaults to Omitted (leaving provenance to server).",
  "custom_model_handling": "Verbatim unredacted model slug and model_provider sent in AnalyticsEventsClient (codex_turn_event, codex_thread_initialized, codex_accepted_line_fingerprints) and Statsig OTLP SessionMetricTagValues.model (even for private/local models if ChatGPT auth is present).",
  "docs_discrepancies": [
    "cli/src/main.rs:621-624 claims analytics are disabled by default for app-server, but AnalyticsEventsClient remains ON unless analytics.enabled=false is explicitly set",
    "README.md, docs/config.md, and codex-rs/config.md omit all documentation of default-on Statsig OTLP metrics, AnalyticsEventsClient, and opt-out steps (moved off-repo)",
    "Unconditional TUI announcement_tip.toml fetch and persistent installation_id/workspace metadata transmission when analytics.enabled=false are undocumented"
  ],
  "privacy_rating": "D",
  "one_line_verdict": "Default-on Statsig OTLP metrics and Codex analytics events lack DO_NOT_TRACK or UI toggles, leak verbatim custom model slugs, use unsalted SHA-1 repo/skill hashes, bypass app-server's documented opt-in default for AnalyticsEventsClient, and still transmit persistent installation_id and workspace git metadata when opted out."
}
```
