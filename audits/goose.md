# Forensic Telemetry & Privacy Audit: Goose (`aaif-goose/goose`)

## Header Metadata

- **Slug**: `goose`
- **Platform**: `Goose`
- **Group**: `B`
- **Repository**: `https://github.com/aaif-goose/goose` (`block/goose` redirects here)
- **Audit Target Commit**: `766923df7a16679ee430eac43c9fc9c73caf4b33` (`main`)
- **Commit Date**: `2026-11-25T04:38:40Z`
- **Commit Subject**: `chore(oi-kit): release v0.1.0-alpha.16 (#13580)`
- **License**: `Apache-2.0` ([`ui/desktop/package.json:167`](ui/desktop/package.json), [`README.md:8`](README.md))
- **Status**: `Active (block/goose redirects here)`
- **Scope**: Strictly read-only static source audit covering Rust core/CLI/ACP crates (`crates/goose`, `crates/goose-cli`, `crates/goose-roaming`, `crates/oi-kit-*`), Electron desktop app (`ui/desktop/`), helper services (`services/`, `oidc-proxy/`, `buzz/`), CI release workflows (`.github/workflows/`), and documentation (`README.md`, `CUSTOM_DISTROS.md`, `documentation/`).

---

## 1. Default State & Opt-Out Mechanism

### Default Runtime State
Goose splits outbound network activity across five subsystems with different default behaviors:

