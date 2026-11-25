# Forensic Privacy & Telemetry Audit: Pi (`earendil-works/pi`)

- **Slug:** `pi`
- **Platform:** Pi
- **Group:** B
- **Repository:** `earendil-works/pi` (`https://github.com/earendil-works/pi`)
- **License:** `MIT`
- **Status:** Active (`badlogic/pi-mono` redirects here)
- **Branch:** `main`
- **SHA:** `c11a73bfba1310dc4075ad836254ea87003f409b`
- **Commit Date:** `2026-11-25T00:53:26+01:00`
- **Commit Subject:** `docs(ai): add changelog entry for overflow identifier match fix (#12504)`

---

## 1. Default State & Opt-Out Mechanism

### Default Telemetry State
Pi ships **zero third-party analytics, crash-reporting, or OpenTelemetry exporter SDKs** and performs **no event/prompt/usage telemetry**, yet operates several **default-ON outbound channels** centered on `pi.dev`, provider attribution headers, update checks, and packaged macOS Sparkle updates:

1. **Install/Upgrade Beacon & Provider Attribution (`enableInstallTelemetry: true` — ON by default):**
   - `packages/coding-agent/src/core/settings-defaults.ts:32` sets `enableInstallTelemetry: true`.
   - On interactive startup when `session.state.messages.length === 0` (i.e., not resuming/continuing), `getChangelogForDisplay()` calls `reportInstallTelemetry(VERSION)` on first install (`!lastVersion`) or after an upgrade (`newEntries.length > 0`), firing `GET https://pi.dev/api/report-install?version=<VERSION>` with a detailed `User-Agent` (`packages/coding-agent/src/modes/interactive/interactive-mode.ts:1421-1450,1520-1537`). Print (`-p`), JSON, and RPC entrypoints do not invoke `InteractiveMode.getChangelogForDisplay()`.
   - While enabled, `getClientIdentity(settingsManager)` (`packages/coding-agent/src/core/client-identity.ts:10-16`, wired into `packages/coding-agent/src/core/sdk.ts:352`) also attaches `PI_ATTRIBUTION` (`{ name: "pi", url: "https://pi.dev", categories: ["cli-agent"] }`) to LLM API requests targeting OpenRouter, NVIDIA NIM, or OpenCode (`packages/ai/src/utils/identity.ts:24,45-46,92-124`).
   - **Standalone Library Defaults:** Downstream SDK users of `@earendil-works/pi-agent-core` (`packages/agent/src/agent-loop.ts:41,415`) and `@earendil-works/pi-durable` (`packages/durable/src/harness/harness.ts:194,200-204`) receive `DEFAULT_AGENT_IDENTITY` / `DEFAULT_HARNESS_IDENTITY` (`{ attribution: PI_ATTRIBUTION }`) without reading `PI_TELEMETRY` or `settings.json`, unless overridden with `identity: { attribution: null }`.

2. **Startup Version Check (`https://pi.dev/api/latest-version` — ON by default in interactive mode):**
   - `InteractiveMode.run()` unconditionally schedules `checkForNewPiVersion(this.version)` (`packages/coding-agent/src/modes/interactive/interactive-mode.ts:1256`), sending `GET https://pi.dev/api/latest-version` with detailed `User-Agent` (`packages/coding-agent/src/utils/version-check.ts:5,51-69,97-109`).

3. **Auth-Gated Remote Model Catalog Refresh (`https://pi.dev/api/models/providers/<id>` — ON by default in interactive & RPC modes):**
   - `ModelRuntime` wraps every built-in provider except `radius` with `withRemoteCatalog` (4-hour cache + `ETag`/`If-None-Match`) or, for `openrouter`, `withPersonalizedCatalog` (skips the 4-hour interval and ETag revalidation) (`packages/coding-agent/src/core/model-runtime.ts:229-241`; `packages/coding-agent/src/core/remote-catalog-provider.ts:14,16,111-157,200,204-248`).
   - Both `InteractiveMode.run()` (`packages/coding-agent/src/modes/interactive/interactive-mode.ts:1246-1253`) and RPC startup (`packages/coding-agent/src/main.ts:1020-1027`) trigger `refresh()`. Because `ModelsImpl.refresh()` invokes `resolveNetworkRefresh()` (`packages/ai/src/models.ts:573-653,692-716`) and `withRemoteCatalog` sets `refreshModelsAuth: "configured"` (`remote-catalog-provider.ts:127`), **requests to `pi.dev/api/models/providers/<id>` are sent selectively only for built-in providers for which local credentials/env keys exist**, disclosing the user's configured AI provider set to `pi.dev`.

