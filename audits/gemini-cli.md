# Forensic Telemetry & Privacy Audit: Google Gemini CLI

## Header Metadata
- **slug:** `gemini-cli`
- **platform:** `Google Gemini CLI`
- **group:** `B`
- **repo:** `google-gemini/gemini-cli` (`https://github.com/google-gemini/gemini-cli`)
- **license:** `Apache-2.0`
- **status:** `Active`
- **branch:** `main`
- **sha:** `08e9a20f749aefcbda38622ac757899c3d6216e0`
- **commit_date:** `2026-11-24T23:51:44Z`
- **commit_subject:** `fix(core): normalize hook command names with leading environment variable assignments (#30448)`

---

## 1. Default State & Opt-Out Mechanism

Gemini CLI operates **four distinct outbound telemetry / data-collection / update channels**, with two active by default and asymmetric, fragmented opt-out controls:

1. **Google Clearcut (`ClearcutLogger`) — ON by default:**
   - Controlled exclusively by `privacy.usageStatisticsEnabled`, which defaults to `true` in both the CLI settings schema (`packages/cli/src/config/settingsSchema.ts:974-982`) and `Config` (`packages/core/src/config/config.ts:1112`).
   - `ClearcutLogger.getInstance(config)` returns an active singleton whenever `config?.getUsageStatisticsEnabled()` is truthy (`packages/core/src/telemetry/clearcut-logger/clearcut-logger.ts:340-347`).
   - **Hard-to-discover / incomplete opt-out:**
     - `privacy.usageStatisticsEnabled` is marked `showInDialog: false` (`packages/cli/src/config/settingsSchema.ts:972,981`), hiding it from the `/settings` interactive dialog.
     - No CLI flag (`packages/cli/src/config/config.ts:846-860,1104-1105`) and no environment variable controls `usageStatisticsEnabled`; `DO_NOT_TRACK` is completely ignored (`0` matches repo-wide).
     - The `/privacy` slash command does **not** control Clearcut (`packages/cli/src/ui/commands/privacyCommand.ts:13-21`); it only displays terms or toggles Code Assist Free-tier training (`packages/cli/src/ui/privacy/PrivacyNotice.tsx:25-35`).
     - Setting `telemetry.enabled: false` or `GEMINI_TELEMETRY_ENABLED=false` disables **only OpenTelemetry**, leaving Clearcut active.
     - **Lifecycle quirk when OTEL is off:** `runExitCleanup` calls `shutdownTelemetry(config)` only if `isTelemetrySdkInitialized()` is true (`packages/cli/src/utils/cleanup.ts:140-145`), and `ClearcutLogger.getInstance()?.shutdown()` (`END_SESSION` + flush) is only invoked inside `shutdownTelemetry` (`packages/core/src/telemetry/sdk.ts:475`), so short-lived sessions (< 60s flush interval in `packages/core/src/telemetry/clearcut-logger/clearcut-logger.ts:249,517-520`) flush `START_SESSION` immediately (`clearcut-logger.ts:734-740`) and extension lifecycle events immediately, while `END_SESSION` is dropped when OTEL is disabled.

2. **OpenTelemetry (`OTEL_*` / `GEMINI_TELEMETRY_*`) — OFF by default, `logPrompts` ON by default:**
   - `telemetry.enabled` defaults to `false`, `telemetry.traces` to `false`, `telemetry.target` to `'local'`, `telemetry.otlpEndpoint` to `'http://localhost:4317'`, `telemetry.otlpProtocol` to `'grpc'`, `telemetry.useCollector` to `false`, `telemetry.useCliAuth` to `false` (`packages/core/src/config/config.ts:1100-1111`; `packages/core/src/telemetry/index.ts:7-15`; `packages/core/src/config/config.ts:2922-2973`).
   - `Config` initializes the OTEL `NodeSDK` only when `this.telemetrySettings.enabled` is true (`packages/core/src/config/config.ts:1423-1426`; `packages/core/src/telemetry/sdk.ts:188-194`). `bufferTelemetryEvent` discards queued events if `!config.getTelemetryEnabled()` (`packages/core/src/telemetry/sdk.ts:119-129`).
   - **Critical default:** `telemetry.logPrompts` defaults to **`true`** (`packages/core/src/config/config.ts:1107,2930-2932`). Enabling OTEL (e.g., `GEMINI_TELEMETRY_ENABLED=true`) without also setting `GEMINI_TELEMETRY_LOG_PROMPTS=false` logs full prompt text, response JSON, tool arguments, and hook inputs/outputs/stdout/stderr immediately.
   - `telemetry` is also hidden from `/settings` (`showInDialog: false`, `packages/cli/src/config/settingsSchema.ts:993`) and configured via `settings.json` or `ALLOWED_TELEMETRY_ENV_KEYS` (`packages/core/src/telemetry/config.ts:79-92`). Note `--telemetry*` CLI flags do not exist in `packages/cli/src/config/config.ts`.

3. **Code Assist Server Telemetry & Free-Tier Training Opt-Out (`LOGIN_WITH_GOOGLE` / `COMPUTE_ADC`) — Unconditional / Default ON:**
   - `CodeAssistServer.generateContentStream` and `generateContent` call `recordConversationOffered` (`packages/core/src/code_assist/server.ts:154-161,225-232`), and interactive/non-interactive/agent loops call `recordToolCallInteractions` (`packages/cli/src/ui/hooks/useGeminiStream.ts:404`, `packages/cli/src/nonInteractiveCli.ts:623`, `packages/core/src/agent/legacy-agent-session.ts:329`), posting `recordCodeAssistMetrics` (`packages/core/src/code_assist/server.ts:376-413`; `packages/core/src/code_assist/telemetry.ts:33-85`) **without checking `usageStatisticsEnabled` or `telemetry.enabled`**.
   - `listExperiments` (`packages/core/src/code_assist/experiments/experiments.ts:25-63`) and 5-minute polling `fetchAdminControls` (`packages/core/src/code_assist/admin/admin_controls.ts:186-218`) also run for authenticated Code Assist sessions.
   - For Free-tier Google accounts (`UserTierId.FREE`), `freeTierDataCollectionOptin` defaults to **`true`** both on fallback (`?? true`) and on HTTP 404 (`packages/cli/src/ui/hooks/usePrivacySettings.ts:153-177`). Opt-out requires running `/privacy` and choosing *"No, deny permission"* (`packages/cli/src/ui/privacy/CloudFreePrivacyNotice.tsx:103-181`).

