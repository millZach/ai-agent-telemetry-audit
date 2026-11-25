# Forensic Telemetry & Privacy Audit: Aether (`PrashnaLabs/aether`)

- **Slug:** `aether`
- **Platform:** `Aether`
- **Group:** `A`
- **Repository:** [`PrashnaLabs/aether`](https://github.com/PrashnaLabs/aether)
- **License:** `MIT` (`Cargo.toml:8`)
- **Status:** `Active`
- **Branch:** `main`
- **Audited Commit SHA:** `a9a941541dbd3a744ad9276afa56ddbdeb638632`
- **Commit Date:** `2026-11-13T21:05:01+05:45`
- **Commit Subject:** `release: alpha-0.1.1`
- **Overall Privacy Rating:** **`A`** *(Zero-Telemetry / Local-First Clean)*

---

## Executive Summary

**Aether contains zero first-party or third-party product telemetry, zero analytics SDKs, zero crash reporters, zero persistent device/install identifiers, and zero auto-update checks.** No network calls contact Prashna Labs (`prashnalabs.com`, `aether.prashnalabs.com`, or `durbarai.prashnalabs.com`) anywhere in runtime code or packaging. All local voice transcription runs on-device via `whisper-rs`, and Pi CLI's built-in update check is explicitly suppressed (`PI_SKIP_VERSION_CHECK=1`).

However, to power its cross-agent desktop features (status-bar usage gauges, OpenRouter model picker, and token-cost dashboard), Aether performs **unconditional startup and periodic background actions** with no settings opt-out toggle: (1) reading external Claude Code and Codex credentials (including the OS keychain entry `"Claude Code-credentials"`, which code comments note can trigger a macOS keychain prompt) on startup and every 10 minutes to poll Anthropic and ChatGPT quota endpoints; (2) calling OpenRouter `list_models()` on startup even when no API key is set; and (3) fetching LiteLLM's model pricing JSON from GitHub (`raw.githubusercontent.com`) when the Usage page is opened. None of these transmit telemetry to Aether's developers or link identities across providers.

---

## 1. Default State & Opt-Out Mechanism

### Product Telemetry / Crash Reporting / Auto-Updater
- **Default State:** **Absent (Zero-Telemetry).** There is no telemetry module, analytics client, crash reporter, OTEL exporter, or updater in `Cargo.toml`, `Cargo.lock`, `.github/workflows/build.yml`, `scripts/build.py`, or `crates/`.
- **Opt-Out Controls & `DO_NOT_TRACK`:** None exist because there is no telemetry pipeline to disable (`crates/core/src/settings.rs:884-1091,1179-1274`). Negative searches confirm zero handling of `DO_NOT_TRACK`, `TELEMETRY_DISABLED`, or analytics toggles. Aether also sets `PI_SKIP_VERSION_CHECK=1` when spawning the external `pi` CLI (`crates/driver/src/catalog.rs:188,514`).

### Non-Telemetry Feature Network Defaults & Controls
Even though Aether sends zero telemetry, several user-facing integrations issue outbound requests by default or without an offline toggle:
- **Unconditional Startup & 10-Minute Provider Quota Polling (No Opt-Out):** On startup (`crates/app/src/app/new.rs:954-955`) and every 30 seconds while the window is active when cached usage is >=10 minutes stale (`crates/app/src/app/new.rs:810-818`; `crates/app/src/app/providers.rs:19,25-28,162-171`), Aether calls `refresh_all_provider_usages()` (`crates/app/src/app/providers.rs:185-196`) and `detect_subscription()` (`crates/app/src/app/providers.rs:41-55`). There is **no setting** in `Settings` (`crates/core/src/settings.rs:884-1091`) or UI to disable quota polling or cross-tool credential reading. Toggling `"usage"` or individual agents in **Settings > Bottom bar** (`crates/core/src/settings.rs:1018-1023`; `crates/app/src/settings_view.rs:692-780`) only hides UI rendering and does not stop `refresh_all_provider_usages()`.
- **Unconditional Startup OpenRouter Model Fetch (No Opt-Out):** `crates/app/src/app/new.rs:740` calls `load_models()`, which constructs `OpenRouterClient::new(OpenRouterConfig::new(api_key, "openrouter/auto"))` (`crates/app/src/app/providers.rs:198-200`, `crates/agent/src/agent.rs:340-344`, `crates/agent/src/models.rs:15-18`) and calls `client.list_models().await` even when `self.api_key` is `None` (passing `""`).
- **LiteLLM Pricing Fetch on Opening Usage Page (No Offline Toggle):** Opening `Page::Usage` (`crates/app/src/app/mod.rs:770-772`; `crates/app/src/usage_view/mod.rs:151`) triggers `usage_history::scan_cached()` -> `load_rate_table()` (`crates/agent/src/usage_history/aggregate.rs:166`; `crates/agent/src/usage_history/pricing.rs:151-184`), which fetches LiteLLM's model price JSON from GitHub (`crates/agent/src/usage_history/pricing.rs:9-12,115-131`) whenever `~/.aether/usage-model-rates.json` is absent or >24 hours old.
- **User-Controlled Features:**
  - `web_search`: Defaults to `true` (`crates/core/src/settings.rs:1194`), only registers a DuckDuckGo tool for OpenRouter turns (`crates/agent/src/tools/mod.rs:39-41`), defaults to `ToolPolicy::Ask` requiring per-call approval (`crates/core/src/settings.rs:1164-1171`), and can be toggled off in **Settings > Model > Web search** (`crates/app/src/settings_view.rs:1133-1151`).
  - `notify_webhook`: Defaults to `""` (off) (`crates/core/src/settings.rs:1006-1008,1235`; `crates/app/src/notify.rs:78-82`); fires only if explicitly populated in `settings.json`.
  - `hackernews_enabled` & `voice_enabled`: Default to `true` (`crates/core/src/settings.rs:1246,1258`), but perform **zero background network traffic** unless the user manually opens the Hacker News page (`crates/app/src/hackernews/view.rs:777-852`) or clicks to download the Whisper speech model (`crates/voice/src/model.rs:52-90`).
  - `AETHER_RECORD`: Local scrubbed debug recording is opt-in via environment variable and writes only to disk (`crates/driver/src/record.rs:17-70`).

---

## 2. External Auth / Local Non-Workspace File Reads

**Aether reads external coding-agent credentials, configs, and session transcripts strictly for local orchestration, quota display, and transcript viewing—never for developer telemetry.** However, credential reads occur automatically on launch without user confirmation:

### A. External Credential & Keychain Reads (Startup + Every 10 Minutes)
1. **Claude Code OAuth Token & Subscription Tier** (`crates/agent/src/usage.rs:484-549`):
   - Triggered automatically on startup (`crates/app/src/app/new.rs:954-955`), on 10-minute active refresh (`crates/app/src/app/providers.rs:19,57-82,162-171`), and on spawn (`crates/app/src/app/providers.rs:174-183`).
   - Checks `ANTHROPIC_AUTH_TOKEN` and `CLAUDE_CODE_TOKEN` env vars (`usage.rs:513-522`).
   - Reads OS keychain service `"Claude Code-credentials"` for accounts `[env("USER"), "claude", ""]` via `keyring::Entry::new("Claude Code-credentials", acct)` (`usage.rs:487-498,524-530`). Comment at `usage.rs:524` explicitly notes: `// keychain read can raise a blocking macOS password prompt: ask once, reuse until forgotten`.
   - Falls back to reading `~/.claude/.credentials.json`, `~/.claude.json`, and `~/.config/claude/credentials.json` (or `<account_dir>/.credentials.json` when configured) (`usage.rs:508-512,532-546`).
   - Sends token **only** to `https://api.anthropic.com/api/oauth/usage` (`usage.rs:566`).
2. **OpenAI Codex Auth Token** (`crates/agent/src/usage.rs:581-600`):
   - Triggered on startup (`new.rs:954-955`) and every 10 minutes (`providers.rs:83-97`).
   - Reads `$CODEX_HOME/auth.json` or `~/.codex/auth.json` (JSON pointer `/tokens/access_token`, or `<account_dir>/auth.json`).
   - Sends token **only** to `https://chatgpt.com/backend-api/wham/usage` (`usage.rs:608`).
3. **OpenCode Local Service Auth** (`crates/driver/src/opencode/service.rs:120-169`; `crates/agent/src/usage.rs:658-670`):
   - Reads local `opencode/service.json` (URL + password) from `$XDG_STATE_HOME`, `~/.local/state`, `$XDG_CONFIG_HOME`, `~/.config`, and `~/.opencode`. Used only for localhost HTTP basic auth against the local OpenCode process.
4. **Aether's Own OpenRouter Key** (`crates/app/src/settings.rs:91-104`; `crates/agent/src/usage.rs:622-636`):
   - Reads `OPENROUTER_API_KEY` or OS keychain service `"aether"` / account `"openrouter-api-key"`. Stored in keychain (`settings_view.rs:1040`), never in `settings.json`.

### B. Other Non-Workspace Local Reads (Functional Workspace / Agent Features)
- **Startup Codex Model Context Cache (`crates/app/src/settings.rs:21-30`; `crates/driver/src/catalog.rs:332-334,405-417`):** `settings::load()` unconditionally reads `$CODEX_HOME/models_cache.json` or `~/.codex/models_cache.json` on startup (and writes context window sizes into Aether's `settings.json`). `probe_providers()` (`new.rs:791`) also reads `models_cache.json`.
- **Startup & Periodic External Agent Process/Session Scanning (`crates/app/src/app/new.rs:792,885-896`; `crates/app/src/agents_view.rs:18-20`; `crates/agent/src/external.rs:55-105,119-266,278-289,640-698`):** Runs on startup and every 10 seconds (2 seconds while `Page::Agents` is open), invoking `ps` and `lsof` and reading `~/.claude/sessions/<pid>.json` (comment `external.rs:120` explicitly skips sibling credential files: `// only <pid>.json: the sibling <pid>.*.key files are credentials`), `~/.pi/agent/sessions`, `~/.codex/session_index.jsonl`, `~/.gemini/antigravity-cli/brain`, and local OpenCode `/api/session`. When a user opens an external agent entry, `external::transcript()` copies/queries `~/.gemini/antigravity-cli/conversations/<id>.db` or reads provider transcript JSONL files.
- **Startup Skills Discovery (`crates/app/src/app/new.rs:616`; `crates/agent/src/skills.rs:10-73`):** Scans `SKILL.md` in Aether's config dir plus `~/.claude/skills`, `~/.config/claude/skills`, `~/.gemini/skills`, `~/.gemini/antigravity-cli/builtin/skills`, `~/.gemini/config/plugins/superpowers/skills`, `~/.config/opencode/skills`, `~/.opencode/skills`, `~/.codex/skills`, `~/.config/codex/skills`, and `~/.agents/skills`.
- **Usage Dashboard Transcript Scan (`crates/app/src/app/mod.rs:770-772`; `crates/agent/src/usage_history/discovery.rs:16-46,123-150`):** On opening `Page::Usage`, reads transcripts from `~/.claude/projects`, `~/.codex/sessions`, `~/.pi/agent/sessions`, `~/.gemini/antigravity-cli/brain`, `~/.gemini/antigravity/brain`, and `~/.local/share/opencode/storage/{message,session}`.
- **Context Panel Inventory (`crates/app/src/context_panel.rs:174-189`; `crates/agent/src/inventory.rs:68-127,284-333`):** When the Context Panel renders, scans external memory (`CLAUDE.md`, `AGENTS.md`, `GEMINI.md`), MCP configs (`~/.claude.json`, `~/.codex/config.toml`, `~/.pi/agent/mcp.json`, `~/.config/opencode/opencode.json{,c}`, `~/.gemini/config/mcp_config.json`), and custom agent dirs.
- **CLI Session Drivers (`crates/driver/src/pi.rs:331-340`; `crates/driver/src/antigravity.rs:742-745,834-857`):** Pi handshake reads `$PI_CODING_AGENT_DIR` / `~/.pi/agent/{settings.json,trust.json}`; Antigravity reads `~/.gemini/antigravity-cli/annotations/<id>.pbtxt` for titles and `~/.gemini/config/mcp_config.json` (or runs `agy mcp add aether`) for MCP registration.
- **Aether Config & `.env`:** Stores local state in `<config_dir>/aether/` (`settings.json` with `0o600` Unix perms, SQLite `aether.db`, `attachments/`, `models/ggml-base.en.bin`) (`crates/core/src/paths.rs:11-50`; `crates/app/src/settings.rs:75-85`; `crates/app/src/app/new.rs:961-966`) plus `~/.aether/usage-model-rates.json` (`crates/agent/src/usage_history/aggregate.rs:166`). Calls `dotenvy::dotenv().ok()` on launch (`crates/app/src/main.rs:54`).

---

## 3. Persistent Device / Install IDs & Hashing

- **Persistent Device / Install / Machine / Analytics IDs:** **None (`none`).**
  - Aether's `Settings` schema (`crates/core/src/settings.rs:884-1091`) and SQLite schema (`crates/agent/src/store.rs:11-55`) store no UUID, installation ID, device ID, or host fingerprint.
  - `getrandom` (`Cargo.toml:28`; `crates/core/src/sys.rs:51-55`) is used exclusively for three **ephemeral / local-only** purposes:
    1. In-memory per-lead orchestrator bearer tokens (`crates/agent/src/agent.rs:263-272`) protecting the local Unix socket (`0600` in `0700` directory) or loopback `127.0.0.1:0` socket (`crates/agent/src/orchestrator/server.rs:44-79`).
    2. Ephemeral UUIDv4 Claude session IDs passed to `--session-id` (`crates/driver/src/claude/spawn.rs:411,456-462`).
    3. Random temporary directory suffix for copying Antigravity's WAL SQLite DB (`crates/agent/src/external.rs:685-686`).
- **Hashing:** **Zero PII or identifier hashing.**
  - `sha2::Sha256` (`Cargo.toml:36`; `crates/voice/src/model.rs:2,44-46,92-118`) is used solely to verify the downloaded Whisper model file (`ggml-base.en.bin`) against its pinned SHA-256 digest.
  - `std::hash::DefaultHasher` (`crates/driver/src/claude/spawn.rs:424-428`) hashes `lead.token` purely to name a local temp file (`aether-mcp-<hash>.json`, `0o600`), with no external transmission.

---

## 4. Exact Outbound Telemetry / Analytics / Crash / OTEL / Relay / Update Endpoints

### Telemetry / Analytics / Crash / OTEL / Auto-Update Endpoints
**None.** Exhaustive `reqwest`, `http://`, and `https://` audits across `crates/` and `vendor/gpui-base/src/` confirm **zero** first-party Prashna Labs APIs, **zero** analytics/telemetry collectors, **zero** crash-reporting endpoints, and **zero** update-check endpoints. `Cargo.lock:253-262` includes `adk-telemetry 2.3.0` transitively via `adk-rust` (`Cargo.toml:21`), but `Cargo.lock` confirms it depends only on `thiserror`, `tracing`, and `tracing-subscriber`—with zero OpenTelemetry/OTLP or network exporters.

### Exhaustive Catalog of Non-Telemetry Outbound Endpoints

| Endpoint / URL | Trigger & Frequency | Purpose | File:Line |
|---|---|---|---|
| `https://api.anthropic.com/api/oauth/usage` | Automatic on startup + every 10m if active (plus spawn/manual) | Claude Code quota/rate-limit bars | `crates/agent/src/usage.rs:566` (`new.rs:954`, `providers.rs:66,168`) |
| `https://chatgpt.com/backend-api/wham/usage` | Automatic on startup + every 10m if active (plus spawn/manual) | Codex quota/rate-limit bars | `crates/agent/src/usage.rs:608` (`new.rs:954`, `providers.rs:87,168`) |
| `https://openrouter.ai/api/v1/models` *(via `adk-rust` `OpenRouterClient::list_models`)* | Automatic on startup (`api_key` or `""`) + on API key save | Populate OpenRouter model picker, context lengths, prices | `crates/app/src/app/providers.rs:198-200,684`; `crates/agent/src/agent.rs:341`; `crates/agent/src/models.rs:15-18` |
| `https://openrouter.ai/api/v1/credits` | Automatic on startup + every 10m if `OPENROUTER_API_KEY` or keychain key exists | Display OpenRouter credit balance | `crates/agent/src/usage.rs:644` (`providers.rs:114`) |
| OpenRouter Chat Completions *(via `adk-rust` `OpenRouterClient`)* | User sends message with OpenRouter selected | LLM inference | `crates/agent/src/turn.rs:51` |
| `https://raw.githubusercontent.com/BerriAI/litellm/main/model_prices_and_context_window.json` | User opens Usage page (cached 24h in `~/.aether/usage-model-rates.json`) | Token cost estimation | `crates/agent/src/usage_history/pricing.rs:9-12,121` (`usage_view/mod.rs:151`) |
| `https://huggingface.co/ggerganov/whisper.cpp/resolve/5359861c739e955e79d9a303bcbc70fb988958b1/ggml-base.en.bin` | Explicit user click to download speech model | Local Whisper dictation model (141 MB, SHA-256 verified) | `crates/voice/src/model.rs:10-12,77` |
| `https://html.duckduckgo.com/html/` | OpenRouter `web_search` tool call (`ToolPolicy::Ask` approval) | Web search tool | `crates/agent/src/tools/web_search/duckduckgo.rs:6,22-30` |
| `https://hacker-news.firebaseio.com/v0/{feed}.json` & `/item/{id}.json` | Explicit user opens Hacker News page / changes feed / refreshes | Built-in HN reader | `crates/app/src/hackernews/client.rs:9`; `crates/app/src/hackernews/view.rs:787,824-865` |
| User-configured `notify_webhook` URL | Notification event (`done` / `request`) if non-empty in `settings.json` | User notification webhook (off by default) | `crates/app/src/notify.rs:78-102`; `crates/app/src/app/chat/send.rs:605` |
| `https://www.google.com/search?q={encoded}` / arbitrary user URLs | Explicit user input in embedded `wry` browser panel | In-app browser search & browsing | `crates/app/src/browser.rs:116,167-244` |

*(Localhost-only HTTP: OpenCode local daemon at `http://{host}:{port}` from `service.json` in `crates/driver/src/opencode/service.rs:138-145`.)*

---

## 5. Exact Data & Properties Collected

- **Telemetry Payloads Collected by Aether:** **None.** Zero events, zero error stack traces, zero system/OS fingerprints, and zero user/workspace telemetry leave the machine to Prashna Labs.
- **Outbound Payloads in Functional Provider/Feature Requests:**
  1. **Anthropic Usage GET** (`crates/agent/src/usage.rs:566-570`): `Authorization: Bearer <claude_oauth_token>`, `anthropic-version: 2023-06-01`, `User-Agent: Claude-Code/0.1.0` (no request body).
  2. **ChatGPT/Codex Usage GET** (`crates/agent/src/usage.rs:608-611`): `Authorization: Bearer <codex_access_token>`, `User-Agent: codex/0.163.0` (no request body).
  3. **OpenRouter Credits GET** (`crates/agent/src/usage.rs:644-646`): `Authorization: Bearer <openrouter_api_key>`, `User-Agent: aether/0.1.0`.
  4. **LiteLLM Pricing GET** (`crates/agent/src/usage_history/pricing.rs:121`): `Accept: application/json` (static file download; no model names or usage stats sent).
  5. **DuckDuckGo POST** (`crates/agent/src/tools/web_search/duckduckgo.rs:22-30`): Form parameter `q=<search_query>` with Safari `User-Agent`, `Origin`, `Referer` headers.
  6. **Hacker News GET** (`crates/app/src/hackernews/client.rs:90,111`): `User-Agent: Aether-HackerNews/1.0`.
  7. **Optional User Webhook POST** (`crates/app/src/notify.rs:83-98`; `crates/app/src/app/chat/send.rs:587-605`): Only when `notify_webhook` is configured; sends JSON `{"title": <chat title or prompt preview <=60 chars>, "body": <reply preview <=180 chars>, "chat": <chat_id>, "state": "done"|"request"}`.

---

## 6. Docs / Privacy Policy vs Code Discrepancies

- **No Dedicated Privacy Policy:** Neither `PRIVACY.md` nor a privacy policy section in `README.md` exists.
- **Undisclosed Automatic Cross-Tool Auth Reads & Quota Polling:** `README.md:20-23,42` describes driving already-signed-in CLIs and showing usage across providers, and `SECURITY.md:38-39` treats leaking keyring entries/credentials into network requests as a security bug, while `README.md` does **not** disclose that Aether automatically reads `"Claude Code-credentials"` from the OS keychain (potentially triggering a macOS password prompt per `crates/agent/src/usage.rs:524`), reads `~/.codex/auth.json` and `~/.claude.json`/`~/.claude/.credentials.json`, and polls Anthropic, ChatGPT, and OpenRouter (even unauthenticated `list_models()`) on every launch without an opt-out toggle. Nor does it disclose the GitHub raw fetch to `BerriAI/litellm` when opening the Usage dashboard (`crates/agent/src/usage_history/pricing.rs:9-10`).
- **Voice On-Device & Security Commitments Verified Accurate:** `README.md:114` (`"Speech turns into text on this device"`) and `scripts/build.py:71` (`"Audio stays on this device"`) match the code (`whisper-rs` offline inference after optional SHA-256-verified HuggingFace model download). `README.md:171-172` (`"Packaging never copies .env or credentials"`) matches `scripts/build.py` and `.github/workflows/build.yml`.
- **Minor Non-Privacy Docs Discrepancies Observed:**
  - `crates/app/src/settings_search.rs:134` indexes `"webhook"` under `"notifications"`, and `README.md:125-127` documents `notify_webhook`, but `crates/app/src/settings_view.rs:492-557` exposes no UI field for `notify_webhook` (requires manual `settings.json` edit).
  - `README.md:46-47` states Aether runs on macOS and Linux (omitting Windows), whereas `README.md:191`, `.github/workflows/build.yml:130`, `docs/releases/alpha-0.1.1.md:61`, and `crates/app/src/main.rs:2` explicitly build and ship `Aether-<version>-windows-x86_64.zip`.
  - `README.md:171` says `"Development builds can also read a local .env"`, whereas `crates/app/src/main.rs:54` runs `dotenvy::dotenv().ok()` unconditionally in release builds as well.
  - `features.md:1-5` is an unimplemented/stale scratch note (`"1. aether agent (beta feature)"`, `~/.config/aether/`) whereas actual config uses `aether_core::config_dir()` (`~/Library/Application Support/aether` on macOS, `%APPDATA%/aether` on Windows, `$XDG_CONFIG_HOME/aether` on Linux per `crates/core/src/paths.rs:11-32`).

---

## 7. Zero-Telemetry / Negative-Check Verification

All checks below were executed read-only against `repos/aether` at commit `a9a941541dbd3a744ad9276afa56ddbdeb638632`.

### Exact Verification Commands & Results

1. **Cargo Dependencies & Lockfile Scan for Telemetry / Analytics / Crash SDKs:**
   ```sh
   rg -n -i 'telemetry|sentry|posthog|mixpanel|amplitude|datadog|bugsnag|\brollbar\b|honeycomb|opentelemetry|otlp|plausible|umami|fathom|matomo|statsig|launchdarkly|minidump|segment|snowplow|rudderstack|appcenter|bugly|crashlytics' \
     repos/aether/Cargo.lock \
     repos/aether/Cargo.toml \
     repos/aether/crates --glob '**/Cargo.toml' --glob '*Cargo.lock'
   ```
   - **Result:** Exit code `0`, 13 lines total, **100% false positives**:
     - `adk-telemetry` (`Cargo.lock:101,141,254,272`): inspected at `Cargo.lock:253-262` — depends solely on `thiserror`, `tracing`, and `tracing-subscriber` (no OTEL/HTTP exporters).
     - `unicode-segmentation` (`Cargo.lock:1635,...`): substring match on `segment`.
     - Zero matches for `sentry`, `posthog`, `opentelemetry`, `otlp`, `datadog`, `minidump`, `mixpanel`, etc.

2. **Source Code Negative Search for Telemetry / Analytics / Crash / Machine IDs:**
   ```sh
   rg -n -i '\b(telemetry|privacy|analytics|sentry|posthog|mixpanel|amplitude|datadog|bugsnag|rollbar|honeycomb|opentelemetry|otlp|plausible|umami|fathom|matomo|statsig|launchdarkly|minidump)\b|crash.?report|machine.?id|device.?id|install.?id|anonymous' \
     repos/aether/crates \
     repos/aether/vendor/gpui-base/src \
     --glob '!*.dic' --glob '!*.aff' --glob '!*.svg' --glob '!*spelling*' --glob '!*tests*'
   ```
   - **Result:** Exit code `0`, **9 matches total — all verified false positives** (note: `vendor/gpui-base/README.md:136-159` also contains an unused `Checkbox::new("telemetry")` UI doc example):
     - `crates/ui/src/chat/context_ring.rs:33`: doc comment `/// Clicking it opens a popover with the exact counts and quota telemetry.` (local UI display).
     - `crates/app/src/app/composer/mod.rs:142`: doc comment `/// Context ring with usage telemetry, plus the used and total token counts.` (local UI display).
     - `crates/app/src/sysinfo.rs:38`: macOS `vm_stat` parser reading `"Anonymous pages:"` RAM metric.
     - `crates/app/src/suggest/slash.rs:189`, `crates/app/src/app/chat/commands.rs:78,120`, `crates/app/src/palette.rs:629`: UI labels/keywords (`"Open full analytics dashboard"`) for the local **Usage** page.
     - `crates/app/src/app/composer/voice.rs:19`: macOS user error string `"Check System Settings > Privacy & Security > Microphone."`.
     - `crates/app/src/hackernews/client.rs:48`: fallback string `"anonymous"` when an HN API item omits its `by` author field.

3. **Auto-Updater & Version Check Scan:**
   ```sh
   rg -n -i 'auto.?update|check.?update|releases/latest|appcast|sparkle|updater|update_url|PI_SKIP_VERSION_CHECK' \
     repos/aether/crates \
     repos/aether/vendor/gpui-base/src \
     --glob '!*.dic' --glob '!*.aff' --glob '!*.svg' --glob '!*spelling*' --glob '!*tests*'
   ```
   - **Result:** Exit code `0`, **2 matches only** (`crates/driver/src/catalog.rs:188,514`), both setting `.env("PI_SKIP_VERSION_CHECK", "1")` to suppress the external Pi CLI's update check.

4. **All Outbound URLs & `reqwest` Clients in Non-Test Crate Sources:**
   ```sh
   rg -n 'https?://' repos/aether/crates --glob '!*.svg' --glob '!*tests*' --glob '!*.dic' --glob '!*.aff' --glob '!*spelling*'
   rg -n 'reqwest|ureq|hyper|Client::new|Client::builder' repos/aether/crates repos/aether/vendor/gpui-base/src --glob '!*tests*'
   rg -n -i 'prashnalabs|durbarai|utm_' repos/aether/crates repos/aether/vendor/gpui-base repos/aether/README.md repos/aether/Cargo.toml repos/aether/scripts repos/aether/packaging
   ```
   - **Result:** Confirmed zero PrashnaLabs/DurbarAI endpoints in code (`prashnalabs.com` / `?utm_source=aether&utm_medium=readme` appear solely in `README.md:9-12,306-308`, `Cargo.toml:9`, `PKGBUILD:6`, and `scripts/build.py:130`), and accounted for 100% of `reqwest::Client` instances and URLs in Section 4.

---

```json
{
  "slug": "aether",
  "platform": "Aether",
  "group": "A",
  "repo": "PrashnaLabs/aether",
  "sha": "a9a941541dbd3a744ad9276afa56ddbdeb638632",
  "commit_date": "2026-11-13T21:05:01+05:45",
  "license": "MIT",
  "status": "Active",
  "default_state": "Zero telemetry/analytics/crash-reporting/auto-update; non-telemetry provider quota polling, OpenRouter model listing, and local agent/credential discovery run automatically on startup",
  "opt_out_controls": "None needed for telemetry (none exists); no toggle to disable automatic startup/10m provider quota polling, startup OpenRouter list_models(), or LiteLLM pricing fetch on Usage view (Bottom Bar toggles hide UI only); web_search defaults Ask and can be disabled in Settings; notify_webhook defaults off",
  "do_not_track": "Not checked (no telemetry exists)",
  "opt_out_bypasses": "None for telemetry; hiding Usage entries in Settings > Bottom Bar only hides UI and does not stop background quota polling or credential reads",
  "persistent_ids": "None (getrandom used only for ephemeral orchestrator tokens, ephemeral Claude session UUIDs, and temp dir names)",
  "hashing": "None for identifiers or PII (sha256 verifies Whisper ggml-base.en.bin download integrity; DefaultHasher names local temp MCP config file from ephemeral token)",
  "external_auth_reads": "Unconditional startup & 10m reads for quota display (never sent to developer): OS keychain 'Claude Code-credentials', ~/.claude/.credentials.json, ~/.claude.json, ~/.config/claude/credentials.json, ANTHROPIC_AUTH_TOKEN, CLAUDE_CODE_TOKEN, $CODEX_HOME/auth.json or ~/.codex/auth.json, local OpenCode service.json; also reads external CLI sessions/skills/configs/caches",
  "telemetry_endpoints": "None (non-telemetry feature endpoints only: api.anthropic.com/api/oauth/usage, chatgpt.com/backend-api/wham/usage, openrouter.ai/api/v1/{models,credits}, raw.githubusercontent.com/BerriAI/litellm, optional huggingface.co whisper model, html.duckduckgo.com, hacker-news.firebaseio.com, user notify_webhook)",
  "ip_geo_handling": "No telemetry or ingest pipeline; direct TLS/HTTP client connections to user-integrated provider/feature endpoints only",
  "custom_model_handling": "Kept locally in settings.json and sent only to user's OpenRouter API during chat turns; LiteLLM pricing JSON is fetched statically and matched locally via normalize_model_name() without transmitting model names",
  "docs_discrepancies": "No privacy policy; README does not disclose automatic startup/10m keychain/auth file reads (nor macOS keychain prompt risk), background quota polling, unauthenticated OpenRouter list_models(), or GitHub LiteLLM pricing fetch; Settings search mentions 'webhook' though UI lacks notify_webhook field; README omits Windows despite Windows release builds; main.rs calls dotenvy unconditionally",
  "privacy_rating": "A",
  "one_line_verdict": "Zero product telemetry, zero analytics/crash SDKs, zero persistent IDs, and zero update pings, though it automatically reads Claude/Codex local auth (including keychain) and polls provider quota/model endpoints on startup without an opt-out toggle."
}
```
