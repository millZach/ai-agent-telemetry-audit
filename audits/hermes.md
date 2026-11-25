# Hermes (`NousResearch/hermes-agent`) — Forensic Telemetry & Privacy Audit

- **slug:** `hermes`
- **platform:** `Hermes`
- **group:** `B`
- **repo:** `https://github.com/NousResearch/hermes-agent` (`NousResearch/hermes-agent`)
- **license:** `MIT` (`pyproject.toml:17`)
- **status:** `Active`
- **branch:** `main`
- **sha:** `a39b00bf378f0a9ca6e8e8ba27af4ca4bf8cd86d`
- **commit_date:** `2026-11-24T23:05:06-06:00`
- **commit_subject:** `test(desktop): restore the Math.random pin in afterEach`

---

## 1. Default State & Opt-Out Mechanism

**Default telemetry state is OFF (Opt-In across all telemetry and analytics subsystems):**

1. **NeMo Relay Shared Metrics (`telemetry.shared_metrics`):**
   - Both `telemetry.shared_metrics.enabled` (local SQLite collection) and `telemetry.shared_metrics.send` (daily upload to Nous) default to `False` (`hermes_cli/config_defaults.py:2276-2285`, `cli-config.yaml.example:2264-2268`).
   - Collection and sending are separately gated: `resolve_send_config()` enforces `enabled = shared.get("enabled") is True` and `send_requested = shared.get("send") is True`. If `send: true` without `enabled: true`, it logs an error and returns `SendConfig(enabled=False, send=False, ...)` (`hermes_cli/observability/shared_metrics_send_config.py:61-74`).
   - When `telemetry.shared_metrics` is disabled, `relay_shared_metrics.enabled()` returns `False` and deactivates any active runtime (`hermes_cli/observability/relay_shared_metrics.py:1375-1384`), and `_reconcile_send_consent_once()` explicitly skips creating `$HERMES_HOME/telemetry/shared_metrics/metrics.sqlite3` when `send` is `False` and no store exists yet (`hermes_cli/observability/relay_shared_metrics.py:1393-1416`).
   - First-run consent prompts ask once per profile via:
     - `hermes setup` (`hermes_cli/setup.py:531-560`, `hermes_cli/observability/shared_metrics_consent.py:12-15`)
     - Interactive CLI/TUI chat (`hermes_cli/observability/shared_metrics_consent.py:28-32,150-190`, where `OFFER_CHOICES` has `"Share with Nous"` and `"No thanks"`, with `"No thanks"` as terminal default; skipped for `-q`/`--oneshot`/piped/`HERMES_DESKTOP`/managed installs)
     - Hermes Desktop non-blocking composer strip & details modal (`apps/desktop/src/components/shared-metrics/consent-strip.tsx:20-104`, `apps/desktop/src/components/shared-metrics/consent-dialog.tsx:44-150`, `apps/desktop/src/app/chat/composer/status-stack/index.tsx:239-241`)
     - Web dashboard banner (`web/src/components/SharedMetricsConsentBanner.tsx:18-113`, backed by `/api/shared-metrics/consent` in `hermes_cli/web_routers/shared_metrics.py:34-48`).
   - **Pre-consent / re-prompt nuances (minor caveats, no silent transmission bypass):**
     - **One-time `OFFER_VERSION = 2` re-ask:** Profiles that previously wrote `enabled: false` without `offer_version: 2` (prior to a terminal type-ahead fix) read `decided = False, reask = True` and are prompted once more across CLI/Desktop/Web (`hermes_cli/observability/shared_metrics_consent.py:53-71`, `apps/desktop/src/i18n/en_shared_metrics.ts:27`, `web/src/components/SharedMetricsConsentBanner.tsx:88-90`).
     - **Pre-consent installer receipt:** Full runs of `scripts/install.sh:141-153` and `scripts/install.ps1:843-860` write a bounded local JSON receipt (`$HERMES_HOME/telemetry/shared_metrics/pending_installs/<id>.json`) before consent is asked. On the next Hermes start, `shared_metrics_process.begin_process()` deletes `pending_installs/`, `pending_updates/`, `process_markers/`, `provider_setup_markers/`, and `desktop_onboarding/` unreported whenever `_collection_on()` is `False` (`hermes_cli/observability/shared_metrics_process.py:129-147`; 7-day max receipt age in `hermes_cli/observability/shared_metrics_install_run.py:40`). Even if the user opts in during setup on install day, `consented_day()` delays recording until a subsequent UTC day inside an active consent window (`hermes_cli/observability/shared_metrics_install_run.py:105-131`).
     - **Desktop in-memory pre-consent hold:** Before first-run consent is answered, `apps/desktop/src/store/desktop-metrics.ts:15-17,853-863,1116-1119,1214-1224` holds `recordOnboarding()` transitions only in an in-memory `preConsent` array (never persisted to `localStorage` or sent over RPC). Answering `"Share with Nous"` replays them; answering `"No thanks"` (`decided=true`) clears `preConsent = null` and drops them. All other Desktop `recordFeatureUse`, `recordFriction`, `recordDislike`, `recordAction`, `recordLongFrames` calls are gated on `$desktopMetricsGate.get() === 'on'` (`apps/desktop/src/store/desktop-metrics.ts:530-571,598-600,668-671,779,825-829`).
     - **Fail-closed consent windows (`send_consent_windows`):** Packages are only transmitted when the package's entire `[period_start, period_end]` falls inside an open/confirmed window in `send_consent_windows` (`hermes_cli/observability/shared_metrics_sender.py:186-203`).