4. **Silent Opt-Out Bypasses & Sub-Package Defaults:**
   - **A2A Server bypass (`packages/a2a-server`):** Although `packages/a2a-server/src/config/settings.ts:155-157,248,766-771` defines `privacy.usageStatisticsEnabled`, `packages/a2a-server/src/config/config.ts:351-407` **omits `usageStatisticsEnabled`** from `ConfigParameters`, forcing Clearcut and `x-gemini-api-privileged-user-id` ON even when disabled in `settings.json`.
   - **SDK bypass (`packages/sdk`):** `GeminiCliAgentOptions` (`packages/sdk/src/types.ts:31-80`) and `GeminiCliSession` (`packages/sdk/src/session.ts:74-95`) expose no `usageStatisticsEnabled` or `telemetry` option, leaving Clearcut ON by default for all SDK consumers.
   - **OTEL `bufferTelemetryEvent` omission:** `logChatCompression` emits `logger.emit(logRecord)` and `recordChatCompressionMetrics(...)` directly without wrapping in `bufferTelemetryEvent(..., config)` (`packages/core/src/telemetry/loggers.ts:458-475`), though the OTEL `NodeSDK` itself is uninitialized unless `telemetry.enabled` is true.
   - **Direct API ID header on custom gateways:** When `usageStatisticsEnabled` is true, `contentGenerator.ts:336-356` attaches `x-gemini-api-privileged-user-id: <installationId>` even when `AuthType.GATEWAY` / `GOOGLE_GEMINI_BASE_URL` / `GOOGLE_VERTEX_BASE_URL` points to a third-party endpoint.
   - **Bundled `chrome-devtools-mcp`:** Browser Agent passes `--no-usage-statistics --no-performance-crux` **only** when `!this.config.getUsageStatisticsEnabled()` (`packages/core/src/agents/browser/browserManager.ts:668-671`), leaving Chrome CrUX / MCP telemetry ON by default during browser agent runs.
   - **Auto-update & Marketplace checks:** CLI checks npm via `latest-version` on interactive startup unless `general.enableAutoUpdateNotification` is `false` or `DEV === 'true'` (`packages/cli/src/interactiveCli.tsx:192-200`, `packages/cli/src/ui/utils/updateCheck.ts:56-79`), checks GitHub/Git extension updates (`packages/cli/src/ui/hooks/useExtensionUpdates.ts:94-110`, `packages/cli/src/config/extensions/update.ts:192-230`), and `vscode-ide-companion` unconditionally checks VS Code Marketplace on activation outside managed surfaces (`packages/vscode-ide-companion/src/extension.ts:40-73,118-123`).

---

## 2. External Auth / Local Non-Workspace File Reads