4. **Startup Package Update Checks & Automatic `fd`/`rg` Binary Downloads (ON by default in interactive mode):**
   - `InteractiveMode.run()` calls `checkForPackageUpdates()` (`interactive-mode.ts:1263,1352-1368`), querying `npm` / `git` for unpinned installed packages (`packages/coding-agent/src/core/package-manager.ts:1396-1459`).
   - Missing `fd` or `rg` tools trigger `ensureTool()` downloads from `github.com/sharkdp/fd` and `github.com/BurntSushi/ripgrep` (`packages/coding-agent/src/utils/tools-manager.ts:42-58,99-127,247-272,337-375`).

5. **Experimental Analytics Opt-In (`enableAnalytics: false` — OFF / dormant):**
   - `enableAnalytics` defaults to `false` (`packages/coding-agent/src/core/settings-defaults.ts:33`). First-time onboarding prompts for opt-in only when `PI_EXPERIMENTAL=1` (`packages/coding-agent/src/core/experimental.ts:1-3`; `packages/coding-agent/src/main.ts:717-722`). `trackingId` generated on opt-in is unused by any outbound network code.

6. **Local Diagnostic Telemetry (`@earendil-works/pi-telemetry` & `apps/p2` — OFF by default, local-only):**
   - `@earendil-works/pi-telemetry` defines vendor-neutral span/metric interfaces only (`packages/telemetry/README.md:15,67`). `apps/p2` defaults `telemetry` to off (`apps/p2/src/runtime/settings.ts:75-79`); when enabled in `<data>/settings.json`, it writes exclusively to local `<data>/telemetry.sqlite` (`apps/p2/src/hosts/server.ts:194-219`; `apps/p2/src/runtime/host-dirs.ts:65-87`; `apps/p2/plugins/telemetry/server.ts:31-36`).

7. **Packaged `apps/p2-mac` Sparkle Auto-Updater (ON by default in packaged macOS builds):**
   - Packaged macOS app builds inject `SUFeedURL` (`https://pkg.earendil.com/-/apps/pi-mac/nightly/appcast.xml`) and start Sparkle automatically (`apps/p2-mac/scripts/package-app.sh:22,88`; `apps/p2-mac/Sources/P2Mac/Updater.swift:1-15`).

### Opt-Out Controls & Bypasses
- **UI Toggle:** `/settings` -> `"Install telemetry"` (`enableInstallTelemetry`: `"true"` / `"false"`, `packages/coding-agent/src/modes/interactive/components/settings-selector.ts:575-580`).
- **Config (`~/.pi/agent/settings.json`):** `{ "enableInstallTelemetry": false }`.
- **Environment & CLI Flags:**
  - `PI_TELEMETRY=0` / `false` / `no` (overrides `enableInstallTelemetry`, checked via `isTruthyEnvFlag` in `packages/coding-agent/src/core/telemetry.ts:3-13`).
  - `PI_SKIP_VERSION_CHECK=1` (stops `latest-version` only, `version-check.ts:98`).
  - `--offline` or `PI_OFFLINE=1` (`true`/`yes`) (sets `PI_OFFLINE="1"` and `PI_SKIP_VERSION_CHECK="1"` in `packages/coding-agent/src/main.ts:610-614`, stopping `report-install`, `latest-version`, `models/providers/<id>`, package checks, and `fd`/`rg` auto-downloads).
- **`DO_NOT_TRACK` / `NO_TELEMETRY`:** **Not supported** (`0` matches across the repository).
- **Opt-Out Bypasses & Asymmetries:**
  1. `PI_TELEMETRY=0` / `enableInstallTelemetry: false` stops only `report-install` and `coding-agent` `PI_ATTRIBUTION`; it does **not** stop `pi.dev/api/latest-version`, credential-gated `pi.dev/api/models/providers/<id>`, package update checks, `fd`/`rg` downloads, or `apps/p2-mac` Sparkle polling. There is no granular UI setting to keep Pi online while disabling `pi.dev` catalog or version checks (only all-or-nothing `--offline` / `PI_OFFLINE`).
  2. Detailed `User-Agent` (`pi/<version> (<platform>; <runtime>; <arch>)`) remains active on all `pi.dev`, `github.com`, and LLM requests even when `PI_TELEMETRY=0` and `enableInstallTelemetry: false` (`client-identity.ts:12`, `pi-user-agent.ts:4-5`).
  3. Standalone `@earendil-works/pi-agent-core` (`agent-loop.ts:41,415`) and `@earendil-works/pi-durable` (`harness.ts:194,200-204`) ignore `PI_TELEMETRY` and `settings.json`, sending `PI_ATTRIBUTION` unless overridden in code.
  4. `PI_OFFLINE` truthiness mismatch: `main.ts:610`, `package-manager.ts:84-88`, and `tools-manager.ts:15-19` accept only `"1"/"true"/"yes"`, whereas `interactive-mode.ts:1246,1353,1521`, `version-check.ts:55,98`, and `model-runtime.ts:248` check raw JS truthiness or `!== undefined`, causing `PI_OFFLINE=0` / `PI_OFFLINE=false` to behave inconsistently across checks.