2. **Gateway Health Monitoring / OTLP (`monitoring.gateway_health_export`, `monitoring.export.otlp`):**
   - Disabled by default (`enabled: False`, `endpoint: ""`, `hermes_cli/config_defaults.py:2025-2043`).
   - Ships **no default OTLP endpoint**; `otlp_exporter.is_enabled()` requires both `otlp.enabled` and non-empty `otlp.endpoint` (`agent/monitoring/otlp_exporter.py:3-4,257-259`).
   - OpenTelemetry SDK packages (`opentelemetry-sdk==1.45.2`, `opentelemetry-exporter-otlp-proto-http==1.45.2`) live strictly in the optional `[otlp]` extra (`pyproject.toml:462-469`), never in core dependencies.

3. **Third-Party / Subprocess / Sidecar Telemetry:**
   - **Computer Use (`cua-driver`):** Upstream PostHog telemetry defaults ON in `cua-driver`; Hermes explicitly forces `CUA_DRIVER_RS_TELEMETRY_ENABLED=0` in every child environment unless `computer_use.cua_telemetry: true` is opted in (`hermes_cli/config_defaults.py:2488-2491`, `tools/computer_use/cua_backend.py:33-34,68-70,154-155`).
   - **Photon iMessage Sidecar (`PHOTON_TELEMETRY`):** Optional plugin Spectrum SDK telemetry is OFF by default (`plugins/platforms/photon/sidecar/index.mjs:62-63,82-84,337`), toggleable via `hermes photon telemetry on|off` (`plugins/platforms/photon/cli.py:331-352`).
   - **Electron Linux Crash Reporter:** Starts `crashReporter` strictly local-only with `uploadToServer: false, compress: false` (`apps/desktop/electron/linux-crash-diagnostics.ts:15-16,39,95`, `apps/desktop/electron/main.ts:1379-1383`).
   - **NeMo Relay OTLP Auto-Export:** Disabled by default unless the user supplies `plugins.toml` or `OTEL_EXPORTER_OTLP_*_ENDPOINT` env vars (`agent/relay_runtime.py:291-320`, `website/docs/developer-guide/relay-shared-metrics.md:47-58`).

4. **Opt-Out Controls & `DO_NOT_TRACK`:**
   - **Opt-out mechanisms:**
     - Wizard / CLI: `hermes setup telemetry` (`hermes_cli/setup.py:531-560`) or choosing `"No thanks"` at setup/REPL/TUI offers.
     - Config keys: `hermes config set telemetry.shared_metrics.enabled false` and `telemetry.shared_metrics.send false` (or manual edit in `$HERMES_HOME/config.yaml`). Note `shared_metrics.endpoint` is deliberately **not** env-overridable (`hermes_cli/observability/shared_metrics_send_config.py:11-15`), and `shared_metrics` flags are profile-owned (cannot be force-enabled by managed scope, `website/docs/developer-guide/relay-shared-metrics.md:80-82`).
     - Desktop UI: Composer strip `"No thanks"` or **Settings → Safety → Privacy** toggle `"Share usage stats with Nous"` (`apps/desktop/src/app/settings/shared-metrics-settings.tsx:22-127`, `apps/desktop/src/i18n/en_shared_metrics.ts:22-24`), which calls `shared_metrics.set`, revokes Web Locks/IndexedDB frame cohorts, purges `localStorage` (`hermes.desktop.metrics.v1:*`), clears `desktop_onboarding/` latches and `pending_*` receipts, and clears Electron `RendererCrashRecorder` entries (`tui_gateway/methods_shared_metrics.py:55-94`, `apps/desktop/src/store/desktop-metrics.ts:1116-1160`, `apps/desktop/electron/renderer-crash-metrics.ts:69-81`).
     - Web Dashboard: `"No thanks"` button in `SharedMetricsConsentBanner.tsx:97-102` (`PUT /api/shared-metrics/consent`, `hermes_cli/web_routers/shared_metrics.py:39-48`).
   - **`DO_NOT_TRACK`:** **NOT honored** by core Hermes (`shared_metrics_send_config.py`, `shared_metrics_consent.py`, `relay_shared_metrics.py`, Desktop, Web). A repo-wide `git grep "DO_NOT_TRACK"` matches only third-party catalog plugin descriptions (`plugin-catalog/renor.yaml:6`, `plugin-catalog/telesec.yaml:5`).

5. **Default-On Non-Telemetry Outbound Features (Operational / Trial / Updates):**
   - Although telemetry is OFF by default, three **non-telemetry** outbound behaviors run by default unless disabled/already configured:
     1. **Passive update checks (`updates.check: true`):** CLI background prefetch (`hermes_cli/banner.py:275-288`, `hermes_cli/source_check.py:588-589`) and Desktop startup/daily update poller (`apps/desktop/src/store/updates.ts:1385-1415`).
     2. **Metadata catalogs:** `https://models.dev/api.json` when disk cache is absent or older than 4 hours (`agent/models_dev.py:27-28,492-547`), plus OpenRouter/Nous `/v1/models` reasoning-capability catalog fetches (`hermes_cli/models_reasoning_caps.py:272-315`).
     3. **Desktop first-run Welcome Trial minting:** When running the local Desktop backend on a fresh machine with no configured provider and no Nous identity, clicking start in onboarding (`onboarding.start` -> `_claim_before_run`) automatically calls `POST https://portal.nousresearch.com/api/anonymous/create` and `/api/anonymous/token` to mint an anonymous free-tier guest (`tui_gateway/methods_onboarding.py:150-199`, `hermes_cli/welcome_trial.py:114-140`, `hermes_cli/anon_auth.py:381-434`).