- **Gemini CLI's own state (`~/.gemini/` or `GEMINI_CLI_HOME/.gemini/`):**
  - `homedir()` resolves `GEMINI_CLI_HOME || os.homedir()` (`packages/core/src/utils/paths.ts:22-28`). Under macOS `sandbox-exec`, runtime storage uses `~/.cache/.gemini` (`packages/core/src/config/storage.ts:92-108`).
  - Reads/writes `~/.gemini/installation_id` (`packages/core/src/config/storage.ts:123-125`), `~/.gemini/google_accounts.json` (`packages/core/src/config/storage.ts:127-129`), `~/.gemini/oauth_creds.json` (`packages/core/src/config/storage.ts:255-257`), `~/.gemini/gemini-credentials.json` (AES-256-GCM encrypted via `FileKeychain`, `packages/core/src/services/fileKeychain.ts:18-45`), `~/.gemini/mcp-oauth-tokens.json` (`packages/core/src/config/storage.ts:70-72`; `packages/core/src/mcp/oauth-token-storage.ts:47-78`), `~/.gemini/settings.json` (`packages/core/src/config/storage.ts:78-80`), `~/.gemini/GEMINI.md`, and `~/.agents/skills` (`packages/core/src/config/storage.ts:62-68,146-148`).
  - Also checks OS system-wide enterprise settings/defaults/policies at `/Library/Application Support/GeminiCli/` (macOS), `C:\ProgramData\gemini-cli\` (Windows), `/etc/gemini-cli/` (Linux), or `GEMINI_CLI_SYSTEM_SETTINGS_PATH` / `GEMINI_CLI_SYSTEM_DEFAULTS_PATH` (`packages/cli/src/config/settings.ts:108-129`; `packages/core/src/config/storage.ts:174-193`).
  - Upward `.env` traversal from `cwd` walks to `/` and falls back to `~/.gemini/.env` (if trusted) and `~/.env` (`packages/cli/src/config/settings.ts:617-653`).
  - Browser Agent uses `~/.gemini/cli-browser-profile` (`packages/core/src/agents/browser/browserManager.ts:660-666`) or `--autoConnect` to the user's signed-in Chrome profile (`packages/core/src/agents/browser/browserManager.ts:641-646`).
- **Google Cloud standard auth / ADC:**
  - `fetchCachedCredentialsList` reads `process.env['GOOGLE_APPLICATION_CREDENTIALS']` after `~/.gemini/oauth_creds.json` for Google OAuth/ADC authentication (`packages/core/src/code_assist/oauth2.ts:739-764`), plus `google-auth-library` ADC when `AuthType.COMPUTE_ADC` or GCP OTEL exporter is active (`packages/core/src/telemetry/sdk.ts:292-303`).
- **IDE / OS Discovery & External Tool Paths:**
  - `getIdeConnectionDir` inspects `${os.tmpdir()}/gemini-ide-${uid}` (POSIX) or `${os.tmpdir()}\gemini\ide` (Windows) (`packages/core/src/utils/paths.ts:44-49`).
  - `packages/cli/src/commands/hooks/migrate.ts:183-227` reads `${cwd}/.claude/settings.json` or `settings.local.json` **only when the user explicitly runs the hook migration command** (`gemini hooks migrate --from-claude`); it is never read automatically or for telemetry.
  - **Negative check:** Zero reads of `~/.claude`, `~/.codex`, `~/.cursor`, `~/.windsurf`, `~/.aider`, `~/.aws/credentials`, `~/.netrc`, `~/.ssh`, or `gh/hosts.yml` in normal startup or telemetry code.

---

## 3. Persistent Device / Install IDs & Hashing

1. **Persistent Installation UUID (`installation_id`):**
   - `InstallationManager.getInstallationId()` reads `~/.gemini/installation_id`; if absent, generates a persistent UUIDv4 via `randomUUID()` and writes it to disk (`packages/core/src/utils/installationManager.ts:41-50`; `packages/core/src/config/storage.ts:123-125`), falling back to hardcoded `'123456789'` on I/O error (`packages/core/src/utils/installationManager.ts:51-57`).
   - Sent in:
     - **Clearcut:** `logEvent.client_install_id` when no cached Google email exists (`packages/core/src/telemetry/clearcut-logger/clearcut-logger.ts:497-501`).
     - **OpenTelemetry:** `'installation.id'` via `getCommonAttributes(config)` on **all** OTEL log records and metrics, **simultaneously** with `'user.email'` when present (`packages/core/src/telemetry/telemetryAttributes.ts:15-29`).
     - **HTTP API header:** `x-gemini-api-privileged-user-id` on Gemini API, Vertex AI, and `GATEWAY` / `GOOGLE_GEMINI_BASE_URL` / `GOOGLE_VERTEX_BASE_URL` requests when `usageStatisticsEnabled` is true (`packages/core/src/core/contentGenerator.ts:336-356`).

2. **Plaintext Google Email & Lifetime Account Counter (`google_accounts.json`):**
   - During Google OAuth login, `fetchAndCacheUserInfo` queries `https://www.googleapis.com/oauth2/v2/userinfo` and caches `email` in `~/.gemini/google_accounts.json` (`packages/core/src/code_assist/oauth2.ts:596-597,788-818`).
   - `UserAccountManager` stores `{ active, old: string[] }` (`packages/core/src/utils/userAccountManager.ts:12-15,100-147`) and exposes `getCachedGoogleAccount()` and `getLifetimeGoogleAccounts()`.
   - `ClearcutLogger` attaches **plaintext `logEvent.client_email = email`** on every Clearcut event whenever `active` is set, plus `GEMINI_CLI_GOOGLE_ACCOUNTS_COUNT` on every event (`packages/core/src/telemetry/clearcut-logger/clearcut-logger.ts:420,497-515,2099-2101`).
   - OTEL attaches **plaintext `'user.email'`** alongside `'installation.id'` (`packages/core/src/telemetry/telemetryAttributes.ts:16,21-23`).

3. **Session & Prompt IDs:**
   - Per-run `session.id` (`GEMINI_CLI_SESSION_ID` in Clearcut, `'session.id'` in OTEL resource/attributes, `trajectoryId` in Code Assist `recordConversationOffered`, and `sessionId` in `/bug` URL), `prompt_id` (`GEMINI_CLI_PROMPT_ID`), and `agent_id` (`packages/core/src/telemetry/clearcut-logger/clearcut-logger.ts:2089-2105`; `packages/core/src/telemetry/telemetryAttributes.ts:20`; `packages/core/src/code_assist/server.ts:160,231`; `packages/cli/src/ui/commands/bugCommand.ts:66`).

4. **Hashing & Key Derivation:**
   - **Extension names & IDs (unsalted SHA-256):** `hashValue()` computes raw `createHash('sha256').update(value).digest('hex')` (`packages/cli/src/config/extension-manager.ts:1593-1595`). `getExtensionId()` hashes `https://github.com/${owner}/${repo}` for GitHub extensions or `installMetadata?.source ?? config.name` otherwise (`packages/cli/src/config/extension-manager.ts:1569-1591`). Because SHA-256 is unsalted, public extension names and GitHub repo URLs are trivially reversible by precomputation; moreover, OTEL logs plaintext `extension_name`, `extension_source`, and `extensions` (`packages/core/src/telemetry/types.ts:148,1596-1599`).
   - **GitHub repository name (unsalted SHA-256):** `ClearcutLogger` hashes `process.env['GITHUB_REPOSITORY']` with unsalted `createHash('sha256').update(ghRepositoryName).digest('hex')` (`packages/core/src/telemetry/clearcut-logger/clearcut-logger.ts:332-337,454-459`), while logging plaintext `GH_WORKFLOW_NAME`, `GITHUB_EVENT_NAME`, `GH_PR_NUMBER`, `GH_ISSUE_NUMBER`, and `GH_CUSTOM_TRACKING_ID` (`clearcut-logger.ts:422-487`).
   - **FileKeychain local key derivation:** Local AES-256-GCM encryption for `~/.gemini/gemini-credentials.json` derives its key via `scryptSync('gemini-cli-oauth', `${hostname}-${username}-gemini-cli`, 32)` — local-only, not transmitted (`packages/core/src/services/fileKeychain.ts:24-45`).
   - **No hardware UUID / MAC hashing:** Does not read `/etc/machine-id` or MAC addresses, though `START_SESSION` sends hardware CPU/RAM/GPU strings (`clearcut-logger.ts:706-730`).

---

## 4. Exact Outbound Telemetry / Analytics / Crash / OTEL / Relay / Update Endpoints