---

## 2. External Auth / Local Non-Workspace File Reads

Pi does **not** harvest or read other AI agents' credential stores (`~/.claude`, `~/.codex`, `~/.cursor`, `~/.opencode`, `~/.gemini`, `~/.windsurf`, or `~/.aider`). All non-workspace reads fall into three narrow, documented categories:

1. **Pi's Own State & Legacy Migrations:**
   - `~/.pi/agent/` (`settings.json`, `auth.json`, `models.json`, `models-store.json`, `crashes.json`, `sessions/`), `~/.pi/p2` / `~/.pi/p2-tui`, and legacy migration of `~/.pi/agent/oauth.json` and `settings.json` `apiKeys` into `auth.json` (`packages/coding-agent/src/migrations.ts:21-73`).
   - Project & ancestor `.pi/` directories and `AGENTS.override.md` / `AGENTS.md` / `AGENTS.MD` / `CLAUDE.md` / `CLAUDE.MD` in `<agentDir>` and `cwd`->root ancestors (unless `--no-context-files`, `packages/coding-agent/src/core/resource-loader.ts:214-233,262-304`).
   - Shared agent-skills convention `~/.agents/skills` and ancestor `.agents/skills` (`packages/coding-agent/src/core/package-manager.ts:545,2700,2740,2791`).

2. **Cloud / Tool Credentials Read Strictly When Relevant Feature/Provider Runs:**
   - **Google Vertex ADC:** `hasVertexAdcCredentials()` checks `GOOGLE_APPLICATION_CREDENTIALS` or `~/.config/gcloud/application_default_credentials.json` (`existsSync` check / Vertex SDK ambient auth) only when resolving `google-vertex` availability/credentials (`packages/ai/src/env-api-keys.ts:40-71`).
   - **Hugging Face Token (for `/llama` extension):** `findHuggingFaceToken()` reads `$HF_TOKEN`, `$HF_TOKEN_PATH`, `$HF_HOME/token`, `$XDG_CACHE_HOME/huggingface/token`, or `~/.cache/huggingface/token` to authenticate requests to `huggingface.co` (`packages/coding-agent/src/extensions/llama/huggingface.ts:5,46-61`).
   - **GitHub CLI (`gh`) (for `/share` Gist fallback):** `shareSession()` spawns `gh auth status` and `gh gist create --public=false` only when the user runs `/share` without Radius auth (`packages/coding-agent/src/modes/interactive/session-share.ts:57-102,189`).

---

## 3. Persistent Device / Install IDs & Hashing

1. **Coding-Agent Telemetry Requests (`report-install`, `latest-version`, `models/providers/<id>`):**
   - **Zero persistent IDs transmitted.** `reportInstallTelemetry` sends only `?version=<VERSION>` and `User-Agent`; it does not read `deviceId` or `trackingId` (`interactive-mode.ts:1529-1534`).
2. **`deviceId` (`~/.pi/agent/settings.json`, global settings only):**
   - Generated **lazily on first provider login use** via `randomUUID()` (`packages/coding-agent/src/core/settings-manager.ts:1095-1102`), invoked only during interactive/CLI provider authentication (`packages/coding-agent/src/main.ts:147`; `packages/coding-agent/src/cli/auth-login.ts:72`; `interactive-mode.ts:6595`).
   - Used solely as `ext_agent_host_id: urn:uuid:<deviceId.toLowerCase()>` in OpenAI ChatGPT OAuth (`packages/ai/src/auth/oauth/openai-chatgpt.ts:226-260`). Explicitly stripped from `/bug` diagnostic bundles (`packages/coding-agent/src/core/bug-report.ts:61-64`).
3. **`trackingId` (`~/.pi/agent/settings.json`):**
   - Generated via `randomUUID()` **only** if a user runs under `PI_EXPERIMENTAL=1` and opts into `enableAnalytics` during first-time setup (`settings-manager.ts:1080-1088`; `experimental.ts:1-3`). Currently **dormant/unwired** to any network endpoint, and stripped from `/bug` reports (`bug-report.ts:61-64`).