---

## 2. External Auth / Local Non-Workspace File Reads

Hermes reads several external tool authentication and session stores under `~` / `%USERPROFILE%`. **Critically, none of these external reads are used for telemetry, analytics, or cross-provider telemetry ID stitching** — they exist strictly to borrow API credentials for LLM/GitHub requests or to let users import prior conversations/setups. However, `auth.adopt_external_logins` defaults to **`true`**:

| External File / Store | Purpose | Default / Gating Condition | Source Citations |
|---|---|---|---|
| `~/.claude/.credentials.json` (or `$CLAUDE_CONFIG_DIR/.credentials.json`) & macOS Keychain `"Claude Code-credentials"` | Borrow & write back rotated Claude Code OAuth tokens for Anthropic inference | **Enabled by default** (`auth.adopt_external_logins: true`). Gated inside `read_claude_code_credentials()` on `adopt_external_logins_enabled()`. Also probed by `_has_any_provider_configured()` when a non-default model is set (`not strict_profile_scope`) | `hermes_cli/config_defaults.py:1704`, `agent/credential_sources.py:32-49`, `agent/anthropic_credentials.py:261-278,339-376,543-615`, `hermes_cli/main.py:1013-1023` |
| `~/.codex/auth.json` (or `$CODEX_HOME/auth.json`) | Recover expired/spent `openai-codex` refresh tokens or interactive `hermes auth add openai-codex` import | **Enabled by default** for automatic recovery (`_recover_codex_tokens_from_cli` checks `adopt_external_logins_enabled()`; interactive `hermes auth add` asks regardless of setting). `read_only=True` status/doctor skips adoption | `hermes_cli/auth_codex.py:237-276,587-618`, `agent/credential_pool.py:2626-2633` |
| `~/.qwen/oauth_creds.json` | Read & write refreshed Qwen CLI OAuth credentials for `qwen-oauth` | **Bypass caveat:** `resolve_qwen_runtime_credentials()` and `_seed_qwen_singleton()` read `~/.qwen/oauth_creds.json` **without checking `adopt_external_logins_enabled()`** when `qwen-oauth` pool/status/runtime is invoked (though `resolve_provider("auto")` does not scan it unless configured in `auth.json` / `config.yaml`) | `hermes_cli/auth_qwen.py:25-49,118-152`, `agent/credential_pool.py:2579-2597,2663-2664` |
| `gh auth token` (GitHub CLI subprocess) | Authenticate `api.github.com` requests during passive update checks (`source_check`) and skills hub | **Automatic fallback** in `_request()` (`github_token_from_env()` then `gh auth token`, cached per process) unless `updates.check: false` | `hermes_cli/github_api.py:9-64`, `hermes_cli/source_check.py:129-149` |
| `~/.openclaw`, `~/.clawdbot`, `~/.moltbot`, `~/.claude`, `~/.codex`, `~/.cursor`, Grok Bot, OpenBot app dirs | Detect prior agent installs during interactive `hermes setup` to offer `agent-importers` migration | **Interactive `hermes setup` only** (`is_dir()` / `glob()` presence checks; skipped when non-interactive; actual import installs plugin only after user confirms) | `hermes_cli/setup_agent_import.py:31-87` |
| `~/.claude/projects/*/*.jsonl` & `~/.codex/sessions/**/rollout-*.jsonl` | Import foreign chat transcripts into Hermes | **Explicit user action only** (`hermes sessions import` CLI / Desktop session import UI) | `hermes_cli/foreign_sessions.py:176-229,240-338` |
| Local machine-used marker & all local Hermes roots (`known_dirs.machine_marker_files()`) | Prevent one OS user from claiming the Desktop Welcome Trial multiple times across profiles/installs | Reads small known Hermes config/env/auth/DB markers and writes a marker outside `$HERMES_HOME` during Desktop local boot / trial check | `hermes_cli/install_state.py:31-112`, `tui_gateway/methods_onboarding.py:107-127`, `hermes_cli/free_tier_bootstrap.py:220-238` |
| `os.homedir()` `birthtimeMs` & `os.userInfo().username` | Desktop onboarding welcome/guide helper | **Local IPC only** (`hermes:machine:profile`); never exported in telemetry | `apps/desktop/electron/machine-profile.ts:15-48` |

---

## 3. Persistent Device / Install IDs & Hashing

Hermes generates three distinct random UUID identifiers for separate purposes. **No hardware IDs, MAC addresses, machine-ids, usernames, account emails, or cross-provider telemetry hashes exist:**

