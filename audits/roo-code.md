# Privacy & Telemetry Forensic Audit: Roo Code (`RooCodeInc/Roo-Code`)

## Header Metadata

- **Slug:** `roo-code`
- **Platform:** `Roo Code`
- **Group:** `B`
- **Repository:** `RooCodeInc/Roo-Code` (`https://github.com/RooCodeInc/Roo-Code`)
- **License:** `Apache-2.0`
- **Status:** `Archived (2026-05-15)`
- **Branch:** `main`
- **Commit SHA:** `b867ec9145750d0ae1ff7f02d35406e9bf2a0b16`
- **Commit Date:** `2026-05-15T14:04:45-04:00`
- **Commit Subject:** `Remove roocode.com web app (#12375)`

> **Archival Note:** Prior to archival on `2026-05-15` (`README.md:66`, `src/core/webview/ClineProvider.ts:152`, `webview-ui/src/i18n/locales/en/chat.json:354-359`), the maintainers removed all telemetry/analytics plumbing (PR `#12324`), Roo Code Cloud/eval infrastructure (PR `#12328`), MDM/org enforcement (PR `#12323`), MCP marketplace (PR `#12326`), and the `roocode.com` web app (PR `#12375`) (`.changeset/v3.54.0.md:1-15`). `packages/telemetry/` and `packages/cloud/` do not exist in the final tree. This audit documents the actual code at commit `b867ec9145750d0ae1ff7f02d35406e9bf2a0b16` while flagging residual docs, CI, and shim artifacts.

---

## 1. Default State & Opt-Out Mechanism

### Default State

1. **Telemetry, Analytics, Crash Reporting & Cloud Relay:** **Absent / Disabled (`0` active telemetry pipelines).**
   - `packages/telemetry/` and `@roo-code/cloud` were removed before archival (`.changeset/v3.54.0.md:5-6`).
   - All workspace `package.json` manifests (`package.json`, `src/package.json`, `webview-ui/package.json`, `apps/*/package.json`, `packages/*/package.json`) contain zero telemetry or crash-reporting dependencies (no PostHog, Sentry, Segment, Mixpanel, Amplitude, Bugsnag, Datadog, or OTEL runtime SDKs). Only `pnpm-lock.yaml:3704,17354,19675` records a transitive `@opentelemetry/api@1.9.0` dependency via the `ai` devDependency (`src/package.json:538`), unimported in runtime source.
   - `globalSettingsSchema` (`packages/types/src/global-settings.ts:80-232`) contains no `telemetrySetting` or telemetry configuration field.
   - Neither `WelcomeViewProvider` (`webview-ui/src/components/welcome/WelcomeViewProvider.tsx:18-144`), `SettingsView` (`webview-ui/src/components/settings/SettingsView.tsx:97-114`), nor `About` (`webview-ui/src/components/settings/About.tsx:16-106`) presents telemetry toggles or notices; `About.tsx:57-75` exposes only a local `debug` checkbox (`roo-cline.debug`, default `false` in `src/package.json:380-384`).

2. **Unsolicited Non-Telemetry Startup Network Activity (Default ON):**
   - `activate()` (`src/extension.ts:291`) unconditionally schedules `initializeModelCacheRefresh()` (`src/api/providers/fetchers/modelCache.ts:208-227`), which runs `2000ms` after extension activation and calls `refreshModels` for two public catalogs (`"openrouter"` and `"vercel-ai-gateway"`) without requiring API keys or prior user interaction.

3. **Local Debug/IPC Logs (Default OFF):**
   - CLI debug logging (`DEBUG_LOG_PATH = path.join(os.homedir(), ".roo", "cli-debug.log")`) initializes with `debugLogEnabled = false` (`packages/core/src/debug-log/index.ts:22-42`) and is enabled only when the CLI is launched with `--debug` (`apps/cli/src/agent/extension-host.ts:185-187`).
   - IPC message file logging (`path.join(os.tmpdir(), "roo-code-messages.log")`) runs only when `ROO_CODE_IPC_SOCKET_PATH` is set (`src/extension.ts:237-238`, `src/extension/api.ts:44,52-61`).
   - `roo-cline.debug` and `roo-cline.debugProxy.enabled` / `roo-cline.debugProxy.tlsInsecure` all default to `false` (`src/package.json:380-402`).

### Opt-Out Controls & Bypasses