4. **LLM Request Session IDs (Per-Session, Non-Device):**
   - `sessionManager.getSessionId()` is passed into provider transports (`packages/coding-agent/src/core/sdk.ts:427`). Depending on provider compat flags (`openai-completions.ts:776-786,834-837,1699-1700`, `openai-responses.ts:311-320,364`, `anthropic-messages.ts:239,1204-1207`, `identity.ts:130-135`), per-session IDs may be sent as `prompt_cache_key`, `x-session-id`, `session_id`/`session-id`, `x-client-request-id`, `x-session-affinity`, or `x-opencode-session`.
5. **P2 Server & Desktop Persistent UUIDs (`apps/p2`, `apps/p2-mac`):**
   - `<data>/device.json` (`randomUUID()` created lazily for P2 login/backup sync, `apps/p2/plugins/backup/target.ts:83-97`, `apps/p2/plugins/login/logins.ts:54`) and `radius.json` (`serverId` UUID for Radius tunneling, `apps/p2/plugins/radius/brain.ts:31,109,259`, `apps/p2-mac/Sources/P2Mac/RadiusAccount.swift:130`).
6. **Hashing:**
   - **No telemetry ID hashing or cross-provider SHA-1/SHA-256 user/repo fingerprinting** exists in telemetry code.

---

## 4. Exact Outbound Telemetry / Analytics / Crash / OTEL / Relay / Update Endpoints

| Endpoint | Purpose / Channel | Default State | Required to Stop | Source Location |
|---|---|---|---|---|
| `https://pi.dev/api/report-install?version=<v>` | Anonymous install/upgrade ping (interactive only) | **ON** (first launch & version upgrade) | `PI_TELEMETRY=0`, `/settings` `"Install telemetry": false`, `enableInstallTelemetry: false`, or `--offline`/`PI_OFFLINE=1` | `interactive-mode.ts:1434,1446,1520-1537` |
| `https://pi.dev/api/latest-version` | CLI update availability check (interactive startup) | **ON** | `PI_SKIP_VERSION_CHECK=1`, `--offline`, or `PI_OFFLINE=1` (`PI_TELEMETRY=0` does NOT stop) | `version-check.ts:5,55,97-109`; `interactive-mode.ts:1256` |
| `https://pi.dev/api/models/providers/<id>?types=chat,image,classifier` | Remote model catalog overlay (interactive & RPC startups, **only for configured built-in providers**) | **ON** (4h TTL; every startup for OpenRouter) | `--offline` or `PI_OFFLINE=1` (`PI_TELEMETRY=0` does NOT stop) | `remote-catalog-provider.ts:14,16,88-100,127,200,231`; `model-runtime.ts:229-249`; `models.ts:643-653` |
| `https://pi.dev/api/installer/releases/...` | Managed self-update release/manifest download | Manual (`pi update`) | N/A | `package-manager-cli.ts:55,96-102` |
| `https://pi.dev/oauth` | MCP OAuth Client ID Metadata Document URL | Manual (`/mcp login` or OAuth challenge) | N/A | `extensions/mcp/oauth.ts:160-181` |
| `https://pkg.earendil.com/-/apps/pi-mac/nightly/appcast.xml` | `apps/p2-mac` Sparkle auto-update feed (packaged macOS builds only) | **ON** (packaged builds with `SUFeedURL`) | Unpackaged/dev build or disable Sparkle updates | `apps/p2-mac/scripts/package-app.sh:22,88`; `Updater.swift:13-14` |
| `https://github.com/sharkdp/fd/releases/...` & `https://github.com/BurntSushi/ripgrep/releases/...` | Auto-download `fd` / `rg` when missing from `~/.pi/agent/bin` and `PATH` | **ON** (interactive startup if binary absent) | Pre-install `fd`/`rg` or `--offline`/`PI_OFFLINE=1` | `tools-manager.ts:42-58,99-127,247-272,337-375`; `interactive-mode.ts` |
| Configured npm registry / git remotes | Startup package update checks for unpinned packages | **ON** (interactive startup if unpinned packages exist) | Pin package versions or `--offline`/`PI_OFFLINE=1` | `package-manager.ts:1396-1459`; `interactive-mode.ts:1263,1352-1368` |
| `https://radius.pi.dev` (`PI_RADIUS_GATEWAY`) `/v1/bug-reports` & `/v1/artifacts` | Manual `/bug` upload and `/share` Radius upload | **Manual only** | Never runs automatically | `bug-report-upload.ts:12-25`; `session-share.ts:133-145` |
| GitHub Gist (`gh gist create --public=false`) + `https://pi.dev/session/` (`PI_SHARE_VIEWER_URL`) | Manual `/share` fallback when Radius unconfigured | **Manual only** | Never runs automatically | `session-share.ts:85-102,189,218`; `config.ts:617` |
| Configured LLM / OAuth / HF endpoints (`openrouter.ai`, `integrate.api.nvidia.com`, `opencode.ai`, `platform.claude.com`, `huggingface.co`, etc.) | Inference + `User-Agent`, `PI_ATTRIBUTION` headers, OpenRouter `/models/user`, OAuth PKCE | Active inference / login / `/llama` | `PI_TELEMETRY=0` stops `coding-agent` `PI_ATTRIBUTION` headers; `User-Agent` stays `detailed` | `identity.ts:75-124`; `anthropic.ts:16,23`; `huggingface.ts:5` |