1. **`shared_metrics` Profile `install_id` (Telemetry Identity — OFF until opted in):**
   - Generated as `str(uuid.uuid4())` inside SQLite table `telemetry_state` (`$HERMES_HOME/telemetry/shared_metrics/metrics.sqlite3`, `hermes_cli/observability/shared_metrics.py:567-578`). Scoped to one `HERMES_HOME` profile, not a hardware device.
   - **Not created when disabled:** When `telemetry.shared_metrics.enabled` and `.send` are `False`, `_reconcile_send_consent_once()` checks `if not resolved.send and not default_path.exists(): return` (`hermes_cli/observability/relay_shared_metrics.py:1410-1416`), and `_run_task_hook` returns immediately on `if not enabled(): return` (`hermes_cli/observability/relay_shared_metrics.py:1606-1608`), so neither the SQLite file nor `install_id` is created.
   - **Raw transmission when opted in (no HMAC/rotation):** Once `send: true` is active, `SharedMetricsSender._freeze_identity()` locks `install_id` onto `package_outbox.sent_install_id` and `_body()` rebuilds the JSON envelope with `"install_id": transmitted_id` **verbatim** (`hermes_cli/observability/shared_metrics_sender.py:331-345`). Per the 2026-08-27 design decision in `website/docs/developer-guide/relay-shared-metrics.md:951-1010`, the earlier 30-day rotating HMAC-SHA256 pseudonym was intentionally removed so daily envelopes from one profile correlate indefinitely until the user deletes `$HERMES_HOME/telemetry/shared_metrics`.

2. **OTLP Gateway Monitoring `monitoring.install_id` (Operator Self-Hosted — OFF by default):**
   - Generated as `str(uuid.uuid4())` and persisted in `config.yaml` (`monitoring.install_id`) **only when `monitoring.gateway_health_export` or `monitoring.export.otlp` is enabled** (`agent/monitoring/policy.py:17-44`, `agent/monitoring/otlp_exporter.py:118-147`).
   - **Hashing:** `_safe_instance_id()` transforms `monitoring.install_id` using an **unsalted truncated SHA-256** (`f"sha256:{hashlib.sha256(value).hexdigest()[:24]}"`, `agent/monitoring/gateway_health.py:105-108`) as OpenTelemetry resource attribute `service.instance.id`.

3. **Core `$HERMES_ROOT/install_id` (Local Gateway / Hosted-Room Routing — Never sent to telemetry):**
   - `read_or_create_install_id()` creates `uuid.uuid4().hex` in `$HERMES_ROOT/install_id` (`hermes_cli/install_identity.py:61-85`).
   - Used locally to deduplicate gateway identities on `/api/status` (`hermes_cli/web_routers/status.py:663-665`), RPC `profiles.list` (`tui_gateway/methods_profiles.py:290-293`), and peer-to-peer hosted-room placement (`tui_gateway/hosted_room_peer_http.py:42-43`). Never attached to `shared_metrics` payloads.

4. **Desktop Local Hashing (`RendererCrashRecorder`):**
   - `apps/desktop/electron/renderer-crash-metrics.ts:48-53` hashes the Desktop profile scope key with unsalted `sha256(...).slice(0, 16)` in local file `renderer-crash-pending.json` solely so main-process crash entries stay isolated per profile and are never stored by plaintext profile name; only the bucketed crash reason (`crash`/`oom`/`killed`/`other`) is forwarded to the backend when consent is `on`.

---

## 4. Exact Outbound Telemetry / Analytics / Crash / OTEL / Relay / Update Endpoints