1. **Product Analytics (`crates/goose/src/posthog.rs`) — Backend Opt-In (`false` when unset), Pre-Checked UI/CLI Nudges**:
   - Compiled by default in CLI and Desktop-backed binaries (`crates/goose-cli/Cargo.toml:99,122,136` includes `telemetry` in both `default` and `portable-default` features; `crates/goose/Cargo.toml:14,95` defines `telemetry = []` and includes it in `portable-default`, though `goose`'s own `default = []` at `crates/goose/Cargo.toml:12`). Lean `goose-acp` omits `telemetry` and `otel` (`.github/workflows/build-acp.yml:86-90`).
   - `is_telemetry_enabled()` returns `get_telemetry_choice().unwrap_or(false)` (`crates/goose/src/posthog.rs:50-52`), so **`session_started` telemetry is OFF until explicitly enabled** (`GOOSE_TELEMETRY_ENABLED: true`).
   - **Misleading UI Switch Default**: In Desktop Settings, `TelemetrySettings.tsx` initializes `useState(true)` (`ui/desktop/src/components/settings/app/TelemetrySettings.tsx:54`) and sets `setTelemetryEnabled(value === null ? true : Boolean(value))` (`ui/desktop/src/components/settings/app/TelemetrySettings.tsx:61`). When `GOOSE_TELEMETRY_ENABLED` is unset (`null`), Desktop renders the "Anonymous usage data" toggle **ON**, directly contradicting `is_telemetry_enabled() == false`.
   - **Pre-Checked Consent Nudges**:
     - Interactive CLI first-time setup and interactive session entry (`crates/goose-cli/src/cli.rs:1958-1961,2205-2208,2767-2770`; `crates/goose-cli/src/commands/configure.rs:234-235`) call `configure_telemetry_consent_dialog()` whenever `get_telemetry_choice().is_none()`, presenting `cliclack::confirm(...).initial_value(true)` (`crates/goose-cli/src/commands/configure.rs:210-212`; settings dialog uses `.initial_value(current_choice.unwrap_or(true))` at `crates/goose-cli/src/commands/configure.rs:1605-1607`).
     - Desktop onboarding pre-checks opt-in via `useState(true)` (`ui/desktop/src/components/onboarding/OnboardingSuccess.tsx:51,93-101`). Returning desktop sessions where a provider is already set and `GOOSE_TELEMETRY_ENABLED === null` show `TelemetryConsentPrompt` (`ui/desktop/src/App.tsx:697`; `ui/desktop/src/components/TelemetryConsentPrompt.tsx:54-56`), with "Yes, share anonymous usage data" focused via `autoFocus` (`ui/desktop/src/components/TelemetryConsentPrompt.tsx:102-109`).
   - **Dormant Frontend Analytics & Disabled Backend Events**:
     - Frontend `analytics.ts` is currently stubbed to no-ops (`ui/desktop/src/utils/analytics.ts:8-17`: `// Frontend telemetry is disabled.`), so UI callers (`usePageViewTracking`, `ErrorBoundary`, `OnboardingGuard`, `TelemetryConsentPrompt`, `TelemetrySettings`, `autoUpdater`) emit nothing.
     - Backend `emit_error_with_context()` and `emit_custom_slash_command_used()` immediately return early (`crates/goose/src/posthog.rs:294-326`: `// Temporarily disabled - only session_started events are sent`).

2. **Startup Model Catalog Fetch (`online-model-meta`) — ON by Default, No Runtime Opt-Out**:
   - Both `goose-cli` (`crates/goose-cli/Cargo.toml:110`, `crates/goose-cli/src/main.rs:24-25`) — which Desktop also spawns as `goose serve --platform desktop` (`ui/desktop/src/gooseServe.ts:365-375`) — and lean `goose-acp` (`crates/goose/src/bin/goose-acp.rs:33-34`, `.github/workflows/build-acp.yml:89`) call `goose::model_catalog::initialize()` unconditionally on startup.
   - This spawns an immediate HTTPS GET to `https://models.dev/api.json` (`crates/goose/src/model_catalog.rs:7,21-25`) regardless of `GOOSE_TELEMETRY_OFF`, `GOOSE_TELEMETRY_ENABLED`, or `DO_NOT_TRACK`. There is **no runtime flag or config key** to disable it (only compile-time `--no-default-features` without `online-model-meta`, used in `portable-default` musl builds per `crates/goose-cli/Cargo.toml:131-143` and `.github/workflows/build-cli-linux.yml:144-146`).

3. **Desktop Startup Update Check (`UPDATES_ENABLED = true`) — ON by Default, No Runtime Check Opt-Out**:
   - `ui/desktop/src/updates.ts:1` hardcodes `UPDATES_ENABLED = true`. `ui/desktop/src/main.ts:72-75,2498-2511` unconditionally runs `setupAutoUpdater()` 2 seconds after window creation, which waits another 5 seconds (`ui/desktop/src/utils/autoUpdater.ts:414-533`) and triggers `autoUpdater.checkForUpdates()` against GitHub Releases (falling back to `https://api.github.com/repos/aaif-goose/goose/releases/latest` at `ui/desktop/src/utils/autoUpdater.ts:463-469` and `ui/desktop/src/utils/githubUpdater.ts:502-529`).
   - Setting `disableAutoDownload` in Desktop settings or `GOOSE_DISABLE_AUTO_DOWNLOAD=1` / `true` (`ui/desktop/src/utils/autoUpdater.ts:49-53,374-384`) only stops background **downloading** of updates; there is **no runtime setting or env var to stop the startup update check**.

4. **OpenTelemetry (OTLP) & Langfuse Tracing — OFF by Default (Explicit Endpoint/Key Opt-In)**:
   - OTLP (`crates/goose/src/otel/otlp.rs:227-258`) is disabled unless `OTEL_EXPORTER_OTLP_ENDPOINT`, `OTEL_EXPORTER_OTLP_{SIGNAL}_ENDPOINT`, `OTEL_{SIGNAL}_EXPORTER` (to `otlp`/`console`), or `config.yaml` key `otel_exporter_otlp_endpoint` (`crates/goose/src/otel/otlp.rs:261-272`) is set, and can also be killed via `OTEL_SDK_DISABLED=true`.
   - Langfuse (`crates/goose/src/tracing/langfuse_layer.rs:155-186`) is disabled unless both a public key (`LANGFUSE_PUBLIC_KEY` or `LANGFUSE_INIT_PROJECT_PUBLIC_KEY`) and secret key (`LANGFUSE_SECRET_KEY` or `LANGFUSE_INIT_PROJECT_SECRET_KEY`) are non-empty.

5. **P2P Roaming (`goose roam`) & ML Security Classifier — OFF Unless Explicitly Invoked/Enabled**:
   - `goose roam` only binds an `iroh` endpoint and contacts relays when a user runs `goose roam share` / `connect` / `bridge` (`crates/goose-cli/src/commands/roam.rs:1-16`; `crates/goose-roaming/src/node.rs:137-160`).
   - ML prompt injection classification (`SECURITY_PROMPT_CLASSIFIER_ENDPOINT`) is gated on `SECURITY_PROMPT_CLASSIFIER_ENABLED` defaulting to `false` (`crates/goose/src/security/scanner.rs:81-83,95-110`).

### Opt-Out Controls, Precedence, & Bypasses
- **Config precedence** (`crates/goose/src/config/base.rs:160-179,757-780`): `get_param::<T>(key)` checks uppercase environment variable first, then merged YAML configs (`GOOSE_SYSTEM_CONFIG` / `/etc/goose/config.yaml` or `%PROGRAMDATA%\goose\config.yaml`, `GOOSE_ADDITIONAL_CONFIG_FILES`, and user `config.yaml`).
- **Working telemetry opt-out controls**:
  - `GOOSE_TELEMETRY_OFF=1` or `true` (case-insensitive) overrides config and forces `get_telemetry_choice()` to `Some(false)` (`crates/goose/src/posthog.rs:22-27,33-36`).
  - `GOOSE_TELEMETRY_ENABLED=false` (env var or `config.yaml`) disables PostHog `session_started`.
  - UI `Settings -> App -> Privacy -> Anonymous usage data` (`ui/desktop/src/components/settings/app/TelemetrySettings.tsx:49,78-84`) and CLI `goose configure -> goose settings -> Telemetry` (`crates/goose-cli/src/commands/configure.rs:1507-1508,1587-1617`) toggle `GOOSE_TELEMETRY_ENABLED`.
- **`DO_NOT_TRACK`**: **Not supported** (`0` matches in `rg -n "DO_NOT_TRACK"` across the entire repo).
- **Non-functional documented variable**: `CUSTOM_DISTROS.md:93` instructs setting `GOOSE_DISABLE_TELEMETRY=1`, which is **never read in code** (`0` code matches).
- **Latent PostHog Consent / Opt-Out Bypass (`crates/goose/src/posthog.rs:556-568`)**: `posthog::emit_event()` **does not check `is_telemetry_enabled()`, `get_telemetry_choice()`, or `GOOSE_TELEMETRY_OFF`** when `event_name` starts with `"onboarding_"` or equals `"telemetry_preference_set"` (`// Only onboarding events are enabled for now. These bypass the telemetry check so we can track the funnel before the user makes their choice.`), and writes `telemetry_installation.json` (`posthog.rs:568`). Currently dormant because frontend `analytics.ts:12-17` is a no-op and the only active callers in `crates/goose/src/scheduler/full.rs:946,1068` emit `"schedule_job_started"` / `"schedule_job_completed"`, which return `Ok(())` at `posthog.rs:564-566`.

---

## 2. External Auth / Local Non-Workspace File Reads

Goose does **not** harvest external CLI auth files for telemetry, cross-provider user tracking, or background probing. All non-workspace local file reads fall into **explicitly user-configured providers, explicit session/config import actions, or standard local skill/agent/config discovery**:

| Source / Path Read | Module & Lines | Condition / Purpose |
|---|---|---|
| Goose config/state/data/secrets (`~/.config/goose/`, `~/.local/share/goose/`, `~/.local/state/goose/`, or `%APPDATA%\Block\goose\...`; override via `GOOSE_PATH_ROOT`) + system `/etc/goose/config.yaml` / `%PROGRAMDATA%\goose\config.yaml` | `crates/goose/src/config/paths.rs:8-46`, `crates/goose/src/config/base.rs:160-179` | Core Goose runtime configuration, SQLite sessions, `telemetry_installation.json`, `instance_id`, Goose-managed OAuth token caches (`chatgpt_codex/tokens.json` at `chatgpt_codex.rs:550`, `gemini_oauth/tokens.json` at `gemini_oauth.rs:153`, `githubcopilot/info.json` at `githubcopilot.rs:181-184`, `kimicode/token.json` & `kimicode/device_id` at `kimicode.rs:105,198`, `xai_oauth/tokens.json` at `xai_oauth.rs:106`, `huggingface/oauth/tokens.json` at `huggingface_auth.rs:10,32-34`, `databricks/oauth` at `oauth.rs:129`). |
| Claude Desktop config (`~/.config/Claude/claude_desktop_config.json`, `~/Library/Application Support/Claude/claude_desktop_config.json`, `~/AppData/Roaming/Claude/claude_desktop_config.json`) | `crates/goose/src/acp/server/onboarding.rs:29-54,180-200`, dispatched at `crates/goose/src/acp/server/custom_dispatch.rs:473-478` | **Only** when the Desktop onboarding import flow explicitly calls `OnboardingImportScanRequest` / `OnboardingImportApplyRequest` to import MCP server definitions (with user approval). |
| Claude Code / Codex / Pi session transcripts (`~/.claude/projects/...`, `~/.codex/sessions/...`, `~/.pi/agent/sessions/...`) | `crates/goose/src/session/import_formats/mod.rs:1-14,150-157`, `crates/goose/src/session/session_manager.rs:2847-2854` | Format conversion only when user explicitly invokes `import_session` on provided transcript content; no directory scanning on startup. |
| `~/.claude/agents`, `~/.claude/skills`, `~/.goose/*`, `~/.agents/*`, `~/.config/agents/skills` (plus project `.claude/`, `.goose/`, `.agents/`) | `crates/goose/src/sources.rs:461-475`, `crates/goose/src/skills/mod.rs:170-189`, `crates/goose/src/agents/platform_extensions/summon.rs:250-293` | Explicit MCP/ACP skill/subagent/source discovery and loading when agent or UI enumerates local agent/skill definitions. |
| Google Application Default Credentials (`GOOGLE_APPLICATION_CREDENTIALS` or `~/.config/gcloud/application_default_credentials.json` / `%APPDATA%\gcloud\application_default_credentials.json`) | `crates/goose/src/providers/gcpauth.rs:195-260` | Only when Vertex AI / GCP provider is active. |
| AWS SDK standard chain (`~/.aws/*`, `AWS_PROFILE`, AWS SSO) & Azure CLI (`az account get-access-token`) | `crates/goose/src/providers/bedrock.rs:269-285,335-344`, `crates/goose/src/providers/azureauth.rs:134-169` | Only when AWS Bedrock or Azure OpenAI / Azure Foundry provider is configured and invoked. |
| External coding CLI subprocesses (`claude`, `codex`, `gemini`, `cursor-agent`) | `crates/goose/src/providers/claude_code.rs`, `codex.rs`, `gemini_cli.rs`, `cursor_agent.rs:62-71` | Only when user selects that CLI-backed provider; Goose delegates auth to those installed binaries rather than reading `~/.claude` or `~/.codex` auth tokens directly. |

---

## 3. Persistent Device / Install IDs & Hashing

1. **PostHog `installation_id` (`telemetry_installation.json`)**:
   - Path: `Paths::state_dir().join("telemetry_installation.json")` (`crates/goose/src/posthog.rs:113-115`).
   - Generation: Random UUID v4 (`Uuid::new_v4().to_string()`), paired with `first_seen` timestamp (`DateTime<Utc>`) and `session_count: u32` (`crates/goose/src/posthog.rs:97-111`).
   - Created lazily when `emit_session_started()` runs (only if `is_telemetry_enabled()` is `true` at `crates/goose/src/posthog.rs:265-270`) or inside `emit_event()` for onboarding events (no callers currently active; `crates/goose/src/posthog.rs:568`). Sent verbatim as PostHog `distinct_id` (`crates/goose/src/posthog.rs:75,450,603`).

2. **Databricks Optional `instance_id` (`instance_id`)**:
   - Path: `Paths::state_dir().join("instance_id")` (`crates/goose/src/instance_id.rs:8-10`).
   - Generation: Random UUID v4 (`crates/goose/src/instance_id.rs:12-30`).
   - **Gated OFF by default**: Only initialized and sent to Databricks AI Gateway (`sessionId: "<instance_id>_<session_id>"`) when `GOOSE_DATABRICKS_CLIENT_REQUEST_ID` is explicitly `true` (`crates/goose/src/providers/databricks_def.rs:71,95-100`; `crates/oi-kit-providers/src/databricks.rs:532-539`).

3. **Kimi Code Provider `device_id` (`kimicode/device_id`)**:
   - Path: `Paths::in_config_dir("kimicode/device_id")` (`crates/goose/src/providers/kimicode.rs:197-198`).
   - Generation: Random UUID v4 stripped of hyphens (`crates/goose/src/providers/kimicode.rs:206`).
   - Only created when `kimi_code` provider is configured/instantiated and sent exclusively to `https://api.kimi.com/coding` / `https://auth.kimi.com` as `X-Msh-Device-Id` (`crates/goose/src/providers/kimicode.rs:52-53,214-226`).

4. **P2P Roaming Node Ed25519 Key (`roaming_node_key`)**:
   - Path: Config dir `roaming_node_key` (`0600` hex-encoded Ed25519 keypair, `crates/goose-roaming/src/identity.rs:7-23,46-88`). Only created/used if user runs `goose roam`.

5. **Hashing / Hardware Fingerprinting**:
   - **No hardware, MAC, CPU, disk, or OS machine-ID hashing** (e.g., `/etc/machine-id`, IOPlatformUUID, MachineGuid) exists anywhere in Goose.
   - `ProviderInventoryService` uses `Sha256` strictly as an internal local SQLite cache key over provider config (`crates/goose/src/providers/inventory/mod.rs:130-147`) — never transmitted externally.
   - No cross-provider unsalted PII/identity stitching occurs (`installation_id` is isolated to PostHog when opted in).

---

## 4. Exact Outbound Telemetry / Analytics / Crash / OTEL / Relay / Update Endpoints

| Subsystem / Channel | Exact Endpoint / URL / Host | Trigger Condition | Runtime Opt-Out? |
|---|---|---|---|
| **PostHog Capture API** | `https://us.i.posthog.com/capture/` (`crates/goose/src/posthog.rs:17`), hardcoded project key `phc_RyX5CaY01VtZJCQyhSR5KFh6qimUy81YwxsEpotAftT` (`crates/goose/src/posthog.rs:16`) | Fires `session_started` on `SessionManager::create_session` (`session_manager.rs:1911-1912`) **when opted in** (`is_telemetry_enabled() == true`). (`emit_error`, `emit_custom_slash_command_used`, and UI `analytics.ts` are currently disabled.) | Yes (`GOOSE_TELEMETRY_OFF=1`/`true`, `GOOSE_TELEMETRY_ENABLED=false`, UI/CLI toggle — except dormant `emit_event` bypass at `posthog.rs:560-568`). |
| **Models.dev Canonical Catalog** | `https://models.dev/api.json` (`crates/goose/src/model_catalog.rs:7`, `crates/oi-kit-provider-types/src/canonical/registry.rs:149-160`) | **Unconditional background GET at every startup** of `goose` CLI, `goose serve` (spawned by Desktop), and `goose-acp` (`main.rs:24-25`, `goose-acp.rs:33-34`, `build-acp.yml:89`). Sends `User-Agent: goose/model-catalog (https://github.com/aaif-goose/goose)` and cached `If-None-Match` ETag. | **No runtime opt-out** (only compile-time `--no-default-features` omitting `online-model-meta`, as in musl builds). |
| **Desktop Auto-Updater & GitHub API Fallback** | GitHub Releases (`electron-updater` provider `github`, `owner: aaif-goose`, `repo: goose` via `ui/desktop/src/app-update.yml:1-6`, `ui/desktop/src/utils/autoUpdater.ts:353-364`) + fallback `https://api.github.com/repos/aaif-goose/goose/releases/latest` + macOS `mac-update-requirements.json` asset (`ui/desktop/src/utils/githubUpdater.ts:502-506,573-581`) | **Unconditional check 7 seconds after every Desktop launch** (`2000ms` delay in `ui/desktop/src/main.ts:2498-2511` + `5000ms` delay in `ui/desktop/src/utils/autoUpdater.ts:414-533`). `githubUpdater.ts:526` sends `User-Agent: Goose-Desktop/<version>`. | **Check cannot be disabled** at runtime (`UPDATES_ENABLED = true` at `updates.ts:1`); only **auto-download** can be disabled via `GOOSE_DISABLE_AUTO_DOWNLOAD=1`/`true` or Settings `disableAutoDownload` (`autoUpdater.ts:49-53,374-384`). |
| **CLI Manual Self-Update** | `https://github.com/aaif-goose/goose/releases/download/<tag>/<asset>`, `https://api.github.com/repos/aaif-goose/goose/attestations/sha256:<digest>...`, and GitHub blob storage `bundle_url` (`crates/goose-cli/src/commands/update.rs:129-189,323-332`) | **Only when user explicitly runs `goose update`** (`User-Agent: goose-cli`). | Yes (manual command only; compile-time `disable-update` flag at `crates/goose-cli/Cargo.toml:145`). |
| **OpenTelemetry (OTLP Traces / Metrics / Logs)** | **User-configured only** (`OTEL_EXPORTER_OTLP_ENDPOINT`, `OTEL_EXPORTER_OTLP_{SIGNAL}_ENDPOINT`, or `otel_exporter_otlp_endpoint` in `config.yaml`, appending `/v1/traces` etc.; `crates/goose/src/otel/otlp.rs:227-272`). No hardcoded SaaS collector. | Disabled (`None`) unless user sets endpoint or `OTEL_{SIGNAL}_EXPORTER`. | Off by default; force off with `OTEL_SDK_DISABLED=true`. |
| **Langfuse Tracing Layer** | `LANGFUSE_URL` / `LANGFUSE_BASE_URL` / `LANGFUSE_HOST` (fallback `http://localhost:3000`) + `/api/public/ingestion` (`crates/goose/src/tracing/langfuse_layer.rs:13,81-90,169-172`) | Disabled unless **both** `LANGFUSE_PUBLIC_KEY` (or `LANGFUSE_INIT_PROJECT_PUBLIC_KEY`) and `LANGFUSE_SECRET_KEY` (or `LANGFUSE_INIT_PROJECT_SECRET_KEY`) are non-empty (`langfuse_layer.rs:155-167`). | Off by default (unset keys). |
| **Goose Roam P2P Relays (iroh)** | Default managed relays (`https://usw1-2.relay.michaelneale.mesh-llm.iroh.link./`, `use1-1...`, `euc1-1...`, `aps1-1...`) or user `GOOSE_ROAM_RELAYS` (`crates/goose-cli/src/commands/roam.rs:51-71`) | **Only when user explicitly runs `goose roam`** subcommands (`share`, `connect`, `delegate`, `bridge` per `roam.rs:11-16`; `node.rs:139-160`). | Off unless `goose roam` is run; overrideable with `GOOSE_ROAM_RELAYS` / `RelaySettings::Disabled`. |
| **Optional ML Prompt-Injection Classifier** | User-configured `SECURITY_PROMPT_CLASSIFIER_ENDPOINT` (or `SECURITY_ML_MODEL_MAPPING`; `crates/goose/src/security/scanner.rs:90-110`) | Only when `SECURITY_PROMPT_CLASSIFIER_ENABLED=true` (defaults `false` at `scanner.rs:81-83`). | Off by default. |
| **Docs Website (`goose-docs.ai`, non-binary)** | Google Analytics `gtag` tracking ID `G-ZS5D6SB4ZJ` (`anonymizeIP: true`, `documentation/docusaurus.config.ts:142-148`) | Website visitors only (`NODE_ENV === "production"`); not bundled in CLI or Desktop binaries. Builtin `goose-doc-guide` skill points agent docs lookups at `GOOSE_DOCS_ROOT` defaulting to `https://goose-docs.ai` (`crates/goose/src/skills/mod.rs:107,122-125`). | Configurable via `GOOSE_DOCS_ROOT` for offline docs. |
| **Other Repo Services (non-client)** | `oidc-proxy/` (Cloudflare Worker auth/rate-limit proxy, `oidc-proxy/src/index.js:1-60`) and `services/ask-ai-bot/` (Discord bot, `services/ask-ai-bot/package.json:1-28`) | Server/community bots only; not shipped in Goose CLI or Desktop clients. | N/A |

---

## 5. Exact Data & Properties Collected

### A. PostHog `session_started` Payload (when opted in, `crates/goose/src/posthog.rs:389-451`)
Sent to `https://us.i.posthog.com/capture/` (`CaptureEvent` at `posthog.rs:58-78`) with `distinct_id = installation.installation_id` (UUID v4) and `timestamp = Utc::now().to_rfc3339()`:
- **System / Environment**: `os` (`std::env::consts::OS`), `arch` (`std::env::consts::ARCH`), `version` (`CARGO_PKG_VERSION`), `is_dev` (`cfg!(debug_assertions)`), `platform_version` (from `sw_vers -productVersion` / `/etc/os-release` `VERSION_ID=` / `cmd /C ver` at `posthog.rs:154-193`), `install_method` (`"homebrew"`, `"cargo"`, `"desktop"`, or `"binary"`, `posthog.rs:195-217`).
- **Session / Behavioral Profile**: `interface` (`"cli"` from `builder.rs:666`; `"unknown"` from Desktop `goose serve` because `set_session_context` is not called there and `GOOSE_DESKTOP` is unset), `is_resumed`, `session_number` (lifetime counter in `telemetry_installation.json`), `days_since_install`, `setting_mode` (`GOOSE_MODE`), `setting_max_turns` (`GOOSE_MAX_TURNS`), `db_schema_version` (`CURRENT_SCHEMA_VERSION`), `total_sessions`, and `total_tokens` (aggregated across all local SQLite sessions via `SessionManager::get_insights()`, `crates/goose/src/session/session_manager.rs:2636-2667`).
- **Provider / Model / Extensions (Verbatim, Unsanitized)**:
  - `provider` (`config.get_goose_provider()`, `posthog.rs:410-412`) and **`model` (`config.get_goose_model()`, `posthog.rs:413-415`)** — sent **verbatim** without canonicalization, redaction, or `sanitize_value()`, exposing custom/self-hosted model IDs and custom declarative provider names.
  - `extensions_count` and **`extensions` (full JSON array of enabled extension names, `posthog.rs:424-436`)** — exposes custom/internal MCP server names enabled in config.
- **IP / GeoIP Handling**: **No IP suppression.** `posthog_capture()` (`posthog.rs:67-90`) does **not** set `"$ip": null` or `"disable_geoip": true` in `properties` (and has no server proxy stripping IPs), so PostHog ingests client IP address and resolves GeoIP (`$geoip_*`) on every event.
- **Prompts / Code / Paths / Stack Traces in PostHog**: `session_started` does **not** include prompts, code, file paths, or stack traces. `emit_error_with_context()` (which previously sent `sanitize_string(error_message)` via regex patterns at `posthog.rs:513-536` — notably **without** stripping full file paths outside `/Users/`, `/home/`, `C:\Users\` or stack traces) and `emit_event()`'s UI error path are currently **disabled** (`posthog.rs:299-301`; `analytics.ts:12-17`).

### B. Startup `models.dev` & Desktop Update Requests (Always-On)
- **`models.dev` (`registry.rs:152-158`)**: Standard HTTP GET headers (`User-Agent: goose/model-catalog (https://github.com/aaif-goose/goose)` and optional cached `If-None-Match` ETag) + client IP visibility to `models.dev`. No install ID or local telemetry payload attached.
- **Desktop Updater (`githubUpdater.ts:523-529`, `autoUpdater.ts`)**: `User-Agent: Goose-Desktop/<version>`, client IP visibility to GitHub / Fastly CDN. No machine ID sent.

### C. User-Configured OTLP & Langfuse Export
- **OTLP Resource & Log Metadata (`crates/goose/src/otel/otlp.rs:274-296,449-456`)**: Unconditionally includes `host.name = session_host()` (`gethostname`), `user.name = session_user()` (`USER`/`LOGNAME` env vars, `crates/goose/src/session_context.rs:74-84`), `EnvResourceDetector`, `TelemetryResourceDetector`, `service.version`, and span log attributes `session.id`, `session.user`, `session.host`, `session.agent_type`.
- **Prompts / Replies / Tool Args & Results (`crates/goose/src/agents/gen_ai_telemetry.rs:9-14,91-111`)**: Strictly gated behind `OTEL_INSTRUMENTATION_GENAI_CAPTURE_MESSAGE_CONTENT=true` (verified across `services.rs:1009-1016`, `state_machine/session.rs:270-285,329-334`, `reply_parts.rs:299-304`, `ops_toolcalling.rs:128,304`, and `state_machine/tests/mod.rs:82-93`). MCP `rmcp::service` spans are explicitly suppressed from OTLP export to avoid leaking ~400KB+ `InitializeResult` instructions/memory PII (`crates/goose/src/otel/otlp.rs:547-562,576-578`).
- **Langfuse (`observation_layer.rs`, `langfuse_layer.rs`)**: Exports all tracing spans/events at `>= DEBUG` (`crates/goose/src/logging.rs:103-105`), which can include span metadata if `OTEL_INSTRUMENTATION_GENAI_CAPTURE_MESSAGE_CONTENT=true` or debug logs are enabled.

---

## 6. Docs / Privacy Policy vs Code Discrepancies

1. **Broken Telemetry Disable Env Var in `CUSTOM_DISTROS.md`**:
   - `CUSTOM_DISTROS.md:93` states: `Disable telemetry: Set GOOSE_DISABLE_TELEMETRY=1`.
   - **Code reality**: `GOOSE_DISABLE_TELEMETRY` is **not read anywhere in the codebase** (`0` code hits). The actual working env vars are `GOOSE_TELEMETRY_OFF=1`/`true` (`crates/goose/src/posthog.rs:23-25`) or `GOOSE_TELEMETRY_ENABLED=false` (`crates/goose/src/posthog.rs:20,39`). A downstream distribution following `CUSTOM_DISTROS.md` fails to disable telemetry.

2. **Misleading `Anonymous usage data` Switch Default in Desktop Settings**:
   - Docs (`documentation/docs/guides/config-files.md:75`, `documentation/docs/guides/environment-variables.md:320`) state `GOOSE_TELEMETRY_ENABLED` defaults to `false`, and backend `is_telemetry_enabled()` returns `false` when unset (`crates/goose/src/posthog.rs:50-52`).
   - **Code reality**: `ui/desktop/src/components/settings/app/TelemetrySettings.tsx:54,61` sets the switch state to `true` when `read('GOOSE_TELEMETRY_ENABLED')` returns `null`, displaying telemetry as **enabled** before the user has opted in.

3. **Inaccurate Disclosure of Collected Telemetry Properties (`usage-data.md`, `PrivacyInfoModal.tsx`, `configure.rs`)**:
   - `documentation/docs/guides/usage-data.md:20-22`, `ui/desktop/src/components/onboarding/PrivacyInfoModal.tsx:31-39`, and `crates/goose-cli/src/commands/configure.rs:189-197` all tell users Goose collects:
     - `"Extensions and tool usage counts (names only)"` — **False**: `posthog.rs:425-436` collects the enabled `extensions` name list and `extensions_count`, **not** tool usage counts, while tool/slash-command counting (`emit_custom_slash_command_used`) is disabled.
     - `"Session metrics (duration, interaction count, token usage)"` — **Inaccurate**: `posthog.rs:403-448` sends `session_number`, `days_since_install`, `is_resumed`, lifetime SQLite `total_sessions` and `total_tokens`, `setting_mode`, `setting_max_turns`, and `db_schema_version` — **not** per-session duration or interaction count.
     - `"Error types (e.g., \"rate_limit\", \"auth\" - no details)"` — **Stale**: `emit_error_with_context()` is hard-disabled (`posthog.rs:299-301`) and frontend `analytics.ts` is a no-op.
   - **Undisclosed items**: None of `usage-data.md`, `PrivacyInfoModal.tsx`, or `configure.rs` disclose the persistent UUIDv4 `installation_id` stored in `telemetry_installation.json`, PostHog's receipt of client IP / GeoIP, verbatim `model` string transmission (including custom/private models), the unconditional startup `models.dev` request, or Desktop's unconditional startup GitHub update check.

4. **Stale OTEL Legacy Field Warning in `environment-variables.md`**:
   - `documentation/docs/guides/environment-variables.md:455` warns that `user_message`, `trace_input`, `trace_output`, and `input` can still be exported when `OTEL_INSTRUMENTATION_GENAI_CAPTURE_MESSAGE_CONTENT` is unset/false, contradicting line `457` and current Rust code (`services.rs:1009-1016`, `state_machine/session.rs:270-285,329-334`, `gen_ai_telemetry.rs:91-111`, verified by unit test `state_machine_trace_omits_content_without_capture` at `state_machine/tests/mod.rs:82-93`).

5. **Minor Docs Site Analytics Note**:
   - `documentation/docs/guides/usage-data.md` covers only Goose app telemetry and omits that the hosted docs site uses Google Analytics `gtag` (`G-ZS5D6SB4ZJ`, `anonymizeIP: true` at `documentation/docusaurus.config.ts:142-148`), which is normal web analytics outside the client binaries.

---

## 7. Zero-Telemetry / Negative-Check Verification

Read-only `rg` sweeps across `crates/`, `ui/`, `.github/`, `services/`, `oidc-proxy/`, and `Cargo.toml` (excluding lockfiles, generated files, and vendored/minified assets) confirmed the **absence** of any undisclosed crash reporters, ad/marketing trackers, or hidden SDKs:

- **Third-Party Analytics / Crash Reporters (`rg -n -i -w "sentry|crashReporter|amplitude|mixpanel|statsig|launchdarkly|bugsnag|rollbar|logrocket|datadog|honeycomb|newrelic|plausible|fathom|umami|clarity|gtag|posthog"`)**:
  - **Zero** occurrences in client source (`crates/`, `ui/desktop/`) of Sentry, Electron `crashReporter`, Amplitude, Mixpanel, Statsig, LaunchDarkly, Bugsnag, Rollbar, LogRocket, Honeycomb, NewRelic, Plausible, Fathom, Umami, Clarity, Segment (`rg "segment\.io"`), or Google Analytics (`gtag` appears only in `documentation/docusaurus.config.ts:142` and `documentation/src/theme/Root.tsx:11-14`; `datadog` appears only in a doc comment at `crates/goose/src/otel/otlp.rs:806`).
  - Only `crates/goose/src/posthog.rs` (custom `reqwest` client to `https://us.i.posthog.com/capture/`) and user-configured OTLP (`crates/goose/src/otel/otlp.rs`) / Langfuse (`crates/goose/src/tracing/langfuse_layer.rs`) exist as telemetry/observability exporters.
- **CI Secret Injection Check (`.github/workflows/release.yml`, `canary.yml`, `build-cli-linux.yml`, `build-acp.yml`)**:
  - **Zero** analytics API keys or telemetry secrets are injected at build time (`POSTHOG_API_KEY` is checked into `crates/goose/src/posthog.rs:16` in plaintext).
- **`DO_NOT_TRACK` / `GOOSE_DISABLE_TELEMETRY` (`rg -n "DO_NOT_TRACK|GOOSE_DISABLE_TELEMETRY|GOOSE_TELEMETRY_OFF|GOOSE_TELEMETRY_ENABLED"`)**:
  - Confirmed `DO_NOT_TRACK` has **zero** hits repo-wide; `GOOSE_DISABLE_TELEMETRY` appears **only** in `CUSTOM_DISTROS.md:93`.
- **Conclusion / Rating**: **C (Moderate / Mixed)** — Because backend PostHog `session_started` telemetry defaults to `false` when unset, Desktop/UI `analytics.ts` is currently stubbed during the ACP migration (leaving the onboarding bypass dormant), and Goose embeds zero third-party crash SDKs, Goose rises to **C** rather than **D**, while being held back from **B** because: (1) default CLI/Desktop/ACP binaries perform unconditional startup network calls (`https://models.dev/api.json` and Desktop GitHub release update checks) with **no runtime opt-out**; (2) Desktop Settings displays the privacy switch **ON** when unset while onboarding/CLI prompts default to `true`; (3) `CUSTOM_DISTROS.md` documents a broken `GOOSE_DISABLE_TELEMETRY=1` flag and `DO_NOT_TRACK` is ignored; (4) `posthog::emit_event` retains an explicit `GOOSE_TELEMETRY_OFF`/consent bypass creating persistent UUIDv4 `installation_id`s for onboarding events (dormant while UI is stubbed); (5) opted-in PostHog lacks IP/GeoIP suppression and sends custom model/provider/extension names verbatim; and (6) `usage-data.md` and consent modals inaccurately describe collected data.

```json
{
  "slug": "goose",
  "platform": "Goose",
  "group": "B",
  "repo": "aaif-goose/goose",
  "sha": "766923df7a16679ee430eac43c9fc9c73caf4b33",
  "commit_date": "2026-11-25T04:38:40Z",
  "license": "Apache-2.0",
  "status": "Active (block/goose redirects here)",
  "default_state": "Mixed: Backend PostHog session_started is OFF until opt-in (false when unset) and frontend analytics.ts is stubbed to no-op; OTLP and Langfuse are OFF unless configured; HOWEVER default binaries unconditionally phone https://models.dev/api.json on every startup (online-model-meta) and Desktop unconditionally checks GitHub Releases on every launch (UPDATES_ENABLED=true), while CLI/onboarding consent prompts default to true and Desktop TelemetrySettings renders the toggle ON when unset.",
  "opt_out_controls": [
    "GOOSE_TELEMETRY_OFF=1 or true (env var overriding config for PostHog session_started)",
    "GOOSE_TELEMETRY_ENABLED=false (config.yaml or env var)",
    "CLI: goose configure -> goose settings -> Telemetry",
    "Desktop UI: Settings -> App -> Privacy -> Anonymous usage data",
    "OTEL_SDK_DISABLED=true (disables user-configured OTLP)",
    "GOOSE_DISABLE_AUTO_DOWNLOAD=1/true or disableAutoDownload setting (stops update download only, NOT update check)",
    "No runtime opt-out for startup models.dev catalog fetch or Desktop startup update check (compile-time flags only: omit online-model-meta / disable-update)"
  ],
  "do_not_track": false,
  "opt_out_bypasses": [
    "posthog::emit_event() explicitly bypasses is_telemetry_enabled() and GOOSE_TELEMETRY_OFF and creates telemetry_installation.json for onboarding_* and telemetry_preference_set events (currently dormant because UI analytics.ts sendEvent is stubbed to no-op)",
    "Startup https://models.dev/api.json fetch runs unconditionally in default CLI/Desktop/goose-acp builds regardless of GOOSE_TELEMETRY_OFF or GOOSE_TELEMETRY_ENABLED=false",
    "Desktop startup update check runs unconditionally regardless of telemetry opt-out settings or GOOSE_DISABLE_AUTO_DOWNLOAD"
  ],
  "persistent_ids": [
    "PostHog installation_id: UUIDv4 in <state_dir>/telemetry_installation.json (created when opted in or if emit_event onboarding fires)",
    "Databricks instance_id: UUIDv4 in <state_dir>/instance_id (only read/sent if GOOSE_DATABRICKS_CLIENT_REQUEST_ID=true, default false)",
    "Kimi Code device_id: hyphenless UUIDv4 in <config_dir>/kimicode/device_id (only if kimi_code provider used)",
    "Roaming Ed25519 node key: <config_dir>/roaming_node_key (only if goose roam used)"
  ],
  "hashing": "None for device/user IDs (random UUIDv4); Sha256 used only locally for ProviderInventoryService SQLite cache key",
  "external_auth_reads": "No silent tracking harvest. Explicit/conditional reads only: ~/.config/Claude or Claude Desktop claude_desktop_config.json via user-triggered onboarding import; Claude Code/Codex/Pi session JSONL via explicit import_session; GCP gcloud ADC / AWS / Azure az CLI only when those cloud providers are configured; ~/.claude/agents and ~/.claude/skills for extension/skill discovery.",
  "telemetry_endpoints": [
    "https://us.i.posthog.com/capture/ (PostHog analytics, opt-in session_started)",
    "https://models.dev/api.json (unconditional startup catalog fetch in default builds)",
    "https://github.com/aaif-goose/goose/releases & https://api.github.com/repos/aaif-goose/goose/releases/latest (unconditional Desktop startup update check + manual CLI goose update)",
    "User-configured OTEL_EXPORTER_OTLP_ENDPOINT / otel_exporter_otlp_endpoint (OTLP traces/metrics/logs, off by default)",
    "User-configured LANGFUSE_URL (default http://localhost:3000/api/public/ingestion when keys set, off by default)",
    "Goose Roam managed iroh relays (*.relay.michaelneale.mesh-llm.iroh.link, only when user runs goose roam)"
  ],
  "ip_geo_handling": "No IP suppression on PostHog ($ip: null / disable_geoip absent in posthog.rs:posthog_capture); client IP exposed to us.i.posthog.com (when opted in), models.dev, and GitHub Releases.",
  "custom_model_handling": "Verbatim: config.get_goose_provider(), config.get_goose_model(), and full enabled extensions array are sent raw in PostHog session_started without canonicalization or sanitization.",
  "docs_discrepancies": [
    "CUSTOM_DISTROS.md:93 documents non-existent GOOSE_DISABLE_TELEMETRY=1 (ignored by code; real env var is GOOSE_TELEMETRY_OFF=1)",
    "TelemetrySettings.tsx:54,61 renders UI switch ON when GOOSE_TELEMETRY_ENABLED is unset, contradicting docs and backend false default",
    "usage-data.md, PrivacyInfoModal.tsx, and configure.rs claim collection of tool usage counts, session duration, interaction count, and error types, whereas posthog.rs sends lifetime total_sessions/total_tokens/days_since_install/extensions array/setting_mode/setting_max_turns and has disabled error tracking, while omitting installation_id, IP visibility, models.dev startup fetch, and Desktop update check",
    "environment-variables.md:455 warns legacy OTEL prompt fields bypass OTEL_INSTRUMENTATION_GENAI_CAPTURE_MESSAGE_CONTENT, contradicting line 457 and current Rust code"
  ],
  "privacy_rating": "C",
  "one_line_verdict": "Opt-in PostHog backend (session_started defaults false when unset), dormant stubbed UI analytics during ACP migration, and zero third-party crash SDKs earn C, held below B by unconditional startup calls to models.dev and GitHub update checks with no runtime opt-out, misleading default-ON UI/CLI consent nudges, a broken GOOSE_DISABLE_TELEMETRY=1 doc flag, missing PostHog IP suppression, verbatim custom model reporting, a dormant onboarding consent bypass, and inaccurate data disclosures."
}
```