*Note on Crash Reporting:* `crashes.json` (`packages/coding-agent/src/core/crash-log.ts:1-45`) is strictly local; there is **no automatic crash upload**.
*Note on IP / Geo Handling:* Direct HTTP calls to `pi.dev`, `pkg.earendil.com`, `radius.pi.dev`, and `github.com` carry no IP stripping or anonymizing proxy; `pi.dev` backend source is closed/not in this repository.

---

## 5. Exact Data & Properties Collected

1. **Install/Upgrade Telemetry (`GET https://pi.dev/api/report-install`):**
   - Query parameter: `version` (`VERSION`) (`interactive-mode.ts:1529`).
   - Header: `User-Agent: pi/<version> (<process.platform>; node/<v> | bun/<v>; <process.arch>)` (`interactive-mode.ts:1531`, `pi-user-agent.ts:4-6`, `identity.ts:66-82`).
   - Transport layer: client IP visible to `pi.dev`. No UUID, hostname, username, cwd, repo, prompt, or model list is sent.
2. **Version Check (`GET https://pi.dev/api/latest-version`):**
   - Headers: `User-Agent` (same detailed OS/runtime/arch string) and `accept: application/json` (`version-check.ts:57-64`).
3. **Remote Model Catalog Refresh (`GET https://pi.dev/api/models/providers/<providerId>?types=chat,image,classifier`):**
   - Path leaks **which built-in provider IDs are locally authenticated/configured** (`models.ts:643-653`, `remote-catalog-provider.ts:88-90,127`), plus `User-Agent` (`remote-catalog-provider.ts:95`), optional `If-None-Match` ETag (`remote-catalog-provider.ts:155-156`), and client IP.
4. **LLM Request Identity & Custom Model Handling (`models.json`):**
   - **Zero custom model telemetry to `pi.dev`:** `model-runtime.ts:229-241` wraps only `builtinProviderCatalog.builtinProviders()`, so private/custom provider IDs and model slugs in `models.json` are never sent to `pi.dev/api/models/providers/...`.
   - **Headers sent to configured LLM / custom endpoints:**
     - `User-Agent`: Always `pi/<version> (<platform>; <runtime>; <arch>)` (`client-identity.ts:12`, `identity.ts:75-82,141-143`, except Anthropic OAuth which uses `claude-cli/<claudeCodeVersion>`, `anthropic-messages.ts:1190`), overridable via `model.headers`.
     - `PI_ATTRIBUTION` (when `enableInstallTelemetry` / `PI_TELEMETRY` is on): `HTTP-Referer: https://pi.dev`, `X-OpenRouter-Title: pi`, `X-OpenRouter-Categories: cli-agent` for OpenRouter; `X-BILLING-INVOKE-ORIGIN: Pi` for NVIDIA NIM; `x-opencode-client: pi` for OpenCode (`identity.ts:92-124`).
     - Session affinity / prompt cache keys: `sendSessionAffinityHeaders` defaults to **`false`** for non-OpenRouter `openai-completions` and `anthropic-messages` custom endpoints (`openai-completions.ts:1699`, `anthropic-messages.ts:239`), whereas `openai-responses` sends `x-client-request-id: <sessionId>` (and `prompt_cache_key: <sessionId>` unless `cacheRetention === "none"` or `promptCacheKeyMode === "disabled"`) by default (`openai-responses.ts:68-70,90-91,311-320,356-367`), and `openai-completions` sends `prompt_cache_key` for `api.openai.com` or `cacheRetention === "long"` (`openai-completions.ts:825-839`).