| Subsystem | Default State | Endpoint / Destination | Protocol / Client Headers | Code Citation |
|---|---|---|---|---|
| **Shared Metrics Ingest** | **OFF** (`enabled: false, send: false`) | `https://telemetry.nousresearch.com/v1/telemetry` (config-only override; HTTPS required except loopback; non-env-overridable) | HTTPS POST JSON (gzip if `> 4096` B), `User-Agent: hermes-agent-shared-metrics/1` | `hermes_cli/observability/shared_metrics_send_config.py:11-45,61-89`, `hermes_cli/observability/shared_metrics_sender.py:92-104`, `hermes_cli/config_defaults.py:2276-2285` |
| **Gateway Monitoring / OTLP** | **OFF** (`enabled: false`, no default URL) | Operator-configured `monitoring.export.otlp.endpoint` | OTLP/HTTP (`opentelemetry-exporter-otlp-proto-http`, optional `[otlp]` extra) | `agent/monitoring/otlp_exporter.py:1-8,150-158,257-259`, `pyproject.toml:462-469` |
| **NeMo Relay Exporters** | **OFF** unless user-configured `plugins.toml` or `OTEL_EXPORTER_OTLP_*` | User-configured OTLP / Phoenix / ATIF / ATOF destinations | Native `nemo-relay>=0.11,<0.12` | `agent/relay_runtime.py:291-320`, `pyproject.toml:223`, `website/docs/developer-guide/relay-shared-metrics.md:47-58,84-96` |
| **CUA Driver PostHog** | **OFF** (forced `"0"` unless `computer_use.cua_telemetry: true`) | Upstream `cua-driver` PostHog endpoint (disabled by env override) | Child env `CUA_DRIVER_RS_TELEMETRY_ENABLED=0` | `tools/computer_use/cua_backend.py:33-34,68-70,154-155`, `hermes_cli/config_defaults.py:2488-2491` |
| **Photon Spectrum Telemetry** | **OFF** (`PHOTON_TELEMETRY=false`) | Optional Photon iMessage sidecar (`@photon-ai/otel` / Spectrum SDK) | Env `PHOTON_TELEMETRY` (`false` default) | `plugins/platforms/photon/sidecar/index.mjs:62-63,82-84,240,337`, `plugins/platforms/photon/cli.py:331-352` |
| **Linux Electron Crashpad** | **Local-only** (`uploadToServer: false`) | None (local dump DB only) | `crashReporter.start({ uploadToServer: false, compress: false })` | `apps/desktop/electron/linux-crash-diagnostics.ts:15-16,39,95`, `apps/desktop/electron/main.ts:1379-1383` |
| **Manual Debug Bundle (`--nous` / Desktop Send Diagnostics)** | **Manual opt-in button / CLI command** | `POST https://portal.nousresearch.com/api/diagnostics/upload-url` (`HERMES_DIAGNOSTICS_BASE_URL`) + signed S3 `PUT`; public default `hermes debug share` uses `paste.rs` / `dpaste.com` | `User-Agent: hermes-agent/debug-share`; explicit consent notice + prompt (`--yes` / click Upload) | `hermes_cli/diagnostics_upload.py:13-62`, `hermes_cli/debug.py:469-652`, `apps/desktop/src/components/send-diagnostics-dialog.tsx:39-123` |
| **Anonymous Free-Tier Welcome Trial** | **Automatic on first local Desktop onboarding** (when unconfigured; otherwise explicit UI/connector trigger) | `https://portal.nousresearch.com/api/anonymous/create`, `/api/anonymous/token`, and `inference_base_url` | Sends `{}` (or `preview_full_connectors`), plus self-reported `{name, version, surface, platform, capabilities}` on token exchange | `hermes_cli/anon_auth.py:374-434`, `hermes_cli/anon_challenge.py:133-138`, `tui_gateway/methods_onboarding.py:150-199`, `hermes_cli/welcome_trial.py:114-140` |
| **CLI & Desktop Update Checks** | **ON by default** (`updates.check: true`; Desktop startup + daily poll) | CLI: `https://api.github.com/repos/NousResearch/hermes-agent/...` + `git ls-remote`; Desktop: `CLOUDFLARE_R2_PUBLIC_URL` channel/feed (`electron-updater`), App Installer, Microsoft Store, or `updates.desktop_feed_base_url` | CLI uses `GITHUB_TOKEN`/`GH_TOKEN`/`gh auth token` if present | `hermes_cli/banner.py:275-288`, `hermes_cli/source_check.py:114-149,588-589`, `hermes_cli/github_api.py:62-64`, `apps/desktop/src/store/updates.ts:1385-1415`, `apps/desktop/electron-builder.config.cjs:69-90` |
| **Model Catalog Metadata** | **Automatic when needed** (cold cache / >4h stale) | `https://models.dev/api.json` (`models_dev.url` mirror), `https://openrouter.ai/api/v1/models`, Nous Portal `/v1/models` | Conditional `If-None-Match` ETag GET (`models.dev`) | `agent/models_dev.py:27-30,278-282,342-368,492-547`, `hermes_cli/models_reasoning_caps.py:272-292` |

---

## 5. Exact Data & Properties Collected

When **`telemetry.shared_metrics.enabled` and `.send` are both opted in** (`hermes.shared_metrics.v4` schema in `hermes_cli/observability/schemas/hermes.shared_metrics.v4.schema.json:7-85`):

- **Resource envelope & IDs:** `schema_version` (`"hermes.shared_metrics.v4"`), `package_id` (UUIDv4), `install_id` (raw profile-scoped UUIDv4), `period_start`, `period_end`, `generated_at`, and bounded `resource` (`architecture`, `hermes_version`, `install_method`, `os_family`).
- **Counters & bounded usage metrics (contract-enforced in `hermes_cli/observability/shared_metrics_contract.py`):**
  - **Session & task rollups:** `hermes.client.active`, `hermes.session.count` (duration bucket, entrypoint, execution surface, messaging platform from closed/catalog allowlist, turn count bucket, failed turns, last outcome), `hermes.task_run.started`/`.finished`/`.duration` (duration bucket, error/failure class from closed enums).
  - **Model & token metrics:** `hermes.model_route.count`, `hermes.model_tokens.sum` (token totals: input, output, cache read/write, reasoning), `hermes.model_switch.count`, `hermes.fallback.count`, `hermes.model_tool_quality.count`, `hermes.model_friction.count`, `hermes.context_peak.count`.
  - **Tools, skills, plugins, compression, memory & setup:** `hermes.tool_call.count`/`.latency`/`.usage.count`/`.approval.count` (built-in tool names or category only; MCP/custom tools collapse), `hermes.skill.lifecycle.count`/`.load.count` (bundled/optional skill names only; user skills -> `"custom"`), `hermes.extension.install.count` (catalog names or `"custom"` + closed `failure_class` + `registry`), `hermes.slash_command.count` (built-in command names or `"skill"`/`"plugin"`/`"unknown"`), `hermes.compression.count` / `hermes.prepared_compaction.count`, `hermes.memory.op.count`, `hermes.setup.completed`, `hermes.feature_disabled.count` (default-on toggles/features disabled), `hermes.update.run`, `hermes.install.run`, `hermes.startup.latency`.
  - **Install snapshot (`hermes.install.snapshot`):** Bucketed counts (`cron_job_count_bucket`, `mcp_server_count_bucket`, `plugin_count_bucket`, `profile_count_bucket`, `skill_count_bucket`, `messaging_platform_count_bucket`), `display_language`, `install_age_bucket`, `main_provider`, `memory_provider`, `terminal_backend`, `release_channel`, `version_age_bucket`, `behind_bucket`, `ram_bucket`, `gpu_class`, `local_model_provider_used`, `hosting` (`nous_cloud` vs `self_hosted`).
  - **Desktop behavioral profile (when Desktop runs against an opted-in backend):** Area open counts (`desktop_feature_use`), daily rollup (`desktop_daily`: action IDs via click/shortcut/menu/palette, active ms and message count per `bots`/`sessions` mode, bot count), `desktop_long_frames` histogram buckets, `desktop_onboarding` step funnels, `desktop_friction` (renderer crash bucket, backend disconnect, slow frames), and `desktop_dislike` signals (`quick_close`, `cancelled`, `rage_click`, `feature_disabled`, `setting_off_default` carrying config key names from schema defaults only — never values, `apps/desktop/src/store/desktop-metrics.ts:48-150,707-829,1030-1047`, `hermes_cli/observability/shared_metrics_desktop.py:1-140`).