| Channel | Endpoint / Target | Trigger & Gate | File Reference |
| :--- | :--- | :--- | :--- |
| **Google Clearcut** | `https://play.googleapis.com/log?format=json&hasfast=true` (`POST`, `log_source_name: 'CONCORD'`, `application: 102`) | **ON by default** (`privacy.usageStatisticsEnabled: true`); flushes immediately on `START_SESSION` / extension ops and every 60s otherwise | `packages/core/src/telemetry/clearcut-logger/clearcut-logger.ts:244,249,340-347,546-563,734-740` |
| **OTEL OTLP (Local / Custom)** | `http://localhost:4317` (gRPC default) or `GEMINI_TELEMETRY_OTLP_ENDPOINT` / `OTEL_EXPORTER_OTLP_ENDPOINT` (`/v1/traces`, `/v1/logs`, `/v1/metrics` for HTTP) | **OFF by default** (`telemetry.enabled: false`); exports spans, logs, metrics when `enabled: true` & `!outfile` & `target === 'local'` | `packages/core/src/telemetry/index.ts:12-15`; `packages/core/src/telemetry/sdk.ts:263-370` |
| **OTEL Direct GCP** | Cloud Trace (`cloudtrace.googleapis.com`), Cloud Monitoring (`custom.googleapis.com/gemini_cli`), Cloud Logging (`gemini_cli` log) | **OFF by default**; active when `telemetry.enabled: true` & `target === 'gcp'` & `!useCollector` | `packages/core/src/telemetry/sdk.ts:275-303`; `packages/core/src/telemetry/gcp-exporters.ts:25-109` |
| **Code Assist Metrics / Experiments / Admin / Privacy API** | `https://cloudcode-pa.googleapis.com/v1internal:{recordCodeAssistMetrics,listExperiments,fetchAdminControls,getCodeAssistGlobalUserSetting,setCodeAssistGlobalUserSetting}` | **Unconditional** when signed in (`LOGIN_WITH_GOOGLE` / `COMPUTE_ADC`) — `recordConversationOffered` & `recordToolCallInteractions` ignore `usageStatisticsEnabled` & `telemetry.enabled`; 5-min admin polling | `packages/core/src/code_assist/server.ts:73,154-161,225-232,315-413`; `packages/core/src/code_assist/telemetry.ts:33-85`; `packages/core/src/code_assist/admin/admin_controls.ts:193-217` |
| **Google OAuth UserInfo** | `https://www.googleapis.com/oauth2/v2/userinfo` (`GET`) & redirects `developers.google.com/gemini-code-assist/auth_{success,failure}_gemini` | During Google OAuth login; fetches `email` and stores in `google_accounts.json` | `packages/core/src/code_assist/oauth2.ts:95-98,596-611,788-818` |
| **Model Inference Headers** | `generativelanguage.googleapis.com`, Vertex REP endpoints, or `GOOGLE_GEMINI_BASE_URL` / `GOOGLE_VERTEX_BASE_URL` | Sends `x-gemini-api-privileged-user-id: <installationId>` and `User-Agent` (with `version`, `model`, `platform`, `arch`, `surface`) when `usageStatisticsEnabled` is true | `packages/core/src/core/contentGenerator.ts:130-131,245-273,336-356` |
| **Browser Agent MCP (`chrome-devtools-mcp`)** | Google Chrome CrUX / `chrome-devtools-mcp` usage telemetry | **ON by default** when Browser Agent launches; `--no-usage-statistics --no-performance-crux` passed only when `!usageStatisticsEnabled` | `packages/core/src/agents/browser/browserManager.ts:668-671` |
| **CLI npm Update Check** | npm registry via `latest-version` package (`@google/gemini-cli` `latest` & `nightly` tags) | Interactive startup when `general.enableAutoUpdateNotification` is true & `DEV !== 'true'` | `packages/cli/src/interactiveCli.tsx:192-200`; `packages/cli/src/ui/utils/updateCheck.ts:7,56-132` |
| **Extension Update Checks** | `https://api.github.com/repos/${owner}/${repo}/releases*` & `git ls-remote` on extension remotes | Interactive startup via `useExtensionUpdates` (`checkForAllExtensionUpdates`) | `packages/cli/src/ui/hooks/useExtensionUpdates.ts:94-110`; `packages/cli/src/config/extensions/update.ts:192-230`; `packages/cli/src/config/extensions/github.ts:162-322` |
| **VS Code Extension Update** | `https://marketplace.visualstudio.com/_apis/public/gallery/extensionquery` (`POST`) | Unconditional on VS Code extension activation outside managed surfaces | `packages/vscode-ide-companion/src/extension.ts:40-73,118-123` |
| **Manual Bug Report URL** | `https://github.com/google-gemini/gemini-cli/issues/new` (browser `open()`, configurable via `advanced.bugCommand.urlTemplate`) | User-invoked `/bug` command only; exports local history file without auto-uploading | `packages/cli/src/ui/commands/bugCommand.ts:85-135` |

**No Sentry / PostHog / Crash-Upload Endpoint:** Gemini CLI has zero third-party crash/analytics SDKs (`packages/core/package.json:25-96`, `packages/cli/package.json:32-73`) and does not automatically upload crash reports or heap snapshots.

---

## 5. Exact Data & Properties Collected

- **Clearcut Payload & All-Event Default Fields (`clearcut-logger.ts`):**
  - Envelope: `client_email` (plaintext cached Google email) OR `client_install_id` (UUIDv4), `application: 102`, `console_type: 'GEMINI_CLI'`, `event_time_ms`, optional `exp.gws_experiment` (`clearcut-logger.ts:363-372,489-502`).
  - Every `createBasicLogEvent`: `GEMINI_CLI_SURFACE`, `GEMINI_CLI_VERSION`, `GEMINI_CLI_GIT_COMMIT_HASH`, `GEMINI_CLI_OS` (`process.platform`), plus CI env vars if present (`GH_WORKFLOW_NAME`, SHA-256 of `GITHUB_REPOSITORY`, `GITHUB_EVENT_NAME`, `GH_PR_NUMBER`, `GH_ISSUE_NUMBER`, `GH_CUSTOM_TRACKING_ID`) (`clearcut-logger.ts:416-487`).
  - Every `createLogEvent` (`addDefaultFields`): `GEMINI_CLI_SESSION_ID`, `GEMINI_CLI_AUTH_TYPE`, `GEMINI_CLI_GOOGLE_ACCOUNTS_COUNT` (count of current + historical emails), `GEMINI_CLI_PROMPT_ID`, `GEMINI_CLI_NODE_VERSION`, `GEMINI_CLI_USER_SETTINGS` (all top-level boolean properties on `Config` via `safeJsonStringifyBooleanValuesOnly`, `packages/core/src/utils/safeJsonStringify.ts:57-70`), `GEMINI_CLI_INTERACTIVE`, `GEMINI_CLI_ACTIVE_APPROVAL_MODE`, `GEMINI_CLI_EXPERIMENT_IDS` (`clearcut-logger.ts:2086-2133`).