5. **Manual `/bug` Reports:**
   - Redacts `trackingId`, `deviceId`, secrets matching `SENSITIVE_KEY`, and credential URL params (`bug-report.ts:19-24,51-64`). Includes Pi version, `User-Agent`, Node/Bun runtime, OS release/version, shell basename, terminal flags (`TERM`, `TERM_PROGRAM`, `tmux`, `ssh`, `ci`), **names only** of `PI_*` env vars (`bug-report.ts:66-90`), provider config without secrets, crash diagnostics, and optional user-approved transcript/summary.

---

## 6. Docs / Privacy Policy vs Code Discrepancies

1. **Non-existent `/privacy` Command Advertised in Experimental Setup UI (`first-time-setup.ts:59`):**
   - `packages/coding-agent/src/modes/interactive/components/first-time-setup.ts:59` (shown when `PI_EXPERIMENTAL=1`, `experimental.ts:1-3`) tells users: *"You can observe what is shared using /privacy and make changes anytime in settings.json."*
   - Code check confirms **no `/privacy` command exists** anywhere in `packages/` or `apps/`.
2. **CLI `--help`, `/settings`, and Schema Omit Provider Attribution Headers (`args.ts:469`, `settings-selector.ts:577`, `settings-schema.ts:301` vs `docs/settings.md:171`, `docs/environment-variables.md:87`):**
   - `docs/settings.md:171` and `docs/environment-variables.md:87` accurately note that `enableInstallTelemetry` / `PI_TELEMETRY` control *"anonymous install/update reporting and selected provider attribution headers."*
   - In contrast, `packages/coding-agent/src/cli/args.ts:469` (`"Override install telemetry..."`), `settings-selector.ts:577` (`"Send an anonymous version/update ping after changelog-detected updates"`), and `settings-schema.ts:301` omit that the toggle also gates `PI_ATTRIBUTION` headers (`HTTP-Referer`, `X-OpenRouter-Title`, `X-OpenRouter-Categories`, `X-BILLING-INVOKE-ORIGIN`, `x-opencode-client`) sent to third-party inference providers.
3. **Undisclosed Provider-Configuration Disclosure via `pi.dev/api/models/providers/<id>` (`docs/models.md:15`, `docs/environment-variables.md:84-87` vs `models.ts:643-653`, `remote-catalog-provider.ts:127,200`):**
   - `docs/models.md:15` and `README.md:66` describe runtime catalog overlays from `pi.dev`, and `docs/providers.md:122-124` discloses OpenRouter `/models/user` refreshes, **but no documentation discloses** that startup catalog requests to `pi.dev/api/models/providers/<id>` are selectively sent **only for locally authenticated/configured built-in providers** (revealing active provider usage + detailed OS/runtime `User-Agent` + IP to `pi.dev` in both interactive and RPC modes), nor that `PI_TELEMETRY=0` leaves detailed `User-Agent` and `latest-version` active.
4. **Undocumented `apps/p2-mac` Sparkle Updater & `PI_OFFLINE=0` Asymmetry:**
   - Packaged macOS app auto-update polling to `pkg.earendil.com` (`Updater.swift:13-14`; `package-app.sh:22,88`) is undocumented in user privacy/env docs, and setting `PI_OFFLINE=0` / `PI_OFFLINE=false` against `args.ts:467` creates inconsistent suppression across `main.ts:610` vs `interactive-mode.ts:1246,1521`, `version-check.ts:55`, and `model-runtime.ts:248`.

---

## 7. Zero-Telemetry / Negative-Check Verification

Pi is **not** a zero-telemetry repository (it defaults ON for `report-install`, `latest-version`, credential-gated `models/providers/<id>` catalog fetches, `PI_ATTRIBUTION` headers, package/tool checks, and packaged `p2-mac` Sparkle polling). Targeted read-only `rg` checks across `repos/pi` verified the absence of:

- **`DO_NOT_TRACK` / `NO_TELEMETRY` Support (`0` matches):**
  - Command: `rg -n "DO_NOT_TRACK|NO_TELEMETRY" repos/pi -g "!package-lock.json"`
  - Result: **0 matches** (exit code 1).
- **Third-Party Analytics / Crash Reporting SDKs (`0` code/dependency usages):**
  - Command: `rg -ni "\b(posthog|mixpanel|amplitude|sentry|bugsnag|rollbar|raygun|crashlytics|honeycomb|fullstory|logrocket|statsig|datadog)\b|heap\.io|segmentio|segment\.com|analytics\.js" repos/pi/packages/ repos/pi/apps/ repos/pi/package.json repos/pi/.github/ -g "!package-lock.json" -g "!CHANGELOG.md"`
  - Result: **0 analytics/crash SDK dependencies or usages**; hits were exclusively `.github/workflows/{pr,issue}-gate.yml` `sentry[bot]` allowlists, MCP OAuth docs/example config/tests for user-configured `https://mcp.sentry.dev/mcp`, voice audio `amplitude` math/tests, `packages/telemetry/README.md` adapter prose, and `apps/p3/docs/archive/` design notes.