**Specific Rubric Dimension Checks:**
- **Prompts / Code / File Paths / Raw Stack Traces:** Never collected by `shared_metrics` (strictly closed enums, buckets, and catalog allowlists) or by gateway OTLP monitoring (bounded/redacted operational diagnostics). Full logs, file paths, and conversation excerpts can **only** leave the machine via **manual user-confirmed** `hermes debug share --nous` or Desktop's `"Send diagnostics to Nous"` dialog (redacted by default, 14-day retention, `hermes_cli/debug.py:429-455,576-630`, `apps/desktop/src/i18n/en.ts:514-536`).
- **IP Address & GeoIP Handling:** The client performs **zero GeoIP lookups** and adds **no IP-stripping/proxy-suppression headers** (`shared_metrics_sender.py:92-118`). When `send: true` is opted in, direct client HTTPS POSTs expose the caller's source IP address to `telemetry.nousresearch.com` (and on default-on Desktop trial minting / update checks / `models.dev` calls, to `portal.nousresearch.com`, `api.github.com` / R2, and `models.dev`).
- **Custom / Private Model Name Handling (Verified Gap):**
  - `provider_metric_name()` maps `custom`, `custom:*`, unknown providers, and aliases (`ollama`, `vllm`, `llamacpp`) to `"custom"` (`hermes_cli/observability/shared_metrics_catalog.py:122-129,261-272`), and `model_metric_name()` collapses `user_named_model_providers` (`custom`, `lmstudio`, loopback providers), `provider == "unknown"`, `_LOCATION_MODEL` regex matches (`arn:`, paths with `:/`, `.gguf`/`.bin`/`.safetensors`, `localhost`/IPv4/`host:port`), and non-public Azure deployments to `"custom"` (`shared_metrics_catalog.py:103-120,285-325`).
  - **Leak edge case when opted in:** `model_call_fields()` passes `provider` and `model`/`response_model` without inspecting `base_url` (`hermes_cli/observability/shared_metrics_contract.py:1926-1934`, fed by `agent/turn_response_intake.py:88-90`). Meanwhile `_anthropic_env_runtime` (`hermes_cli/runtime_provider.py:1038-1055`), `_api_key_provider_runtime` (`runtime_provider.py:1057-1083`), and named `providers.openai` overrides keep `agent.provider` as the built-in name (e.g. `anthropic`, `openrouter`, `openai`, `bedrock`, `vertex`) even when `model.base_url` points to a private proxy, and `model_metric_name()` does **not** check `public_model_ids()` for non-Azure providers. Consequently, organization-private/fine-tuned model IDs (e.g., `ft:gpt-4o:acme:internal:...`) or custom proxy model IDs on non-Azure built-in providers are exported verbatim (lowercased, max 256 chars, `shared_metrics_contract.py:102-105`) when `shared_metrics` is enabled.

---

## 6. Docs / Privacy Policy vs Code Discrepancies

1. **Direct contradiction in `website/docs/reference/faq.md` (CRITICAL DOCS DISCREPANCY):**
   - `website/docs/reference/faq.md:56-58` states:
     > *"API calls go **only to the LLM provider you configure** (e.g., OpenRouter, your local Ollama instance). Hermes Agent does not collect telemetry, usage data, or analytics."*
   - **Reality:** While telemetry is **opt-in / off by default**, Hermes explicitly implements an opt-in product telemetry pipeline (`telemetry.shared_metrics` sending daily bundles to `https://telemetry.nousresearch.com/v1/telemetry`, Desktop UI behavioral/crash/frame metrics, `cua_telemetry` / `PHOTON_TELEMETRY` toggles, and optional OTLP export), plus default-on non-telemetry calls (`updates.check` to GitHub/R2, `models.dev`, and automatic first-run Desktop anonymous trial minting at `portal.nousresearch.com`). Note that `website/docs/developer-guide/relay-shared-metrics.md`, `cli-config.yaml.example:2245-2268`, and UI consent dialogs document `shared_metrics` accurately, leaving `faq.md:58` dangerously stale.