- **Telemetry Opt-Out Controls:** None exist because all telemetry code has been removed.
- **`DO_NOT_TRACK`:** Not checked (no telemetry to suppress).
- **Startup Prefetch Gap:** No UI setting, VS Code configuration, CLI flag, or environment variable gates `initializeModelCacheRefresh()` (`src/extension.ts:291`, `src/api/providers/fetchers/modelCache.ts:208-227`). Although these are unauthenticated public model-list GET requests carrying zero user/device identifiers, they cannot be disabled via settings in offline/airgapped setups.

---

## 2. External Auth / Local Non-Workspace File Reads

Roo Code does **not** harvest or read other agents' credentials (`~/.claude`, `~/.codex`, `~/.cursor`, `~/.copilot`, or `~/.ssh`) at startup or for telemetry. Non-workspace home-directory reads fall strictly into user-selected provider authentication and Roo Code's own configuration/skills directories:

1. **Conditional External Provider Credentials (User-Selected Provider Only):**
   - **Qwen Code (`qwen-code` provider):** Reads cached OAuth tokens from `~/.qwen/oauth_creds.json` (or user-supplied `qwenCodeOauthPath`) and writes refreshed OAuth tokens back to that file (`src/api/providers/qwen-code.ts:22-23,37-46,83-94,153-159`, `webview-ui/src/components/settings/providers/QwenCode.tsx:13,42`). Triggered strictly when the `"qwen-code"` provider is invoked via `ensureAuthenticated()` -> `loadCachedQwenCredentials()`.
   - **AWS Bedrock (`bedrock` provider):** Calls `fromIni({ profile: this.options.awsProfile, ignoreCache: true })` (`src/api/providers/bedrock.ts:269-274`) reading standard AWS shared config/credentials files strictly when `"bedrock"` is configured with `awsUseProfile` and `awsProfile`.
   - **Google Vertex AI (`vertex` provider):** Uses user-configured `vertexJsonCredentials`, `vertexKeyFile`, or Google Application Default Credentials (`GoogleAuth`) strictly when `"vertex"` is instantiated (`src/api/providers/anthropic-vertex.ts:43-63`).
   - **OpenAI Codex (`openai-codex` provider):** Uses its own PKCE OAuth flow and stores/reads tokens inside VS Code `SecretStorage` under `"openai-codex-oauth-credentials"` (`src/integrations/openai-codex/oauth.ts:27,423-453`); does **not** read `~/.codex/auth.json`.
   - **Removed Legacy Providers:** `ProviderSettingsManager.ts:168-187` deletes legacy `claudeCodePath` / `claudeCodeMaxOutputTokens` settings; `"roo"` and other retired providers throw `isRetiredProvider` (`src/api/index.ts:114-121`, `packages/types/src/provider-settings.ts:135-145`); `"gemini-cli"` remains only as a type/schema stub (`packages/types/src/provider-settings.ts:111,431`) with no provider handler (`src/api/index.ts:123-182`).

2. **Roo Code & Open-Spec Home Directories (Local Only):**
   - **`~/.roo`:** Global Roo rules/commands/modes (`src/services/roo-config/index.ts:26-29,53-56`) and CLI settings `~/.roo/cli-settings.json` written with `0o600` permissions (`apps/cli/src/lib/storage/config-dir.ts:5`, `apps/cli/src/lib/storage/settings.ts:9,33-35`).
   - **`~/.agents`:** Shared Agent Skills standard directory (`skills`, `skills-{mode}`) loaded locally into system instructions (`src/services/skills/SkillsManager.ts:607-611,658-683`).
   - **Global MCP Settings Path:** Reads platform Roo/Cline MCP settings directories (`~/AppData/Roaming/Roo-Code/MCP`, `~/Documents/Cline/MCP`, `~/.local/share/Roo-Code/MCP`, fallback `~/.roo-code/mcp`) (`src/core/webview/ClineProvider.ts:1544-1565`).
   - **CLI Mock VS Code Storage (`~/.vscode-mock`):** Stores CLI `global-state.json`, `workspace-storage/<hash>/workspace-state.json`, `secrets.json` (`0o600` on non-Windows), and `.machine-id` (`packages/vscode-shim/src/context/ExtensionContext.ts:82-112`, `packages/vscode-shim/src/storage/SecretStorage.ts:79-85`, `apps/cli/src/lib/task-history/index.ts:8`), redirected to a temp directory under `--ephemeral` (`apps/cli/src/lib/storage/ephemeral.ts:5-10`).
   - **User-Configured Auto-Import (`roo-cline.autoImportSettingsPath`):** Defaults to `""` (`src/package.json:344-348`); only reads the specified path (expanding `~/`) if explicitly configured (`src/utils/autoImportSettings.ts:22-30,71-84`).
   - **Safety Guards & `/init` Prompt Text:** `os.homedir()` is used to protect Desktop/home root from accidental workspace checkpointing or file listing (`src/services/checkpoints/ShadowCheckpointService.ts:111-118`, `src/services/glob/list-files.ts:174-178`, `src/core/environment/getEnvironmentDetails.ts:231`, `src/core/task/Task.ts:467`). References to `CLAUDE.md`, `.cursorrules`, `.cursor/rules/`, and `.github/copilot-instructions.md` in `src/services/command/built-in-commands.ts:35,62,166` are LLM prompt instructions for the user-invoked `/init` slash command inside the workspace, not automated reads.