- **OpenTelemetry Exporter SDKs (`0` matches outside `package-lock.json`):**
  - Command: `rg -n "@opentelemetry|OTEL_EXPORTER" repos/pi/packages/ repos/pi/apps/ repos/pi/package.json -g "!package-lock.json"`
  - Result: **0 matches** (exit code 1). (`@opentelemetry/api` appears in `package-lock.json` solely as an optional peer dependency of `vitest`.)
- **OS / Hardware Machine-ID Fingerprinting (`0` matches):**
  - Command: `rg -n "/etc/machine-id|/var/lib/dbus/machine-id|IOPlatformUUID|MachineGuid|node-machine-id|systeminformation" repos/pi/packages/ repos/pi/apps/`
  - Result: **0 matches** (exit code 1).
- **Competitor AI Tool Credential/Directory Harvesting (`0` runtime reads):**
  - Command: `rg -n "~/\.(claude|codex|cursor|opencode|gemini|windsurf|aider)|\.claude/|\.codex/|\.cursor/|\.opencode/|\.gemini/|\.windsurf/|\.aider" repos/pi/packages/ repos/pi/apps/ -g "!CHANGELOG.md"`
  - Result: **0 runtime auth/config reads** across Pi packages and apps (matched only optional sample extension `packages/coding-agent/examples/extensions/claude-rules.ts`, `examples/extensions/README.md`, and `apps/p3/docs/archive/hermes-opencode-teams.md`).

---