2. **Broken documentation path in `cli-config.yaml.example`:**
   - `cli-config.yaml.example:2262` instructs users: *"See docs/observability/relay-shared-metrics.md (Appendix A) for the full consent, identity, retention, and deletion decisions."* That file does **not** exist (`git ls-files docs/` returns empty); the actual document is `website/docs/developer-guide/relay-shared-metrics.md` (correctly cited in `hermes_cli/config_defaults.py:2275`).
3. **Ingest domain omitted from Official Domains list:**
   - `website/docs/getting-started/official-domains.md:12-20` enumerates Nous subdomains used by Hermes Agent (`hermes-agent.nousresearch.com`, `hermes-assets.nousresearch.com`, `portal.nousresearch.com`, `inference-api.nousresearch.com`, `nousresearch.com`) and warns against unlisted domains (`official-domains.md:49,58`), yet omits `telemetry.nousresearch.com` (`hermes_cli/observability/shared_metrics_send_config.py:15`).
4. **Incomplete external-login documentation in Security guide:**
   - `website/docs/user-guide/security.md:737-746` documents `auth.adopt_external_logins: false` for Codex CLI and Claude Code, stating Hermes borrows those two external logins by default. It does not mention `~/.qwen/oauth_creds.json` (`hermes_cli/auth_qwen.py:25-152`, `agent/credential_pool.py:2579-2597`), which is read when `qwen-oauth` is loaded/status-checked without consulting `adopt_external_logins`.

---

## 7. Zero-Telemetry / Negative-Check Verification

Targeted read-only `git grep` verification across manifests (`pyproject.toml`, `package.json`, `apps/desktop/package.json`, `web/package.json`, `ui-tui/package.json`) and core source directories (`hermes_cli/`, `agent/`, `gateway/`, `tools/`, `tui_gateway/`, `apps/`, `web/`, `ui-tui/`, `.github/workflows/`):

1. **No third-party analytics or hosted crash SDKs in Hermes core, Desktop, Web, or TUI:**
   - `git grep -ni -E "\b(posthog|sentry|mixpanel|amplitude|bugsnag|rollbar|raygun|langfuse|agentops|wandb)\b" -- pyproject.toml package.json apps/ web/ ui-tui/ hermes_cli/ agent/ gateway/ tools/ tui_gateway/ plugins/ ":!*.test.*" ":!tests/**"`
   - **Verified matches only:** icon mappings (`apps/desktop/src/lib/brand-icon.ts:232,324`, `mcp-brands.tsx:87`), UI animation math (`amplitude` in `image-generation-placeholder.tsx:85-91`, `particle-field.tsx:31`, `config_defaults_voice.py:26`), secret/pattern redaction & security scanning (`agent/redact.py:213`, `hermes_cli/plugins_install_scan_context.py:763-767`), `skills_hub_github.py` / `skills_guard.py` (`wandb/skills`), comment/migration shims for the removed optional `langfuse` plugin (`hermes_cli/left_core_migration.py:267-269,584-589`, `plugins/AGENTS.md:58`, `agent/turn_api_request.py:82`), and the comment/guard actively **disabling** upstream `cua-driver` PostHog telemetry (`hermes_cli/config_defaults.py:2489`, `tools/computer_use/cua_backend.py:33-34`). **Zero** analytics/crash SDKs are bundled or initialized.
2. **Website Docusaurus config & workflows free of analytics trackers:**
   - Checked `website/docusaurus.config.ts:1-160` and `.github/workflows/` — zero Google Analytics (`gtag`/`GA4`), Plausible, PostHog, or Sentry configuration (`website/docusaurus.config.ts:156-160` uses Algolia DocSearch public search-only key for docs search).
3. **`DO_NOT_TRACK` check:**
   - `git grep -n "DO_NOT_TRACK"` returned **only** two third-party catalog YAML descriptions (`plugin-catalog/renor.yaml:6`, `plugin-catalog/telesec.yaml:5`), confirming `DO_NOT_TRACK` is not wired into core Hermes telemetry.
4. **No GeoIP / external IP-lookup endpoints:**
   - `git grep -ni -E "geoip|ipinfo\.io|ip-api\.com|maxmind|cdn-cgi/trace"` across code returned only third-party catalog plugin description `plugin-catalog/picky.yaml:4` (excluded `website/`/`skills/`).
5. **No hardcoded OTLP telemetry destination:**
   - `agent/monitoring/otlp_exporter.py:3-4,257-259` confirms zero default OTLP endpoint and optional-only `[otlp]` extra (`pyproject.toml:466-469`).

---