- **Clearcut Event Catalog (`EventNames` at `clearcut-logger.ts:82-142`):**
  - `START_SESSION` (+ cached onto `this.sessionData` and appended to subsequent events!): `model`, `embedding_model`, `sandbox_enabled`, `core_tools_enabled`, `approval_mode`, `api_key_enabled`, `vertex_ai_enabled` (pushed 3x), `debug_enabled`, **plaintext `mcp_servers` and `mcp_tools` names/counts**, `extensions_count`, **unsalted SHA-256 `extension_ids`**, `worktree_active`, plus hardware fingerprint: `os.cpus()[0].model`, `os.availableParallelism()`, total RAM GB, and GPU controller models via `systeminformation` `si.graphics()` (`clearcut-logger.ts:267-290,610-730`).
  - Lifecycle, behavioral & operational events: `NEW_PROMPT` (`prompt_length`), `TOOL_CALL` (`function_name`, `decision`, `success`, `duration_ms`, `error_type`, `tool_type`, `content_length`, `mcp_server_name`, AI/user added/removed lines/chars, `ask_user` interaction counts, `extension_id`), `FILE_OPERATION` (`tool_name`, `operation`, `lines`, `mimetype`, `extension`, `programming_language`), `API_REQUEST` (`model`), `API_RESPONSE` (`model`, `status_code`, `duration_ms`, 5 token counts, context breakdown including per-tool token map `tool_calls` and `mcp_servers` token total), `API_ERROR` (`model`, `error_type`, `status_code`, `duration_ms`), `END_SESSION`, `FLASH_FALLBACK`, `RIPGREP_FALLBACK`, `LOOP_DETECTED`, `LOOP_DETECTION_DISABLED`, `NEXT_SPEAKER_CHECK`, `SLASH_COMMAND`, `REWIND`, `MALFORMED_JSON_RESPONSE`, `IDE_CONNECTION`, `CHAT_COMPRESSION`, `CONVERSATION_FINISHED`, `INVALID_CHUNK` (includes `error_message`), `CONTENT_RETRY`, `CONTENT_RETRY_FAILURE`, `RETRY_ATTEMPT`, `EXTENSION_{INSTALL,UNINSTALL,UPDATE,ENABLE,DISABLE}`, `TOOL_OUTPUT_{TRUNCATED,MASKING}`, `MODEL_ROUTING`, `MODEL_SLASH_COMMAND`, `EDIT_{STRATEGY,CORRECTION}`, `AGENT_{START,FINISH}`, `RECOVERY_ATTEMPT`, `WEB_FETCH_FALLBACK_ATTEMPT`, `LLM_LOOP_CHECK`, `HOOK_CALL` (`hook_event_name`, `duration_ms`, `success`, `exit_code`), `APPROVAL_MODE_{SWITCH,DURATION}`, `PLAN_EXECUTION`, `KEYCHAIN_AVAILABILITY`, `TOKEN_STORAGE_INITIALIZATION`, `ONBOARDING_{START,SUCCESS}`, `STARTUP_STATS` (phase timings, `os_platform`, `os_release`, `is_docker`), Billing (`CREDITS_USED`, `OVERAGE_OPTION_SELECTED`, `EMPTY_WALLET_MENU_SHOWN`, `CREDIT_PURCHASE_CLICK`), Browser Agent (`CONNECTION`, `VISION_STATUS`, `TASK_OUTCOME`, `CLEANUP`), and `CONSECA_{POLICY_GENERATION,VERDICT}`. *(Note: `KITTY_SEQUENCE_OVERFLOW` and `EventMetadataKey.GEMINI_CLI_RESEARCH_*` 66–71 exist in enums only and have zero callers in `packages/`.)*
- **IP Address & Geolocation:**
  - Zero client-side IP lookup, timezone/locale harvesting, or geo-API calls (`0` matches in `packages/`/`scripts/`). However, **no IP suppression or strip proxy** is implemented: Clearcut (`play.googleapis.com/log`), Code Assist (`cloudcode-pa.googleapis.com`), OAuth, OTLP/GCP exporters, GitHub, npm, and VS Code Marketplace receive the user's client IP directly at the network layer. Note `ClearcutLogger.getProxyAgent()` (`clearcut-logger.ts:2136-2146`) is dead/unused code, though `Config` configures `setGlobalProxy` when `HTTP_PROXY`/`HTTPS_PROXY`/`ALL_PROXY` is set (`packages/core/src/config/config.ts:1433-1435`).
- **Custom / Private Model Names:**
  - Sent **verbatim without redaction or allowlisting** across Clearcut (`START_SESSION`, `API_REQUEST`, `API_RESPONSE`, `API_ERROR`, `CONTENT_RETRY`, `CONTENT_RETRY_FAILURE`, `RETRY_ATTEMPT`, `MODEL_ROUTING`, `MODEL_SLASH_COMMAND`, `LLM_LOOP_CHECK`, Billing events: `packages/core/src/telemetry/clearcut-logger/clearcut-logger.ts:614,889,901,980,1218,1240,1276,1443,1521,1685,1872`), OpenTelemetry (`gen_ai.request.model`, `gen_ai.response.model`, `model`, `model_name`: `packages/core/src/telemetry/metrics.ts:1273-1274`; `packages/core/src/telemetry/types.ts:133,448,535,537,705`), and outbound HTTP `User-Agent` (`packages/core/src/core/contentGenerator.ts:272`). `packages/core/src/telemetry/sanitize.ts` sanitizes hook names only.