---

## 3. Persistent Device / Install IDs & Hashing

### Persistent Identifiers

1. **CLI Shim `.machine-id` (Local Shim Compatibility Only — Never Transmitted):**
   - In the CLI runtime's VS Code API shim (`packages/vscode-shim/src/utils/machine-id.ts:16-44`), `machineIdSync()` reads `~/.vscode-mock/.machine-id`; if missing, it computes a SHA-256 hash of `${os.hostname()}-${crypto.randomBytes(16).toString("hex")}` (`lines 32-38`) and persists it to `~/.vscode-mock/.machine-id` to populate `vscode.env.machineId` (`packages/vscode-shim/src/api/create-vscode-api-mock.ts:106`, `packages/vscode-shim/src/interfaces/workspace.ts:88`).
   - Negative `rg` verification across the entire repository (`CHECK 5` in Section 7) confirms **zero non-test references to `machineId` outside `packages/vscode-shim/`**: neither the VS Code extension (`src/`) nor CLI (`apps/cli/`) reads `vscode.env.machineId` or transmits any machine/install/device UUID.

2. **No Persistent Telemetry or Cross-Session User ID:**
   - Ephemeral per-instance or per-task IDs (`uuidv7()` in `src/api/providers/openai-native.ts:88,101-103`, `src/core/task/Task.ts:454`; `crypto.randomUUID().slice(0, 8)` `instanceId` in `src/core/task/Task.ts:469`) are generated transiently in memory. When the user chooses OpenAI-Native, OpenAI Codex (`session_id`, plus JWT `chatgpt_account_id` extracted from the user's OAuth token), Requesty (`requesty.trace_id`), or Unbound (`unbound_metadata.taskId`), these task/session IDs are sent solely to that user-selected LLM API (`src/api/providers/openai-codex.ts:355-363,504-512`, `src/api/providers/requesty.ts:151`, `src/api/providers/unbound.ts:145`).

### Hashing Inventory

All hashes in the codebase are strictly local or standard OAuth PKCE; **none** hash machine IDs, emails, or account IDs for telemetry:

| Hash / Token | Input | Purpose & Scope | Source |
| :--- | :--- | :--- | :--- |
| SHA-256 | `${os.hostname()}-${random16Bytes}` | Local CLI shim `~/.vscode-mock/.machine-id`; never read/sent | `packages/vscode-shim/src/utils/machine-id.ts:32-43` |
| SHA-256 | `workspacePath` / esbuild tool config | Local disk cache folder/filenames | `src/services/code-index/cache-manager.ts:26`, `packages/core/src/custom-tools/custom-tool-registry.ts:291` |
| SHA-256 | PKCE `code_verifier` + random `state` | OAuth 2.0 PKCE S256 challenge (`OpenAI Codex`) | `src/integrations/openai-codex/oauth.ts:188,196` |
| MD5 | Tool call ID >64 chars / todo content+status | OpenAI tool-ID truncation / local todo item ID | `src/utils/tool-id.ts:37`, `src/core/tools/UpdateTodoListTool.ts:191-194` |
| 32-bit int hash | `workspacePath` | Local `.vscode-mock/workspace-storage/<hash>` dir name | `packages/vscode-shim/src/utils/paths.ts:24-32` |
| `randomBytes(6)` | CSPRNG | Local Unix/Windows named-pipe IPC client ID | `packages/ipc/src/ipc-server.ts:46` |

---

## 4. Exact Outbound Telemetry / Analytics / Crash / OTEL / Relay / Update Endpoints

### Telemetry / Analytics / Crash / OTEL / Cloud Relay Endpoints
- **None in active runtime code.** Zero PostHog, Sentry, Segment, OTEL, or Roo Cloud relay URLs exist anywhere in `src/`, `webview-ui/`, `apps/cli/`, or `packages/`.
- **Dormant / Residual CI & Code Artifacts (Dead at Runtime):**
  - `.github/workflows/marketplace-publish.yml:32-33,50` still echoes `POSTHOG_API_KEY=${{ secrets.POSTHOG_API_KEY }}` into `.env`, and `src/esbuild.mjs:73` / `apps/vscode-nightly/esbuild.mjs:69` / `src/.vscodeignore:35-36` still bundle an optional `.env` loaded by `dotenvx` (`src/extension.ts:10-18`). However, zero runtime code reads `process.env.POSTHOG_API_KEY`.
  - `.env.sample:1-6` still lists deprecated keys (`POSTHOG_API_KEY`, `CLERK_BASE_URL`, `ROO_CODE_API_URL`, `ROO_CODE_PROVIDER_URL`).
  - `src/core/webview/ClineProvider.ts:1128,1219` still allows retired image domains (`https://storage.googleapis.com https://img.clerk.com`) in webview CSP `img-src`.
  - `apps/cli/src/lib/sdk/client.ts:19-29` retains an unreferenced `createClient` targeting `${url}/trpc`.

### Automatic Startup Non-Telemetry Endpoints (Default ON)
1. `GET https://openrouter.ai/api/v1/models` (`src/api/providers/fetchers/openrouter.ts:99-102`)
2. `GET https://ai-gateway.vercel.sh/v1/models` (`src/api/providers/fetchers/vercel-ai-gateway.ts:58-61`)
   - Invoked automatically `2000ms` after extension activation via `initializeModelCacheRefresh()` (`src/extension.ts:291`, `src/api/providers/fetchers/modelCache.ts:208-227`) to refresh public model pricing/capability caches. No auth headers or client identifiers are sent.

### Manual CLI Upgrade / Install Endpoints (User-Invoked Only — No Auto-Update Poller)
- `https://api.github.com/repos/RooCodeInc/Roo-Code/releases?per_page=100` (`apps/cli/src/commands/cli/upgrade.ts:6,68-74`) — queried only when the user runs `roo upgrade`.
- `https://raw.githubusercontent.com/RooCodeInc/Roo-Code/main/apps/cli/install.sh` (`apps/cli/src/commands/cli/upgrade.ts:7-8,115`) and `https://github.com/RooCodeInc/Roo-Code/releases/download/cli-v${VERSION}/${TARBALL}` (`apps/cli/install.sh:103,185`).

### User-Initiated OAuth / Key-Info / Configured Provider Endpoints
- **Webview Key/Provider Lookups (CSP `connect-src`, `src/core/webview/ClineProvider.ts:1219`):** Queries `${openRouterDomain}` or `https://api.requesty.ai` only when inspecting OpenRouter/Requesty models or key balances in UI (`webview-ui/src/components/ui/hooks/`).
- **OAuth Redirects / Token Exchanges (User Click Only):**
  - OpenRouter: `https://openrouter.ai/auth` and `POST .../api/v1/auth/keys` (`webview-ui/src/oauth/urls.ts:8`, `src/activate/handleUri.ts:14-29`, `src/core/webview/ClineProvider.ts:1575-1620`).
  - Requesty: `https://app.requesty.ai/oauth/authorize` (`webview-ui/src/oauth/urls.ts:12`, `src/core/webview/ClineProvider.ts:1575-1620`).
  - OpenAI Codex: `https://auth.openai.com/oauth/authorize`, `https://auth.openai.com/oauth/token`, `https://chatgpt.com/backend-api/codex`, `https://chatgpt.com/backend-api/wham/usage` (`src/integrations/openai-codex/oauth.ts:18-21`, `src/api/providers/openai-codex.ts:34`, `src/integrations/openai-codex/rate-limits.ts:3`).
  - Qwen Code: `https://chat.qwen.ai/api/v1/oauth2/token` and `https://dashscope.aliyuncs.com/compatible-mode/v1` (or token `resource_url`) (`src/api/providers/qwen-code.ts:19-20,71,188`).
- **Configured LLM / Embedding Providers & MCP Servers:** Standard inference/embedding calls go strictly to the user-selected provider (`anthropic`, `openai`, `bedrock`, `vertex`, `gemini`, `deepseek`, `mistral`, `moonshot`, `minimax`, `xai`, `baseten`, `sambanova`, `zai`, `fireworks`, `unbound`, `litellm`, `poe`, `ollama`, `lmstudio`, `qdrant`, etc.).
- **Referral Query Params on Manual UI Signup Link:** `webview-ui/src/components/settings/providers/SambaNova.tsx:45` includes `?utm_source=roocode&utm_medium=external&utm_campaign=cloud_signup` on an external signup button clicked manually by the user.

---

## 5. Exact Data & Properties Collected

Because all telemetry and crash-upload SDKs were removed, **zero telemetry payloads, behavioral profiles, or automatic crash reports** leave the user's machine.

1. **IP / Geolocation Handling:**
   - Zero IP-lookup or GeoIP API calls (`CHECK 6` in Section 7: `0` matches).
   - No Roo telemetry ingest or relay proxy exists. Client IP address is exposed only via direct HTTPS requests to the user's selected LLM/OAuth/MCP endpoints, manual `roo upgrade` GitHub API requests, and the two unauthenticated startup model-catalog GET requests (`openrouter.ai`, `ai-gateway.vercel.sh`).

2. **Custom / Private Model Names:**
   - Custom/private model IDs (e.g., LiteLLM, Ollama, LM Studio, OpenAI-compatible, or custom Bedrock/Vertex IDs) are transmitted solely to the user's configured inference/embedding endpoint. Zero telemetry collector exists to capture or leak model names.

3. **Prompts, Code, Paths & Stack Traces:**
   - **LLM Inference Prompts (Selected Provider Only):** Environment blocks sent to the user's active LLM provider (`src/core/prompts/sections/system-info.ts:10-25`, `src/core/environment/getEnvironmentDetails.ts:23-265`) include OS name/platform/release, default shell, `os.homedir()`, `cwd`, open/visible workspace tabs/files, terminal output, and local timezone (`Intl.DateTimeFormat().resolvedOptions().timeZone` + UTC offset when `includeCurrentTime` is enabled). For OpenAI-Native Responses API, `store: false` explicitly opts out of 30-day server retention (`src/api/providers/openai-native.ts:345`).
   - **Provider Headers (Selected Provider Only):** Providers receive standard identification headers such as `HTTP-Referer`/`X-Title` (`src/api/providers/constants.ts:3-7`), `User-Agent: RooCode/${Package.version}` (`src/api/providers/bedrock.ts:253`), or `User-Agent` including `Package.version`, `os.platform()`, `os.release()`/`os.arch()`, and Node version (`src/api/providers/openai-native.ts:96-104`, `src/api/providers/openai-codex.ts:358-363,501-512`, `src/api/providers/qwen-code.ts:73-75`), plus `originator: "roo-code"` on Codex (`openai-codex.ts:508`).
   - **Error Stack Traces / Diagnostics:** React error boundaries render stack traces locally in the UI (`webview-ui/src/components/ErrorBoundary.tsx:58-83`), and diagnostics export writes a local JSON file under `os.tmpdir()/roo-diagnostics-*.json` (`src/core/webview/diagnosticsHandler.ts:35-91`) strictly on user action without uploading anywhere.

4. **Behavioral Profile / Usage Metrics:**
   - Token counts and cost summaries (`TokenUsage`) are tracked locally in extension/CLI state for UI display (`apps/cli/src/ui/components/MetricsDisplay.tsx:47-65`) and local task history (`~/.vscode-mock` or VS Code globalStorage).

---

## 6. Docs / Privacy Policy vs Code Discrepancies

Although the codebase is clean of telemetry at commit `b867ec9145750d0ae1ff7f02d35406e9bf2a0b16`, the documentation and CI/template files were not cleaned up alongside PRs `#12324` and `#12328` and therefore **misstate** the runtime behavior in the over-reporting direction (describing telemetry that no longer exists):

1. **Outdated `PRIVACY.md` (`PRIVACY.md:3,13,17,24`):**
   - `PRIVACY.md` (dated September 11, 2025) still claims Roo Code collects default-on anonymous usage telemetry via PostHog (`VS Code machine ID`, feature usage, error/exception telemetry, hardware specs) with an opt-out toggle in extension settings. In reality, PostHog, `packages/telemetry/`, `vscode.env.machineId` usage, and the settings toggle were all deleted in PR `#12324`.
2. **Outdated Docusaurus Docs (`apps/docs/`):**
   - `apps/docs/docs/advanced-usage/available-tools/attempt-completion.md:100` and `apps/docs/docs/advanced-usage/available-tools/new-task.md:89` still document telemetry capture calls on task completion and creation.
   - `apps/docs/docusaurus.config.ts:437-438` and `apps/docs/src/constants.ts:21` still link to the outdated `PRIVACY.md` policy.
3. **Stale CI / Packaging / Repo Metadata:**
   - `.github/workflows/marketplace-publish.yml:32-33,50` still echoes `POSTHOG_API_KEY=${{ secrets.POSTHOG_API_KEY }}` into `.env` before packaging, and `src/.vscodeignore:35` retains `# Include .env file for telemetry`.
   - `.env.sample:1-6` still documents removed variables (`POSTHOG_API_KEY`, `CLERK_BASE_URL`, `ROO_CODE_API_URL`, `ROO_CODE_PROVIDER_URL`).
   - `.roo/skills/roo-translation/SKILL.md:87` and `.roo/rules-translate/001-general-rules.md:43` still use `i18nKey="welcome:telemetry.changeSettings"` as an example.
4. **Stale Code Comments & Dead Symbols:**
   - `src/api/index.ts:46,57-63` JSDoc mentions retired `X-Roo-Task-ID` and `@default true` for OpenAI `store`, whereas `src/api/providers/openai-native.ts:345` enforces `store: false`.
   - Unused legacy types/stubs/mocks remain unreferenced in code: `packages/types/src/cookie-consent.ts`, `packages/types/src/organization.ts`, `packages/types/src/provider-settings.ts:111,431` (`gemini-cli`), `apps/cli/src/lib/sdk/client.ts:19-29`, and `mockCaptureException` in `src/api/providers/__tests__/mistral.spec.ts:1,64` and `src/api/providers/__tests__/bedrock-error-handling.spec.ts:1,32`.

---

## 7. Zero-Telemetry / Negative-Check Verification

All checks below were executed read-only with `rg --hidden` at repo root `repos/roo-code` (SHA `b867ec9145750d0ae1ff7f02d35406e9bf2a0b16`):

1. **Telemetry SDKs & `telemetry` keyword** (excluding `.git/**`, `apps/docs/**`, `CHANGELOG.md`, `**/CHANGELOG.md`, `pnpm-lock.yaml`, `.changeset/**`, `PRIVACY.md`, `.dockerignore`):
   ```powershell
   rg -n --hidden -w -i "(posthog|sentry|opentelemetry|otel|mixpanel|amplitude|bugsnag|datadog|rollbar|raygun|appinsights|applicationinsights|plausible|umami|matomo|statsig|launchdarkly|growthbook|flagsmith|telemetry)" -g "!.git/**" -g "!apps/docs/**" -g "!CHANGELOG.md" -g "!**/CHANGELOG.md" -g "!pnpm-lock.yaml" -g "!.changeset/**" -g "!PRIVACY.md" -g "!.dockerignore" .
   ```
   - **Result:** `3` matches, **zero** in runtime source or build deps: stale translation doc comments (`.roo/skills/roo-translation/SKILL.md:87`, `.roo/rules-translate/001-general-rules.md:43`) and a comment (`src/.vscodeignore:35`: `# Include .env file for telemetry`). (Note `.env.sample` is ignored by `.gitignore`; manual read of `.env.sample:1-6` verified only `POSTHOG_API_KEY` there with zero consumers.)

2. **Segment SDKs:**
   ```powershell
   rg -n --hidden -i "(@segment/|analytics-node|analytics\.js|cdn\.segment\.com)" -g "!.git/**" -g "!pnpm-lock.yaml" .
   ```
   - **Result:** `0` matches.

3. **`analytics` keyword** (same non-doc exclusions as Check 1):
   ```powershell
   rg -n --hidden -w -i "analytics" -g "!.git/**" -g "!apps/docs/**" -g "!CHANGELOG.md" -g "!**/CHANGELOG.md" -g "!pnpm-lock.yaml" -g "!.changeset/**" .
   ```
   - **Result:** `8` non-telemetry matches only: local `// Task Analytics` section comments (`src/extension/api.ts:408`, `packages/types/src/task.ts:158`, `packages/types/src/events.ts:41,244`), a JSDoc comment on diff strategy naming (`src/shared/tools.ts:364`), and 3 MCP tool-ID test strings (`src/utils/__tests__/tool-id.spec.ts:71,130,170`).

4. **Capture symbols, telemetry env vars & Cloud package references:**
   ```powershell
   rg -n --hidden -i "(ROO_CODE_DISABLE_TELEMETRY|TELEMETRY_API_KEY|POSTHOG_API_KEY|telemetrySetting|captureException|captureEvent|TelemetryService|PostHogClient|createTelemetryLogger|sendBeacon|@roo-code/telemetry|@roo-code/cloud)" -g "!.git/**" -g "!apps/docs/**" -g "!CHANGELOG.md" -g "!**/CHANGELOG.md" -g "!pnpm-lock.yaml" -g "!.changeset/**" .
   ```
   - **Result:** `5` residual non-runtime matches only: `.github/workflows/marketplace-publish.yml:33` (`POSTHOG_API_KEY` env injection into `.env`, unread by code) and `mockCaptureException` mocks in `src/api/providers/__tests__/mistral.spec.ts:1,64` and `src/api/providers/__tests__/bedrock-error-handling.spec.ts:1,32`.

5. **Persistent `machineId` references (non-test code):**
   ```powershell
   rg -n --hidden -w "machineId" -g "!.git/**" -g "!PRIVACY.md" -g "!**/__tests__/**" -g "!**/*.test.*" -g "!**/*.spec.*" .
   ```
   - **Result:** `5` matches confined exclusively to `packages/vscode-shim/` (`machine-id.ts:32,38,43`, `interfaces/workspace.ts:88`, `api/create-vscode-api-mock.ts:106`). Zero reads in `src/` or `apps/cli/`.

6. **IP / GeoIP lookup services:**
   ```powershell
   rg -n --hidden -i "(ipify|ipinfo|ip-api|ipapi|geoip|maxmind|cloudflare\.com/cdn-cgi/trace|ifconfig\.me|icanhazip|checkip)" -g "!.git/**" -g "!pnpm-lock.yaml" .
   ```
   - **Result:** `0` matches.

7. **Other agent credential paths:**
   ```powershell
   rg -n --hidden -i "(\.codex|\.claude/|\.gemini/|\.copilot|\.cursor/|credentials\.json|auth\.json|oauth_creds\.json)" -g "!.git/**" -g "!apps/docs/**" -g "!CHANGELOG.md" -g "!**/CHANGELOG.md" -g "!pnpm-lock.yaml" .
   ```
   - **Result:** `6` matches only: `oauth_creds.json` for user-selected `qwen-code` auth (`src/api/providers/qwen-code.ts:23`, `webview-ui/src/components/settings/providers/QwenCode.tsx:13,42`) and `/init` workspace instruction prompts (`src/services/command/built-in-commands.ts:35,62,166`).

---

```json
{
  "slug": "roo-code",
  "platform": "Roo Code",
  "group": "B",
  "repo": "RooCodeInc/Roo-Code",
  "sha": "b867ec9145750d0ae1ff7f02d35406e9bf2a0b16",
  "commit_date": "2026-05-15T14:04:45-04:00",
  "license": "Apache-2.0",
  "status": "Archived (2026-05-15)",
  "default_state": "Disabled / Zero-Telemetry (all telemetry, analytics, and Roo Code Cloud plumbing removed in PRs #12324 and #12328 prior to archival; only an unauthenticated 2-endpoint public model-catalog prefetch to openrouter.ai and ai-gateway.vercel.sh runs 2s after extension activation)",
  "opt_out_controls": [
    "No telemetry toggles or env vars exist because all telemetry/analytics code was removed (PR #12324)",
    "No UI/CLI/env opt-out exists for the 2s startup model-catalog prefetch (src/extension.ts:291, src/api/providers/fetchers/modelCache.ts:208-227)",
    "roo-cline.debug, roo-cline.debugProxy.enabled, and roo-cline.autoImportSettingsPath all default disabled/empty (src/package.json:344-348,380-402)"
  ],
  "do_not_track": false,
  "opt_out_bypasses": [
    "No telemetry opt-out bypasses (zero telemetry); however, initializeModelCacheRefresh() unconditionally issues unauthenticated GET requests to openrouter.ai and ai-gateway.vercel.sh 2000ms after extension activation with no opt-out switch (src/extension.ts:291, src/api/providers/fetchers/modelCache.ts:208-227)"
  ],
  "persistent_ids": [
    "None in src/ or telemetry; packages/vscode-shim/src/utils/machine-id.ts:16-44 creates ~/.vscode-mock/.machine-id (SHA-256 of hostname + 16 random bytes) solely to populate the CLI VS Code mock env.machineId property, with zero readers in src/ or apps/cli/"
  ],
  "hashing": "No telemetry ID hashing; SHA-256 is used locally for packages/vscode-shim .machine-id (untransmitted), local disk/esbuild caches (cache-manager.ts:26, custom-tool-registry.ts:291), and OpenAI Codex OAuth PKCE S256 (oauth.ts:188,196); MD5 is used locally for tool-ID truncation (tool-id.ts:37) and todo IDs (UpdateTodoListTool.ts:191-194); ephemeral uuidv7()/randomUUID() task/session IDs go only to user-selected LLM providers.",
  "external_auth_reads": "None at startup or for telemetry (0 reads of ~/.claude, ~/.codex, ~/.cursor, ~/.copilot, or ~/.ssh). Conditional user-selected provider auth reads only: ~/.qwen/oauth_creds.json when qwen-code provider is used (qwen-code.ts:22-23,83-94), AWS profile via fromIni when bedrock awsUseProfile is enabled (bedrock.ts:269-274), and GCP ADC/keyfile when vertex is used (anthropic-vertex.ts:43-63). OpenAI Codex OAuth uses VS Code SecretStorage, not ~/.codex.",
  "telemetry_endpoints": [
    "None for telemetry, analytics, crash reporting, OTEL, or Roo Cloud relay (all removed prior to archival)",
    "Non-telemetry startup model-catalog GET prefetch (2s after activation, unauthenticated, no opt-out): https://openrouter.ai/api/v1/models and https://ai-gateway.vercel.sh/v1/models",
    "Dormant CI artifact: .github/workflows/marketplace-publish.yml:32-33 still writes POSTHOG_API_KEY into .env, but zero runtime code reads it"
  ],
  "ip_geo_handling": "Zero IP/GeoIP lookup endpoints or telemetry ingest proxies; client IP is exposed only on direct requests to user-configured LLM/OAuth/MCP providers, manual `roo upgrade` GitHub calls, and the two unauthenticated startup model-catalog GETs (openrouter.ai, ai-gateway.vercel.sh).",
  "custom_model_handling": "Custom/private model IDs are sent solely to the user's configured inference/embedding endpoint; no telemetry backend exists to record or leak model names.",
  "docs_discrepancies": [
    "PRIVACY.md:3,13,17,24 still describes default-on PostHog telemetry, VS Code machineId collection, and a UI opt-out toggle that were removed in PR #12324",
    "apps/docs/docs/advanced-usage/available-tools/attempt-completion.md:100 and new-task.md:89 still document removed telemetry capture calls, and docusaurus.config.ts:437-438 links to outdated PRIVACY.md",
    ".github/workflows/marketplace-publish.yml:32-33,50, .env.sample:1-6, src/.vscodeignore:35, and .roo translation docs retain unused POSTHOG_API_KEY / Cloud / telemetry references",
    "src/api/index.ts:46,57-63 JSDoc mentions retired X-Roo-Task-ID and OpenAI store @default true while openai-native.ts:345 sets store: false",
    "Dead unreferenced artifacts remain from Cloud/telemetry removal: ClineProvider.ts:1128,1219 CSP (img.clerk.com, storage.googleapis.com), apps/cli/src/lib/sdk/client.ts, packages/types/src/{cookie-consent.ts,organization.ts}, gemini-cli provider-settings stub, and mockCaptureException in two test files"
  ],
  "privacy_rating": "A",
  "one_line_verdict": "Archived 2026-05-15 at a clean Zero-Telemetry state after PRs #12324/#12328/#12375 stripped PostHog, packages/telemetry, and Roo Code Cloud; only unauthenticated 2s startup public model-catalog GETs (openrouter.ai, ai-gateway.vercel.sh), conditional provider-selected auth (~/.qwen/oauth_creds.json, AWS/GCP), and stale pre-removal PRIVACY.md/CI artifacts remain."
}
```