```json
{
  "slug": "pi",
  "platform": "Pi",
  "group": "B",
  "repo": "earendil-works/pi",
  "sha": "c11a73bfba1310dc4075ad836254ea87003f409b",
  "commit_date": "2026-11-25T00:53:26+01:00",
  "license": "MIT",
  "status": "Active (badlogic/pi-mono redirects here)",
  "default_state": "Mixed / Light ON by default: zero third-party analytics/crash/OTEL SDKs and no usage/prompt telemetry, but enableInstallTelemetry defaults to true (sending GET pi.dev/api/report-install on first install/upgrade and PI_ATTRIBUTION headers to OpenRouter/NVIDIA/OpenCode), plus default-ON pi.dev/api/latest-version checks, credential-gated pi.dev/api/models/providers/<id> catalog fetches (interactive & RPC), npm/git package update checks, missing fd/rg GitHub downloads, and packaged p2-mac Sparkle appcast polling; experimental enableAnalytics and apps/p2 local SQLite telemetry default OFF.",
  "opt_out_controls": [
    "/settings -> Install telemetry: false, settings.json enableInstallTelemetry: false, or PI_TELEMETRY=0|false|no (stops report-install and coding-agent PI_ATTRIBUTION only)",
    "PI_SKIP_VERSION_CHECK=1 (stops pi.dev/api/latest-version only; no /settings toggle)",
    "--offline or PI_OFFLINE=1|true|yes (master startup network killswitch stopping report-install, latest-version, pi.dev model catalogs, package update checks, and fd/rg downloads)",
    "SDK callers of @earendil-works/pi-agent-core / @earendil-works/pi-durable must explicitly pass identity: { attribution: null }"
  ],
  "do_not_track": false,
  "opt_out_bypasses": [
    "PI_TELEMETRY=0 and /settings Install telemetry=false leave pi.dev/api/latest-version, credential-gated pi.dev/api/models/providers/<configured-provider>, detailed OS/runtime User-Agent headers, npm/git package checks, fd/rg downloads, and p2-mac Sparkle polling active",
    "Standalone @earendil-works/pi-agent-core (agent-loop.ts:41,415) and @earendil-works/pi-durable (harness.ts:194,200-204) default attribution to PI_ATTRIBUTION without consulting PI_TELEMETRY or settings.json",
    "PI_OFFLINE=0 or PI_OFFLINE=false fails to trigger offlineMode in main.ts:610 / package-manager.ts:85 / tools-manager.ts:16 while still suppressing startup catalog/version/install-telemetry calls in interactive-mode.ts:1246,1521, version-check.ts:55, and model-runtime.ts:248"
  ],
  "persistent_ids": [
    "No persistent telemetry ID transmitted by default (report-install sends only ?version=<v> + User-Agent)",
    "~/.pi/agent/settings.json -> deviceId (randomUUID created lazily on first provider login; sent only as ext_agent_host_id=urn:uuid:<deviceId> in OpenAI ChatGPT OAuth; stripped from /bug)",
    "~/.pi/agent/settings.json -> trackingId (randomUUID created only under PI_EXPERIMENTAL=1 when user opts into enableAnalytics; dormant/unwired; stripped from /bug)",
    "apps/p2 <data>/device.json (randomUUID for P2 login/backup) and radius.json serverId (randomUUID for Radius tunneling)"
  ],
  "hashing": "None for telemetry IDs; zero unsalted cross-provider user/repo/skill hashing.",
  "external_auth_reads": "None from competitor agents (zero reads of ~/.claude, ~/.codex, ~/.cursor, ~/.opencode, ~/.gemini, ~/.windsurf, or ~/.aider). Reads own ~/.pi/* dirs, ~/.agents/skills + ancestor .agents/skills, project/global AGENTS*.md/CLAUDE.md (unless --no-context-files), and standard provider credentials only when used (GOOGLE_APPLICATION_CREDENTIALS or ~/.config/gcloud/application_default_credentials.json for google-vertex; HF_TOKEN or ~/.cache/huggingface/token for /llama; gh auth status for /share gist fallback).",
  "telemetry_endpoints": [
    "https://pi.dev/api/report-install?version=<v> (interactive install/upgrade beacon, default ON)",
    "https://pi.dev/api/latest-version (interactive startup version check, default ON)",
    "https://pi.dev/api/models/providers/<id>?types=chat,image,classifier (interactive & RPC catalog overlay for authenticated built-in providers only, default ON)",
    "https://pkg.earendil.com/-/apps/pi-mac/nightly/appcast.xml (packaged apps/p2-mac Sparkle update feed, default ON)",
    "https://github.com/sharkdp/fd/releases/* & https://github.com/BurntSushi/ripgrep/releases/* (automatic binary download if fd/rg missing, default ON)",
    "Configured npm registry / git remotes (interactive startup package update checks, default ON)",
    "https://radius.pi.dev/v1/{bug-reports,artifacts} (manual /bug and /share uploads only)",
    "https://pi.dev/api/installer/releases & https://pi.dev/oauth & https://pi.dev/session/ (manual pi update, MCP OAuth metadata, /share viewer link)",
    "Configured LLM providers receiving PI_ATTRIBUTION headers (OpenRouter, NVIDIA NIM, OpenCode) when enableInstallTelemetry is true"
  ],
  "ip_geo_handling": "No IP suppression or anonymizing proxy; direct HTTPS requests to pi.dev, pkg.earendil.com, radius.pi.dev, and github.com expose client IP to closed-source backends.",
  "custom_model_handling": "Clean toward Pi backend: model-runtime.ts wraps only built-in providers, so private/custom models.json provider names and model slugs are never sent to pi.dev. To the configured inference endpoint itself, Pi sends detailed User-Agent by default (and x-client-request-id/prompt_cache_key for openai-responses; sendSessionAffinityHeaders defaults false on non-OpenRouter openai-completions/anthropic-messages).",
  "docs_discrepancies": [
    "first-time-setup.ts:59 (PI_EXPERIMENTAL=1) advertises a nonexistent /privacy command ('You can observe what is shared using /privacy')",
    "args.ts:469 (--help), settings-selector.ts:577 (/settings), and settings-schema.ts:301 omit that PI_TELEMETRY / enableInstallTelemetry also gates PI_ATTRIBUTION provider headers (disclosed in docs/settings.md:171 and docs/environment-variables.md:87)",
    "Docs omit that pi.dev/api/models/providers/<id> selectively runs only for authenticated built-in providers (leaking active AI provider selection to pi.dev), omit detailed User-Agent persistence when PI_TELEMETRY=0, and omit packaged p2-mac Sparkle appcast polling"
  ],
  "privacy_rating": "C",
  "one_line_verdict": "Minimal telemetry footprint meriting C (zero analytics/crash/OTEL SDKs, zero persistent telemetry IDs, zero custom-model leaks, zero competitor auth harvesting, and a clean --offline switch), held below B/A because DO_NOT_TRACK is ignored, PI_TELEMETRY=0 leaves pi.dev/api/latest-version, credential-gated pi.dev/api/models/providers/<id> (leaking configured AI providers), detailed OS/runtime User-Agent, agent-core/durable attribution, and p2-mac Sparkle polling active without IP suppression, and UI/CLI docs misstate controls including a nonexistent /privacy command."
}
```