- **Prompts, Code, File Paths, System Instructions & Error Stack Traces:**
  - **Code Assist Free tier:** Defaults `freeTierDataCollectionOptin` to `true`, allowing Google to collect prompts, related code, outputs, and edits for 18-month human review (`packages/cli/src/ui/privacy/CloudFreePrivacyNotice.tsx:131-148`; `packages/cli/src/ui/hooks/usePrivacySettings.ts:153-177`).
  - **Conseca -> Clearcut & OTEL:** When `security.enableConseca` is enabled, because `telemetry.logPrompts` defaults to `true`, `conseca-logger.ts:33-47,86-105` sends raw **`user_prompt`**, **`trusted_content`** (tool declarations), **`policy`**, full **`tool_call`** JSON (including tool arguments/paths/code!), and **`verdict_rationale`** directly to **Clearcut** (`play.googleapis.com/log`) as well as OTEL.
  - **ModelRoutingEvent -> Clearcut & OTEL:** Logs `reasoning` (LLM routing explanation) to Clearcut when `logPrompts` is true (`clearcut-logger.ts:1466-1471`), and to OTEL **unconditionally** without checking `logPrompts` (`packages/core/src/telemetry/types.ts:1538-1540`).
  - **OpenTelemetry default `logPrompts: true`:** When OTEL is enabled, `UserPromptEvent.prompt` (`types.ts:222-224`), `ToolCallEvent.function_args` and unfiltered `metadata` (`types.ts:381-395`), `ApiRequestEvent.request_text` (`types.ts:451-453`; full serialized `contents`), `ApiResponseEvent.response_text` (`types.ts:721-723`), and `HookCallEvent` unsanitized `hook_name` (command path), `hook_input`, `hook_output`, `stdout`, and `stderr` (`types.ts:2364-2378`) are all emitted by default.
  - **`logPrompts: false` bypasses in OTEL:**
    - `toGenerateContentConfigAttributes` (`packages/core/src/telemetry/types.ts:647-649`) emits `gen_ai.system_instructions` (which includes workspace/user `GEMINI.md` memory) in `ApiRequestEvent.toSemanticLogRecord`, `ApiErrorEvent.toSemanticLogRecord`, and `ApiResponseEvent.toSemanticLogRecord` (`types.ts:470,567,750`) **without checking `shouldIncludePayloads` or `logPrompts`**.
    - `LoggingContentGenerator` passes `GEN_AI_SYSTEM_INSTRUCTIONS` and `GEN_AI_TOOL_DEFINITIONS` into `runInDevTraceSpan` (`packages/core/src/core/loggingContentGenerator.ts:378-382,472-476`), which sets them when `tracesEnabled` is true regardless of `logPrompts: false` (`packages/core/src/telemetry/trace.ts:137-157`).
    - `ApiErrorEvent`, `ToolCallEvent`, `InvalidChunkEvent`, `HookCallEvent`, and `runInDevTraceSpan` always log raw `error.message` / `span.recordException(meta.error)` (stack traces/paths on Error instances in dev spans) without `logPrompts` gating (`packages/core/src/telemetry/types.ts:397-404,534-538,1351-1353,2380-2383`; `packages/core/src/telemetry/trace.ts:174-182`).
  - **Behavioral / Hardware Profile Fields:** Exact CPU model, core count, RAM GB, GPU model, OS release, Docker status, startup timing phases, approval mode switches/durations, edit lines/chars (`model_added_lines` vs `user_added_lines`), MCP server/tool inventory, IDE/surface, GitHub CI metadata, and billing/credit balance telemetry (`clearcut-logger.ts:706-730,792-835,1725-2055`).

---

## 6. Docs / Privacy Policy vs Code Discrepancies

1. **Direct contradiction — `"We do not collect any personal information, such as your name, email address, or API keys"` (`docs/reference/configuration.md:3147-3148`) vs Clearcut & OTEL email collection:**
   - `docs/reference/configuration.md:3130` describes usage statistics as *"anonymized"* and lines `3147-3148` explicitly promise *"We do not collect any personal information, such as your name, email address, or API keys."*
   - In code, whenever a Google account is signed in (`~/.gemini/google_accounts.json`), `ClearcutLogger.createBasicLogEvent` sets **`logEvent.client_email = email`** on Clearcut events (`packages/core/src/telemetry/clearcut-logger/clearcut-logger.ts:420,497-501`) and OTEL sets **`'user.email': email`** alongside `'installation.id'` (`packages/core/src/telemetry/telemetryAttributes.ts:16,21-23`).
2. **Contradiction — `"We do not log the content of your prompts or the responses"` (`docs/reference/configuration.md:3140-3152`) vs Conseca, Routing, OTEL defaults & Free tier:**
   - `docs/reference/configuration.md:3137-3152` states *"We do not collect the arguments passed to the tools or any data returned by them"* and *"We do not log the content of your prompts or the responses from the Gemini model."*
   - In code, `telemetry.logPrompts` defaults to **`true`** (`packages/core/src/config/config.ts:1107`): when `security.enableConseca` is enabled, `conseca-logger.ts:33-47,86-105` sends raw `user_prompt`, `trusted_content`, `policy`, full `tool_call` (arguments), and `verdict_rationale` to Clearcut; `ModelRoutingEvent` sends `reasoning` to Clearcut (`clearcut-logger.ts:1466-1471`); enabling OTEL logs prompts, responses, tool args, and hook stdout/stderr by default (while `docs/reference/configuration.md:2560-2561` omits that `logPrompts` defaults to `true`, though `docs/cli/telemetry.md:47` and `docs/cli/enterprise.md:519-521` note it); and Code Assist Free tier defaults `freeTierDataCollectionOptin` to `true`, collecting prompts/code/outputs/edits unless opted out (`packages/cli/src/ui/hooks/usePrivacySettings.ts:158-174`; `packages/cli/src/ui/privacy/CloudFreePrivacyNotice.tsx:131-148`).