```json
{
  "slug": "hermes",
  "platform": "Hermes",
  "group": "B",
  "repo": "NousResearch/hermes-agent",
  "sha": "a39b00bf378f0a9ca6e8e8ba27af4ca4bf8cd86d",
  "commit_date": "2026-11-24T23:05:06-06:00",
  "license": "MIT",
  "status": "Active",
  "default_state": "OFF by default (Opt-In). telemetry.shared_metrics.enabled=false and .send=false; monitoring.gateway_health_export.enabled=false and monitoring.export.otlp.enabled=false; CUA_DRIVER_RS_TELEMETRY_ENABLED forced to 0; PHOTON_TELEMETRY=false; Electron Linux crashReporter uploadToServer=false. Non-telemetry defaults ON: passive update checks (updates.check=true), models.dev/openrouter catalog fetches, and Desktop first-run anonymous trial minting on unconfigured local installs.",
  "opt_out_controls": "Separate profile-scoped enabled & send booleans in $HERMES_HOME/config.yaml; `hermes setup telemetry`; first-run CLI/TUI/Desktop strip/Web banner prompts; Desktop Settings -> Safety -> Privacy toggle; `telemetry.shared_metrics.send` fails closed if enabled=false; shared_metrics endpoint config-only (not env-overridable); computer_use.cua_telemetry=false.",
  "do_not_track": "NOT honored in core telemetry (shared_metrics / monitoring / Desktop / Web). Only mentioned in 3rd-party plugin-catalog/*.yaml descriptions.",
  "opt_out_bypasses": "No silent transmission bypasses when opted out (metrics.sqlite3/install_id not created when off; pre-consent pending_installs/ receipts deleted unreported on next start; Desktop in-memory preConsent onboarding queue dropped on No thanks). Caveat: OFFER_VERSION=2 causes one re-prompt for legacy opt-outs lacking offer_version=2.",
  "persistent_ids": "1) shared_metrics install_id (UUIDv4 in metrics.sqlite3, profile-scoped, ONLY created when opted in; reset by deleting telemetry/shared_metrics); 2) monitoring.install_id (UUIDv4 in config.yaml, ONLY minted when OTLP monitoring enabled); 3) core $HERMES_ROOT/install_id (uuid4 hex, local /api/status & hosted-room routing only, never sent in telemetry).",
  "hashing": "shared_metrics transmits the raw UUIDv4 install_id verbatim without hashing or salt rotation (intentional 2026-08-27 design decision in relay-shared-metrics.md Appendix A.2 for indefinite longitudinal profile correlation). OTLP monitoring hashes monitoring.install_id via unsalted truncated sha256:<24-hex> (`service.instance.id`). Desktop hashes local profile keys in renderer-crash-pending.json via unsalted sha256[:16].",
  "external_auth_reads": "auth.adopt_external_logins defaults to TRUE: automatically reads/refreshes ~/.claude/.credentials.json + macOS Keychain 'Claude Code-credentials' (also probed in _has_any_provider_configured when non-default model set) and ~/.codex/auth.json; ~/.qwen/oauth_creds.json read/refreshed for qwen-oauth WITHOUT checking adopt_external_logins; gh auth token run for GitHub API rate-limit relief on passive update checks; interactive setup checks .openclaw/.claude/.codex/.cursor dirs; explicit session import reads ~/.claude/projects & ~/.codex/sessions. External auth is NEVER sent to telemetry.",
  "telemetry_endpoints": "https://telemetry.nousresearch.com/v1/telemetry (HTTPS POST JSON/gzip, opt-in daily shared_metrics); operator-configured monitoring.export.otlp.endpoint (opt-in OTLP/HTTP, no default); https://portal.nousresearch.com/api/diagnostics/upload-url + signed S3 PUT (manual opt-in debug share --nous). Non-telemetry endpoints: https://portal.nousresearch.com/api/anonymous/{create,token}, api.github.com / R2 update feeds, models.dev/api.json, openrouter.ai/api/v1/models.",
  "ip_geo_handling": "No client-side GeoIP lookups and no IP-suppression headers; direct client HTTPS POSTs expose client source IP to telemetry.nousresearch.com (and portal/update/catalog hosts).",
  "custom_model_handling": "custom/ollama/vllm/llamacpp/lmstudio/loopback providers, _LOCATION_MODEL paths/IPs/ARNs, and unknown Azure deployments collapse to 'custom'. Gap: non-Azure built-in providers overridden with a custom model.base_url (e.g. anthropic/openrouter/providers.openai) or org fine-tunes export verbatim lowercased model IDs (<=256 chars) when opted in, because model_call_fields ignores base_url and model_metric_name skips public_model_ids() outside Azure.",
  "docs_discrepancies": "1) website/docs/reference/faq.md:58 falsely claims 'Hermes Agent does not collect telemetry, usage data, or analytics' and API calls go only to the configured LLM provider, contradicting relay-shared-metrics.md and the shipped opt-in telemetry system; 2) cli-config.yaml.example:2262 points to nonexistent docs/observability/relay-shared-metrics.md; 3) website/docs/getting-started/official-domains.md omits telemetry.nousresearch.com; 4) website/docs/user-guide/security.md omits qwen-oauth (~/.qwen/oauth_creds.json) under Borrowed CLI logins.",
  "privacy_rating": "A",
  "one_line_verdict": "Strict opt-in privacy architecture (shared metrics, OTLP, PHOTON_TELEMETRY, and forced CUA_DRIVER_RS_TELEMETRY_ENABLED=0 all OFF by default, fail-closed, zero install ID or DB created when off, unsent pending_installs purged on launch, zero 3rd-party analytics SDKs), with remaining caveats limited to stale FAQ wording and opt-in-only/external-auth items (verbatim non-Azure custom/fine-tuned model IDs, raw persistent profile UUID + IP when enabled, ignored DO_NOT_TRACK, default-on external CLI auth adoption)."
}
```