3. **Misleading code comment / extension disclosure mismatch (`clearcut-logger.ts:693-697`):**
   - `StartSessionEvent` logging comments `// We deliberately do not log the names of extensions here, to be safe.` while logging unsalted SHA-256 `extension_ids` (trivially reversible against public GitHub extension URLs, `packages/cli/src/config/extension-manager.ts:1569-1595`) AND logging **plaintext `mcp_servers` and `mcp_tools`** right above (`clearcut-logger.ts:654-687`), while OTEL logs plaintext `extensions`, `extension_name`, and `extension_source` (`packages/core/src/telemetry/types.ts:148,1596-1599`).
4. **Undocumented unconditional telemetry & opt-out bypasses:**
   - Neither `docs/reference/configuration.md` nor `docs/resources/tos-privacy.md` discloses that disabling `privacy.usageStatisticsEnabled` does **not** disable Code Assist `recordConversationOffered`/`recordToolCallInteractions` (`POST cloudcode-pa.googleapis.com/v1internal:recordCodeAssistMetrics`) or `vscode-ide-companion` Marketplace pings, nor that `a2a-server` and `sdk` ignore `usageStatisticsEnabled` opt-out (`packages/a2a-server/src/config/config.ts:351-401`; `packages/sdk/src/session.ts:74-95`).
   - `docs/cli/telemetry.md:38-49` and `docs/reference/configuration.md:2544-2565` document OTEL `telemetry.enabled: false` without cross-referencing that Clearcut is ON by default under `privacy.usageStatisticsEnabled`.

---

## 7. Zero-Telemetry / Negative-Check Verification

Static searches across `repos/gemini-cli` (excluding `.git`, `node_modules`, `integration-tests/`, `*.responses`, and `*.test.*` where applicable) verified the following negatives and clean separations:
- **`DO_NOT_TRACK` / `do_not_track`:** `rg -n -i 'DO_NOT_TRACK|do_not_track'` across the entire repository returned exit code `1` (**0 matches** — standard `DO_NOT_TRACK` is ignored).
- **Third-party analytics / crash SDKs:** `rg -n -i '\bposthog\b|\bsentry\b|\bmixpanel\b|\bamplitude\b|segment\.io|\bdatadog\b|\bbugsnag\b|\bstatsig\b|\blaunchdarkly\b|\brollbar\b|\braygun\b|\blogrocket\b|\bnewrelic\b|applicationinsights|scarf\.sh|heap\.io|fullstory'` across `packages/`, `package.json`, `scripts/`, and `sea/` returned **0 matches** (`packages/core/package.json:25-96`, `packages/cli/package.json:32-73`).
- **IP / geolocation / timezone harvesting:** `rg -n -i 'ipify|geoip|ip-api|ipinfo|maxmind|x-forwarded-for|Intl\.DateTimeFormat|timeZone|networkInterfaces'` and `os\.hostname|os\.userInfo|getTimezoneOffset|resolvedOptions\(\)` across `packages/` and `scripts/` found only local UI formatting (`packages/cli/src/ui/hooks/useQuotaAndFallback.ts:380-383`, `packages/cli/src/ui/utils/formatters.ts:136,156-159`), local Footer hostname display (`packages/cli/src/ui/components/Footer.tsx:39`), and local `FileKeychain` encryption salt (`packages/core/src/services/fileKeychain.ts:25`). Zero IP/timezone/hostname telemetry fields exist.
- **Cross-tool credential reads:** `rg` across `packages/` and `scripts/` confirmed zero reads of `~/.codex`, `~/.cursor`, `~/.windsurf`, `~/.aider`, `~/.continue`, `~/.cline`, `~/.aws/credentials`, `~/.netrc`, or `gh/hosts.yml`; the only `.claude` reference in runtime code is the explicit user-invoked `gemini hooks migrate` command reading `${cwd}/.claude/settings*.json` (`packages/cli/src/commands/hooks/migrate.ts:183-227`).
- **Devtools & CI telemetry vars:** `packages/devtools` binds locally and does not beacon; `.github/workflows/eval.yml:31` (`GCLI_LOCAL_FILE_TELEMETRY: 'True'`) and `.github/workflows/gemini-scheduled-issue-triage.yml:181,288-305` (`GEMINI_STRICT_TELEMETRY_LIMITS: 'true'`) are CI-only settings and inject no hidden endpoints into release builds (`esbuild.config.js:89-100`, `scripts/generate-git-commit-info.js:47-77`, `scripts/build_binary.js:178-200`, `sea/sea-launch.cjs:93-150`).
- **Verdict caveat:** Gemini CLI is **not** a zero-telemetry tool. Even setting both `privacy.usageStatisticsEnabled: false` and `telemetry.enabled: false` does not halt Code Assist metrics (`recordCodeAssistMetrics`), `a2a-server` / `sdk` Clearcut logging, or `vscode-ide-companion` Marketplace queries, and its privacy documentation materially contradicts source code regarding plaintext Google email collection.

```json
{
  "slug": "gemini-cli",
  "platform": "Google Gemini CLI",
  "group": "B",
  "repo": "google-gemini/gemini-cli",
  "sha": "08e9a20f749aefcbda38622ac757899c3d6216e0",
  "commit_date": "2026-11-24T23:51:44Z",
  "license": "Apache-2.0",
  "status": "Active",
  "default_state": "Mixed ON by default: Google Clearcut (privacy.usageStatisticsEnabled: true), Chrome CrUX via chrome-devtools-mcp, npm/GitHub/VS Code Marketplace update checks, and Code Assist Free tier data collection opt-in (freeTierDataCollectionOptin ?? true) are ON by default; OpenTelemetry is OFF by default (telemetry.enabled: false) EXCEPT telemetry.logPrompts defaults to true.",
  "opt_out_controls": "Clearcut: set privacy.usageStatisticsEnabled=false in settings.json (hidden from /settings via showInDialog: false; no CLI flag or env var). OTEL: telemetry.enabled=false or GEMINI_TELEMETRY_ENABLED=false, and logPrompts=false / GEMINI_TELEMETRY_LOG_PROMPTS=false. Code Assist Free tier model training: /privacy interactive dialog.",
  "do_not_track": false,
  "opt_out_bypasses": [
    "Setting telemetry.enabled=false or GEMINI_TELEMETRY_ENABLED=false stops OTEL only, leaving Google Clearcut ON",
    "CodeAssistServer.recordConversationOffered and recordToolCallInteractions send POST cloudcode-pa.googleapis.com/v1internal:recordCodeAssistMetrics without checking usageStatisticsEnabled or telemetry.enabled",
    "packages/a2a-server/src/config/config.ts defines privacy.usageStatisticsEnabled in settings.ts but omits it from ConfigParameters, forcing Clearcut and x-gemini-api-privileged-user-id ON",
    "packages/sdk/src/session.ts and types.ts omit usageStatisticsEnabled and telemetry options, forcing Clearcut ON for SDK consumers",
    "packages/vscode-ide-companion unconditionally POSTs to marketplace.visualstudio.com on activation outside managed surfaces",
    "In OTEL when traces=true and logPrompts=false, ModelRoutingEvent.reasoning, gen_ai.system_instructions, gen_ai.tool_definitions, and error messages/stack traces still emit",
    "Short-lived Clearcut sessions with OTEL OFF skip shutdownTelemetry in cleanup.ts, omitting END_SESSION"
  ],
  "persistent_ids": [
    "~/.gemini/installation_id (UUIDv4, fallback '123456789'; sent in Clearcut client_install_id when unauthenticated, OTEL installation.id, and HTTP header x-gemini-api-privileged-user-id)",
    "~/.gemini/google_accounts.json (plaintext active email sent in Clearcut client_email and OTEL user.email, plus historical old[] email list counting GEMINI_CLI_GOOGLE_ACCOUNTS_COUNT)",
    "Per-session session.id / prompt_id / trajectoryId"
  ],
  "hashing": "Unsalted SHA-256 for extension_ids (GitHub repo URL or source/name), hashed_extension_name, and GITHUB_REPOSITORY in Clearcut; local AES-256-GCM scrypt encryption for gemini-credentials.json using hostname+username salt (local only); plaintext email and UUIDv4 installation_id transmitted without hashing",
  "external_auth_reads": [
    "~/.gemini/oauth_creds.json, ~/.gemini/google_accounts.json, ~/.gemini/installation_id, ~/.gemini/gemini-credentials.json, ~/.gemini/mcp-oauth-tokens.json, ~/.gemini/.env, ~/.env",
    "GOOGLE_APPLICATION_CREDENTIALS and ADC (for Google OAuth/Vertex/GCP OTEL)",
    "Explicit user-invoked subcommand only: ${cwd}/.claude/settings*.json via 'gemini hooks migrate --from-claude' (zero automatic external agent auth reads)"
  ],
  "telemetry_endpoints": [
    "https://play.googleapis.com/log?format=json&hasfast=true (Google Clearcut, ON by default)",
    "https://cloudcode-pa.googleapis.com/v1internal:{recordCodeAssistMetrics,listExperiments,fetchAdminControls,getCodeAssistGlobalUserSetting,setCodeAssistGlobalUserSetting} (Code Assist telemetry/experiments/admin/privacy, unconditional for Google auth)",
    "https://www.googleapis.com/oauth2/v2/userinfo (OAuth email lookup cached to google_accounts.json)",
    "http://localhost:4317 or OTEL_EXPORTER_OTLP_ENDPOINT / GEMINI_TELEMETRY_OTLP_ENDPOINT (OTLP gRPC/HTTP, OFF by default)",
    "Google Cloud Trace / Monitoring (custom.googleapis.com/gemini_cli) / Logging (gemini_cli) when telemetry.target=gcp",
    "chrome-devtools-mcp default usage-statistics & performance-crux telemetry (ON by default when Browser Agent used)",
    "npm registry (latest-version), api.github.com/repos/* releases & git ls-remote, and marketplace.visualstudio.com/_apis/public/gallery/extensionquery"
  ],
  "ip_geo_handling": "No client-side IP lookup, geo API, or timezone harvesting in telemetry code, but no IP suppression or strip proxy exists; Google Clearcut, Code Assist, OTLP/GCP, GitHub, npm, and Marketplace endpoints observe client IP at the network layer (global proxy used only if user sets HTTP_PROXY/HTTPS_PROXY/ALL_PROXY).",
  "custom_model_handling": "Model strings (including custom/override models) are transmitted verbatim without redaction or allowlisting in Clearcut events (START_SESSION, API_*, CONTENT_RETRY*, MODEL_ROUTING, Billing), OTEL attributes (gen_ai.request.model, gen_ai.response.model, model), and HTTP User-Agent headers.",
  "docs_discrepancies": [
    "docs/reference/configuration.md:3147-3148 states 'We do not collect any personal information, such as your name, email address, or API keys', directly contradicted by Clearcut client_email (clearcut-logger.ts:420,498) and OTEL user.email (telemetryAttributes.ts:16,23)",
    "docs/reference/configuration.md:3137-3150 claims prompts, responses, and tool arguments are not logged, contradicted when enableConseca=true (sends user_prompt, trusted_content, policy, tool_call args, verdict_rationale to Clearcut via default logPrompts=true), ModelRoutingEvent.reasoning to Clearcut, OTEL default logPrompts=true, and Code Assist Free tier default opt-in",
    "clearcut-logger.ts:693 comments 'We deliberately do not log the names of extensions here' while sending unsalted SHA-256 extension_ids and plaintext mcp_servers/mcp_tools; OTEL sends plaintext extension_name/extensions",
    "Docs omit unconditional Code Assist recordCodeAssistMetrics telemetry, a2a-server/sdk usageStatisticsEnabled bypasses, and default-enabled Browser Agent CrUX telemetry"
  ],
  "privacy_rating": "F",
  "one_line_verdict": "Default-on Google Clearcut sends plaintext user email/installation UUID, hardware specs, MCP lists, and (with Conseca) full prompts/tool calls while docs falsely promise no email or prompt collection, and Code Assist/a2a-server/SDK/Marketplace telemetry bypasses opt-out controls."
}
```
