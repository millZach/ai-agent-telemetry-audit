# Telemetry & Privacy Forensic Audit: Kilo Code (`Kilo-Org/kilocode`)

## Header Metadata

- **Slug:** `kilocode`
- **Platform:** Kilo Code (monorepo shipping CLI/TUI/HTTP server, VS Code extension, JetBrains plugin, and docs site)
- **Group:** B
- **Repository:** `https://github.com/Kilo-Org/kilocode`
- **Branch:** `main`
- **Commit SHA:** `b12e612bd3ad9c4073ea57caaf0197fad7f98d63`
- **Commit Date:** `2026-11-24T14:57:22-05:00`
- **Commit Subject:** `fix(jetbrains): remove Cloud compute badges from Agent Manager tree (#16527)`
- **License:** MIT
- **Status:** Active
- **Architectural Note (Kilo Code vs. Bundled OpenCode Base & Client Surface Inventory):**
  - Kilo Code is built on a forked OpenCode core (`packages/opencode/`, `packages/core/`, `packages/tui/`, `packages/server/`), layered with Kilo-owned packages (`packages/kilo-telemetry/`, `packages/kilo-gateway/`, `packages/kilo-indexing/`, `packages/kilo-vscode/`, `packages/kilo-jetbrains/`, `packages/kilo-console/`, `packages/kilo-docs/`).
  - Neither a `packages/desktop` (Electron/Tauri) nor a `packages/app` mobile client exists in this repository (`packages/*` listing); upstream OpenCode's `@sentry/solid` and `@sentry/vite-plugin` desktop Sentry dependencies are explicitly stripped by `script/upstream/transforms/transform-package-json.ts:330-333`.
  - Kilo Code replaces upstream AI SDK tracing (`experimental_telemetry: { isEnabled: false }` in `packages/opencode/src/session/llm.ts:482` and `packages/opencode/src/kilocode/agent/index.ts:491-495` / `packages/opencode/src/agent/agent.ts:552-554`) and upstream `ShareNext` (`https://opncd.ai` in `packages/opencode/src/share/share-next.ts:225`) with Kilo-owned `@kilocode/kilo-telemetry` (PostHog), `@kilocode/kilo-gateway`, and `KiloSessions` (`https://ingest.kilosessions.ai` via `packages/opencode/src/kilocode/bootstrap.ts:54` and `packages/opencode/src/kilocode/session/index.ts:266-283`).
- **Privacy Rating:** **F** (Severe Red Flags)

---

## 1. Default State & Opt-Out Mechanism

### Default State
- **Product telemetry (`@kilocode/kilo-telemetry` -> PostHog) is ON by default.**
  - In the global configuration schema, `experimental.openTelemetry` defaults to `true`:
    `packages/core/src/v1/config/config.ts:352-354`.
  - Every CLI/server invocation (except `--help`/`--version`, which return early at `packages/opencode/src/kilocode/cli/setup.ts:73-74`) enters `KiloCli.bootstrap` (`packages/opencode/src/index.ts:89`), reads global config `cfg.experimental?.openTelemetry !== false` (`packages/opencode/src/kilocode/cli/setup.ts:93-103`), initializes `Telemetry.init` (`packages/kilo-telemetry/src/telemetry.ts:119-145`), emits `CLI Start`, schedules background flush (`setup.ts:130-133`), and emits `CLI Exit` + bounded 2s shutdown flush (`setup.ts:141,149`).
- **Automatic Kilo Cloud session & diff ingest (`KiloSessions`) is ON by default whenever signed into Kilo.**
  - `KilocodeBootstrap.init` unconditionally initializes `kilo.init()` (`packages/opencode/src/kilocode/bootstrap.ts:54`), which automatically registers every created session and streams session metadata, messages, parts, models, and cumulative diffs to `https://ingest.kilosessions.ai` whenever valid Kilo credentials exist (`packages/opencode/src/kilo-sessions/kilo-sessions.ts:201-236,281-315,776-953`).
- **Optional OTLP export (OpenCode core) is OFF by default** (`packages/core/src/observability/otlp.ts:7,50-56`), requiring explicit `OTEL_EXPORTER_OTLP_ENDPOINT`.
- **Dormant free-model session export pipeline (`SessionExport`) is hard-disabled in source:** `export const enabled = false` (`packages/opencode/src/kilocode/session-export/session-export.ts:35`), gated at `packages/opencode/src/kilocode/bootstrap.ts:75`.

### Precedence & Opt-Out Controls
Telemetry gate logic lives in `Telemetry.level` (`packages/kilo-telemetry/src/telemetry.ts:147-151`):
```ts
const env = process.env.KILO_TELEMETRY_LEVEL?.trim().toLowerCase()
const allowed = !env || env === "all" || env === "on" || env === "true" || env === "1"
return configured && allowed && (host ?? process.env.KILO_TELEMETRY_HOST_DISABLED !== "1")
```
This enforces a strict **logical AND** across three layers (confirmed in tests `packages/kilo-telemetry/src/__tests__/identity-delivery.test.ts:75-100`):
1. **Global Kilo config (`configured`):** `experimental.openTelemetry: false` in global config (`setup.ts:93-103`). Only global config is consulted on startup (`Telemetry.init`), and `POST /telemetry/level` writes global config (`packages/opencode/src/kilocode/server/httpapi/handlers/telemetry.ts:37-43`).
2. **Environment variable (`allowed`):** Any non-empty `KILO_TELEMETRY_LEVEL` value other than `"all"`, `"on"`, `"true"`, or `"1"` (case-insensitive) disables PostHog telemetry. `KILO_TELEMETRY_LEVEL=all` **cannot** override `experimental.openTelemetry: false`.
3. **Host editor gate (`host`):**
   - **VS Code (`packages/kilo-vscode`):** `ServerManager.startServer` spawns `kilo serve` with `KILO_TELEMETRY_HOST_DISABLED: vscode.env.isTelemetryEnabled ? "0" : "1"` (`packages/kilo-vscode/src/services/cli-backend/server-manager.ts:169`). Runtime VS Code telemetry changes call `TelemetryProxy.setHostEnabled(vscode.env.isTelemetryEnabled)` -> `POST /telemetry/setEnabled` -> `Telemetry.setHostEnabled` (`packages/kilo-vscode/src/services/telemetry/telemetry-proxy.ts:40,69-81`; `packages/opencode/src/kilocode/server/httpapi/handlers/telemetry.ts:24-30`). Additionally, Kilo exposes its own **Enable telemetry** switch in **Settings > About Kilo Code** (`packages/kilo-vscode/webview-ui/src/components/settings/AboutKiloCodeTab.tsx:268-289`), handled by `setTelemetryLevel` -> `POST /telemetry/level` (`packages/kilo-vscode/src/KiloProvider.ts:1614-1623`), which toggles `experimental.openTelemetry` in global config.
   - **JetBrains (`packages/kilo-jetbrains`):** Spawns the CLI with `KILO_TELEMETRY_LEVEL=off` **only** in dev mode (`packages/kilo-jetbrains/backend/src/main/kotlin/ai/kilocode/backend/cli/KiloBackendCliManager.kt:616`). It does **not** consult IntelliJ IDE Data Sharing settings, but exposes a **"Send usage data"** toggle under **Settings > Tools > Kilo Code > Advanced** (`packages/kilo-jetbrains/frontend/src/main/kotlin/ai/kilocode/client/settings/AdvancedConfigurable.kt:23-102`; `AdvancedSettingsUi.kt:39,50-54`; `packages/kilo-jetbrains/frontend/src/main/resources/messages/KiloBundle.properties:763-768`) wired to `KiloAppService.setTelemetryLevelAsync` -> `KiloBackendTelemetry.setLevel` -> `POST /telemetry/level` (`KiloBackendAppService.kt:326-334`; `KiloBackendTelemetry.kt:48-52`).
- **`DO_NOT_TRACK` standard:** **Not supported / ignored.** Zero runtime source files inspect `DO_NOT_TRACK` (only `packages/opencode/test/kilocode/startup-speed.test.ts:52` sets it in a test harness, and `packages/kilo-docs/pages/getting-started/settings/product-telemetry.md:73` explicitly notes it is unsupported).
- **Separate non-PostHog opt-outs:**
  - `KILO_DISABLE_SESSION_INGEST=1` or `"true"` disables `KiloSessions` auto-sync (`packages/opencode/src/kilo-sessions/kilo-sessions.ts:312-313,778,1591`).
  - `KILO_DISABLE_MODELS_FETCH=1` or `"true"` (`packages/core/src/flag/flag.ts:58`) and `KILO_DISABLE_AUTOUPDATE=1` or `"true"` (`packages/core/src/flag/flag.ts:47`, `packages/opencode/src/cli/upgrade.ts:10`) / `autoupdate: false` disable catalog fetches and npm auto-upgrades.

### Opt-Out Bypasses & Residual Behavior When Opted Out
1. **Unconditional persistent machine ID creation (`telemetry-id`) & transmission:** Even when telemetry is disabled (`enabled === false`), `Telemetry.init` unconditionally executes `Identity.getMachineId()` (`packages/kilo-telemetry/src/telemetry.ts:141`), generating and persisting a UUID v4 in `<Global.Path.data>/telemetry-id` (`packages/kilo-telemetry/src/identity.ts:42-52`). JetBrains similarly creates `<SystemPath>/kilo/machine-id` unconditionally (`KiloBackendCliManager.kt:655-666`), and VS Code passes `vscode.env.machineId` unconditionally (`server-manager.ts:160`). That ID is still sent on every Kilo Gateway LLM call via `X-KILOCODE-MACHINEID` (`packages/opencode/src/session/llm/request.ts:187-189,261`) and to the auto-routing worker in the JSON body (`packages/opencode/src/session/llm.ts:117,126`; `packages/opencode/src/kilocode/auto-routing/router.ts:141`) even when telemetry is completely turned off.
2. **Automatic `KiloSessions` cloud upload ignores all telemetry and share opt-outs:** `KiloSessions` ignores `experimental.openTelemetry: false`, `KILO_TELEMETRY_LEVEL=off`, `KILO_TELEMETRY_HOST_DISABLED=1`, and `KILO_DISABLE_SHARE=1` (`packages/opencode/src/kilo-sessions/kilo-sessions.ts:311-313,778,1591`). Any Kilo-authenticated session uploads prompts, replies, tool calls/outputs, and diffs to `https://ingest.kilosessions.ai` unless the undocumented `KILO_DISABLE_SESSION_INGEST=1` env var is set.
3. **IDE -> Localhost CLI HTTP forwarding when only Kilo telemetry is disabled:** In VS Code, `TelemetryProxy.capture` only checks `vscode.env.isTelemetryEnabled` (`telemetry-proxy.ts:52`), and in JetBrains, `KiloTelemetryService.send` / `KiloBackendTelemetry.capture` only check sandbox/dev mode (`KiloTelemetryService.kt:52`; `KiloBackendTelemetry.kt:29`). Thus UI events are still POSTed to `http://127.0.0.1:<port>/telemetry/capture` when `experimental.openTelemetry` is `false` or `KILO_TELEMETRY_LEVEL=off` (though the CLI server drops them at `Client.capture` in `packages/kilo-telemetry/src/client.ts:63`).
4. **Docs site analytics (`kilo-docs`):** `packages/kilo-docs/instrumentation-client.ts:3-10` unconditionally initializes browser `posthog-js` whenever `NEXT_PUBLIC_POSTHOG_KEY` is built in (`.github/workflows/docs-build.yml:27`), with no `DO_NOT_TRACK` check.

---

## 2. External Auth / Local Non-Workspace File Reads

| Target Path / Source | Reader Location | Condition / Trigger | Purpose & Outbound Impact |
|---|---|---|---|
| `~/.conda/aau_token` + `conda info --json --offline` | `packages/kilo-telemetry/src/conda.ts:8-49` | **Automatic when PostHog telemetry is enabled** (`telemetry.ts:141,164,171`). Gated off when telemetry is disabled. | Extracts Anaconda's 22-char `aauClientToken` (`c/([A-Za-z0-9_-]{22})`) and attaches it to **every** PostHog event (`telemetry.ts:16,173,198`) as a cross-product telemetry join key. |
| `~/.kilocode/cli/config.json` | `packages/kilo-gateway/src/auth/legacy-migration.ts:11,38-60`; called at `packages/opencode/src/kilocode/cli/setup.ts:118-121` | Automatic on CLI startup if no `"kilo"` entry exists in `auth.json`. | Reads Kilo's own legacy CLI config to migrate `kilocodeToken` / `kilocodeOrganizationId`; migrated token is then used by `Telemetry.updateIdentity` and `KiloSessions`. |
| `~/.kilocode/secrets.json`, `~/.kilocode/globalStorage/`, JetBrains `kilocode-extension-storage.xml` | `packages/kilo-jetbrains/backend/src/main/kotlin/ai/kilocode/backend/migration/LegacyV5Sources.kt:6-60` | Legacy v5 JetBrains plugin migration flow (`KiloMigrationService.kt`). | Reads Kilo's own legacy v5 secrets, MCP settings, custom modes, and task history for migration. |
| Sibling VS Code `roovscode.*` / `rooveterinaryinc.*` `tasks/` & `roo-cline.customStoragePath` | `packages/kilo-vscode/src/roo-import/service.ts:12-58` | User-initiated Roo Code import from Settings (`AboutKiloCodeTab.tsx:346-350`). | Scans Roo Code's VS Code extension `globalStorage` task index to import tasks locally. |
| `~/.claude/CLAUDE.md` & `~/.claude/skills/` | `packages/opencode/src/session/instruction.ts:69-72`; `packages/opencode/src/skill/index.ts:227-234` | **CLI: ON by default** unless `KILO_DISABLE_CLAUDE_CODE` / `KILO_DISABLE_CLAUDE_CODE_PROMPT` / `KILO_DISABLE_CLAUDE_CODE_SKILLS` is set (`packages/core/src/flag/flag.ts:61-64`). **VS Code & JetBrains: OFF by default** (`server-manager.ts:165`; `KiloBackendCliManager.kt:617`) unless `claudeCodeCompat` is enabled (`packages/kilo-vscode/package.json:1115-1118`). | Inherited/adapted OpenCode compatibility: loads Claude instructions/skills into agent context. Note: no non-Kilo Anthropic OAuth credentials are read here. |
| `~/.claude.json`, `~/.claude/CLAUDE.md`, `~/.claude/skills/` | `packages/opencode/src/kilocode/config/claude-migration.ts:16-69`; `packages/kilo-vscode/package.json:1089-1093`; `server-manager.ts:41-43,103-106,170` | **Opt-in only** (`KILO_EXPERIMENTAL_CLAUDE_MIGRATION` / `kilo-code.new.experimental.claudeMigration` defaults `false`, `flag.ts:74`). | One-time local migration of Claude instructions, skills, and disabled MCP configs into Kilo config. |
| `~/.claude/ide/*.lock` (+ Zed SQLite DB) | `packages/tui/src/editor.ts:56-100` | When TUI resolves active IDE lock/selection in `directory`. | Discovers local VS Code / Claude IDE WebSocket server `ws://127.0.0.1:<port>` and local lock `authToken` to read editor context. |
| `~/.claude/projects/<slug>/*.jsonl` & `~/.codex/sessions/**/rollout-*.jsonl` | `packages/opencode/src/kilocode/session-resume/index.ts:159-218` | **Explicit user action only** (`/resume-claude`, `/resume-codex`, `kilo import`). | Imports Claude Code or OpenAI Codex transcripts into a local Kilo session. |
| `git config` (`remote.origin.url`) / `.kilo/config.json` / `.kilocode/config.json` | `packages/opencode/src/kilocode/project-id.ts:45-93`; `packages/opencode/src/kilo-sessions/kilo-sessions.ts:1932-1940,2010-2011` | On Kilo Gateway LLM requests and `KiloSessions` sync. | Reads `git config --get remote.origin.url` / branch; sends normalized project ID (`X-KILOCODE-PROJECTID`) to Kilo Gateway and `gitUrl` (with user/password/query stripped) + `gitBranch` to `ingest.kilosessions.ai`. |

**No silent third-party credential harvesting found:** Searches across `packages/opencode/src/plugin/` and `packages/` found **no** reads of `~/.aws`, `~/.ssh`, `~/.config/gh`, `~/.gemini`, `~/.codex/auth.json`, or `~/.claude/.credentials.json` for tracking or auth piggybacking.

---

## 3. Persistent Device / Install IDs & Hashing

1. **`telemetry-id` (Persistent UUID v4, Unsalted Plaintext):**
   - Created unconditionally at `Telemetry.init` (`packages/kilo-telemetry/src/telemetry.ts:141`) in `<Global.Path.data>/telemetry-id` (`packages/kilo-telemetry/src/identity.ts:42-52`).
   - Overridden by `KILO_MACHINE_ID` when spawned from an IDE (`identity.ts:33-37`):
     - **VS Code:** Passes `vscode.env.machineId` unconditionally (`packages/kilo-vscode/src/services/cli-backend/server-manager.ts:160`).
     - **JetBrains:** Creates and passes a persistent UUID v4 from `<PathManager.getSystemPath()>/kilo/machine-id` unconditionally (`packages/kilo-jetbrains/backend/src/main/kotlin/ai/kilocode/backend/cli/KiloBackendCliManager.kt:655-666`).
   - **Used even when telemetry is OFF:** Sent in `X-KILOCODE-MACHINEID` on every `@kilocode/kilo-gateway` LLM call (`packages/opencode/src/session/llm/request.ts:183-189,261`, `packages/kilo-gateway/src/api/constants.ts:104`, `packages/kilo-gateway/src/headers.ts:107-109`) and in the `machineId` JSON field to `https://auto-routing.kiloapps.io` (`packages/opencode/src/session/llm.ts:117,126`; `packages/opencode/src/kilocode/auto-routing/router.ts:141`).
2. **Plaintext Email `distinct_id` + Cross-Identity `$create_alias` Stitching:**
   - When Kilo auth (`auth.json` `"kilo"` OAuth/API key) is present or completed (`setup.ts:123-128`; `packages/opencode/src/provider/auth.ts:235-242`), `Identity.updateFromKiloAuth` fetches the user's profile from `https://api.kilo.ai/api/profile` (`packages/kilo-gateway/src/api/profile.ts:8-39`) and stores `profile.email` (`packages/kilo-telemetry/src/identity.ts:130-132`).
   - `Identity.getDistinctId()` returns **raw plaintext `userId` (the email)** when authenticated, otherwise `machineId` (`identity.ts:55-57`).
   - `Telemetry.updateIdentity` sends PostHog `$create_alias` linking the prior anonymous `telemetry-id`/`machineId` to the plaintext email (`packages/kilo-telemetry/src/telemetry.ts:186-190`), caching completion in `<dataPath>/telemetry-alias-<sha256>` (`packages/kilo-telemetry/src/client.ts:28-31,40-44,78-92`).
   - `Identity.write` caches `{ token: sha256(token), email, fetchedAt }` locally in `<dataPath>/telemetry-profile.json` (`0o600`) for 7 days (`identity.ts:18-25,71-73,85-103,118-132`).
3. **External Conda `aauClientToken` Join Key (Plaintext):**
   - Read from `~/.conda/aau_token` + `conda info --json --offline` when telemetry is enabled (`packages/kilo-telemetry/src/conda.ts:5-50`) and sent verbatim as `aauClientToken` on PostHog events (`telemetry.ts:16,173,198`).
4. **Ephemeral OTLP Run ID (Inherited OpenCode):**
   - `runID` (non-persistent per-process ID) is attached to `opencode.run` and `service.instance.id` only if `OTEL_EXPORTER_OTLP_ENDPOINT` is configured (`packages/core/src/observability/otlp.ts:5,44-45`).

---

## 4. Exact Outbound Telemetry / Analytics / Crash / OTEL / Relay / Update Endpoints

| Category | Exact Endpoint / URL | Trigger & Consent Behavior | Source Citation |
|---|---|---|---|
| **Product Telemetry (PostHog)** | `https://us.i.posthog.com` (`phc_GK2Pxl0HPj5ZPfwhLRjXrtdz8eD7e9MKnXiFrOqnB6z`, `disableGeoip: false`) | **ON by default** via `posthog-node`. Receives CLI, IDE (via `/telemetry/capture`), MCP, indexing, marketplace, and feedback events. Stopped by config/env/host opt-out. | `packages/kilo-telemetry/src/client.ts:8-9,20-23`; `packages/opencode/src/kilocode/server/httpapi/handlers/telemetry.ts:14-22` |
| **Docs Web Analytics (PostHog)** | `/ingest/*` -> `https://us.i.posthog.com` & `https://us-assets.i.posthog.com/static/*` | `posthog-js` on docs site when `NEXT_PUBLIC_POSTHOG_KEY` is set during build (no `DO_NOT_TRACK` check). | `packages/kilo-docs/next.config.js:46-54`; `packages/kilo-docs/instrumentation-client.ts:1-11`; `packages/kilo-docs/pages/_app.tsx:84,94`; `.github/workflows/docs-build.yml:27` |
| **Release Stats Script (CI/Maintainer Only)** | `https://us.i.posthog.com/i/v0/e/` & `api.npmjs.org` / `api.github.com` | **Not shipped in runtime clients.** Maintainer script only when `POSTHOG_KEY` is set (`.github/workflows/disabled/stats.yml.disabled`). | `script/stats.ts:3-29,57,76` |
| **Kilo Session Ingest & Remote Relay** | `https://ingest.kilosessions.ai` (`/api/session`, ingest sync, title POST) & `wss://events.kiloapps.io` / `wss://ingest.kilosessions.ai` | **ON by default for all Kilo-authenticated sessions** (ignores telemetry & share opt-outs; disabled only via `KILO_DISABLE_SESSION_INGEST=1`). WebSocket remote relay active on `kilo remote` / `KILO_REMOTE=1`. | `packages/opencode/src/kilo-sessions/kilo-sessions.ts:289,1142-1144,1604-1607,1827`; `packages/kilo-gateway/src/api/constants.ts:60`; `packages/kilo-gateway/src/cloud-sessions.ts:150` |
| **Kilo API, Gateway & Auto-Routing** | `https://api.kilo.ai` (`/api/profile`, `/api/user`, `/api/users/notifications`), `https://ai-gateway.kilo.ai/api/v1`, `https://auto-routing.kiloapps.io` | Profile fetched at startup/login (`Telemetry.updateIdentity`); user/notifications fetched on demand; gateway & auto-router receive `X-KILOCODE-MACHINEID` / `machineId` even when telemetry is OFF. | `packages/kilo-gateway/src/api/constants.ts:10,19,43`; `profile.ts:9`; `notifications.ts:43`; `kilo-sessions.ts:211`; `router.ts:132-145`; `request.ts:183-189,261` |
| **Dormant Session Export** | `https://supermassive-black-hole.kiloapps.io/v1/session-export/batch` | **Hard-disabled (`enabled = false`)** in `main`; would otherwise upload free-model traces. | `packages/opencode/src/kilocode/session-export/session-export.ts:35`; `worker/endpoint.ts:1-16`; `eligibility.ts:16-22` |
| **Legacy Upstream Share (Inherited OpenCode)** | `https://opncd.ai` (or `config.enterprise.url`) `/api/share` | Inherited `ShareNext` module; Kilo session share routes call `KiloSessions.share()` (`ingest.kilosessions.ai`) instead of `ShareNext`. | `packages/opencode/src/share/share-next.ts:85-91,225`; `packages/opencode/src/kilocode/session/index.ts:266-278`; `packages/opencode/src/project/bootstrap.ts:9-49` |
| **Optional OTLP (Inherited OpenCode Core)** | `${OTEL_EXPORTER_OTLP_ENDPOINT}/v1/logs` & `${OTEL_EXPORTER_OTLP_ENDPOINT}/v1/traces` | **Strictly opt-in** (disabled unless `OTEL_EXPORTER_OTLP_ENDPOINT` is set). Independent of product telemetry opt-out. | `packages/core/src/observability.ts:11-22`; `packages/core/src/observability/otlp.ts:7,50-77` |
| **Model Catalog Sync (Inherited OpenCode Core)** | `https://models.dev/api.json` | Periodic/startup model catalog refresh unless `KILO_DISABLE_MODELS_FETCH=1`. | `packages/core/src/models-dev.ts:171,187`; `packages/core/src/flag/flag.ts:58` |
| **CLI Auto-Update / Package Manager Checks** | `https://registry.npmjs.org/@kilocode%2fcli/<channel>`, `formulae.brew.sh/api/formula/kilo.json`, `community.chocolatey.org`, `raw.githubusercontent.com/ScoopInstaller/Main/master/bucket/kilo.json` | TUI worker calls `upgrade()` on launch unless `autoupdate: false` or `KILO_DISABLE_AUTOUPDATE=1`; auto-upgrades patch versions on npm/yarn/pnpm/bun. | `packages/opencode/src/cli/cmd/tui.ts:454`; `packages/opencode/src/cli/tui/worker.ts:106-108`; `packages/opencode/src/cli/upgrade.ts:8-54`; `packages/opencode/src/installation/index.ts:248-306`; `packages/opencode/src/kilocode/installation/index.ts:1-25`; `latest.ts:8-15` |
| **JetBrains CLI Downloader / Curl Installer** | `https://api.github.com/repos/Kilo-Org/kilocode/releases/tags`, `https://github.com/Kilo-Org/kilocode/releases/download/...`, `https://kilo.ai/cli/install`, `https://registry.npmjs.org/-/package/%40kilocode%2Fcli/dist-tags` | JetBrains fetches pinned CLI from GitHub Releases if not already cached/bundled; curl installer queries npm dist-tags & GitHub Releases. | `KiloCliDownloader.kt:36-37,64-80`; `install:24,187-200` |

---

## 5. Exact Data & Properties Collected

### A. PostHog Event Envelope & Identity Properties
Every PostHog event dispatched through `Telemetry.track` (`packages/kilo-telemetry/src/telemetry.ts:110-135,193-213`) and `Client.capture` (`packages/kilo-telemetry/src/client.ts:62-76`) includes:
- `distinct_id`: raw plaintext Kilo `email` when signed in (`identity.ts:55-57,131`), otherwise `telemetry-id` / `KILO_MACHINE_ID`.
- Base device/runtime properties: `appName` (`"kilo-cli"` or `"kilo-code"`), `appVersion`, `platform`, `os_name`, `os_version` (`os.release()`), `os_arch`, optional `editorName`, optional `vscodeVersion`, optional `aauClientToken` (Conda AAU ID), and optional `kilocodeOrganizationId`.
- `$set` person properties on `CLI Start` and `Auth Success` (`telemetry.ts:200-211`) + `$create_alias` tying `telemetry-id` to `email` (`telemetry.ts:186-190`).
- JetBrains adds `client: "jetbrains"`, `feature: "jetbrains-plugin"`, `jetbrainsBuild`, `pluginVersion` via `KiloLog.payload` (`packages/kilo-jetbrains/shared/src/main/kotlin/ai/kilocode/log/KiloLog.kt:76-92`; `KiloBackendTelemetry.kt:75-83`).

### B. IP Address & GeoIP Handling
- **Explicitly enabled in PostHog:** `new PostHog(POSTHOG_API_KEY, { host: POSTHOG_HOST, disableGeoip: false })` (`packages/kilo-telemetry/src/client.ts:20-23`). PostHog resolves and stores client geolocation/IP metadata server-side.
- **No IP stripping on Gateway/KiloSessions/OTLP:** Standard HTTPS requests carry source IP; architecture docs (`packages/kilo-docs/pages/contributing/architecture/cloud-security.md:295,326`) note operational telemetry and free-model abuse controls retain customer-linked/IP records on Kilo Cloud.

### C. Custom / Private Model Names & Provider Fields
- **Verbatim custom/private `modelId` and `providerID` ARE transmitted** across multiple PostHog events (even for local/Ollama/custom OpenAI-compatible/private enterprise models):
  - `LLM Completion`: verbatim `apiProvider: input.model.providerID`, `modelId: input.model.id`, `taskId: input.sessionID`, token counts, `cost`, `completionTime` (`packages/opencode/src/kilocode/session/processor.ts:140-152`).
  - `Feedback Submitted`: verbatim `providerID`, `modelID`, `variant`, `rating` across all providers; adds `sessionID`, `messageID`, `parentMessageID` for `providerID === "kilo"` (`packages/opencode/src/kilocode/cli/cmd/tui/feedback.ts:40-52`; `packages/kilo-vscode/webview-ui/src/context/feedback-payload.ts:27-40`).
  - `Indexing Started` / `Completed` / `File Count` / `Batch Retry` / `Error`: verbatim `provider`, `vectorStore`, `modelId`, counts, sanitized `error` (`packages/opencode/src/kilocode/indexing.ts:140-209`).
  - JetBrains `Conversation Send Clicked`, `Conversation Message`, `Model Selected`, `Context Condensed`, `Session Error`: verbatim `model`, `provider`, `modelId`, `agent`, `sessionId`, `textLength` bucket (`SessionController.kt:431-449,3297-3330`).
- **Gateway Auto-Routing:** Sends `model` ID, candidate `clientEntries: [{ provider, model }]`, `machineId`, `sessionId`, `mode`, **plus full OpenAI `messages` and `tools` payloads** to `https://auto-routing.kiloapps.io` (`packages/opencode/src/kilocode/auto-routing/router.ts:132-145`).

### D. Prompts, Code, Paths, Error Strings & Behavioral Telemetry
- **PostHog (`Telemetry`):**
  - Does **not** send raw prompt text or workspace file paths in core CLI telemetry events: JetBrains buckets prompt length into `"empty"|"short"|"medium"|"long"` (`SessionController.kt:3334-3339`) and buckets error descriptions (`SessionController.kt:3349-3356`); MCP auth explicitly strips server names/URLs/OAuth secrets unless matching a public marketplace item (`packages/opencode/src/kilocode/mcp/auth-telemetry.ts:8-9`); Marketplace events strip parameter values/paths (`packages/opencode/src/kilocode/marketplace/telemetry.ts:7`); Indexing sanitizes `error` strings via `sanitizeErrorMessage()` replacing URLs, emails, IPv4s, Unix/Windows paths, and ports (`packages/kilo-indexing/src/indexing/shared/validation-helpers.ts:10-45`), and passes static literal `location` tags (`manager.ts:215,350`, `orchestrator.ts:128,257`, `file-watcher.ts:511,662,683`, `scanner.ts:670`).
  - **Caveat:** `Telemetry.trackError(error, context)` (`telemetry.ts:386-388`) and `TelemetryProxy.capture(AGENT_MANAGER_SESSION_ERROR, { error, context, sessionId })` (`packages/kilo-vscode/src/agent-manager/fork-session.ts:54-59`) and `Telemetry.trackMcpServerError(server, error)` (`telemetry.ts:350-352`) forward raw error/server strings if invoked.
  - Behavioral profile events include `CLI Start`/`Exit`, `LLM Completion`, `Plan Followup` (`choice`), `Suggestion Shown`/`Accepted`, `Tool Used` (`chart` in `tool/chart.ts:70`), `Auth Success`/`Logout`, `Remote Connection Opened` (`kilo-sessions.ts:1362`), `Feedback Submitted`, VS Code `TITLE_BUTTON_CLICKED` (`extension.ts:627-654`), `WORK_STYLE_ONBOARDING_SHOWN` / `WORK_STYLE_SELECTED` (`work-style.tsx:77-99`), `GHOST_SERVICE_DISABLED` (`AutocompleteServiceManager.ts:171`), `AUTOCOMPLETE_UNIQUE_SUGGESTION_SHOWN` (`AutocompleteTelemetry.ts:206`), and JetBrains UI actions.
- **`KiloSessions` Cloud Ingest (`https://ingest.kilosessions.ai`):**
  - Captures **full session contents** on all Kilo-authenticated sessions (across any provider): `kilo_meta` (`platform`, `orgId`, normalized `gitUrl`, `gitBranch`), `session` info (`directory`, `title`), every `message` and `part` (**full user prompts, assistant text, reasoning, tool invocations, tool outputs/file reads**), cumulative `session_diff` (code diffs), `model` metadata, `session_status`, `session_open`/`session_close`, and `session_pr_link` (`packages/opencode/src/kilo-sessions/ingest-queue.ts:13-62`; `packages/opencode/src/kilo-sessions/kilo-sessions.ts:808-953,1878-1929,2001-2019`).
  - Also sends `X-KILOCODE-PROJECTID` (from `.kilo/config.json`, `.kilocode/config.json`, or `git remote.origin.url` repo name) on Kilo Gateway LLM requests (`packages/opencode/src/kilocode/project-id.ts:45-93`; `request.ts:184-186,260`).

---

## 6. Docs / Privacy Policy vs Code Discrepancies

| Document Claim | Verified Source Code Reality | Impact |
|---|---|---|
| **`PRIVACY.md:9-12` ("Last Updated: March 7th, 2025"):** Claims for Code & Files, Commands, and Prompts & AI Requests: *"We do not have access to this data"* / *"We do not have access to or store this data"*, and omits all mention of product telemetry, device IDs, or session sync. | **Contradicted on both counts.** `PostHog` telemetry is ON by default (`client.ts:8-23`), assigns persistent `telemetry-id` linked to plaintext `email` (`identity.ts:42-57,130-132`), reads `~/.conda/aau_token` (`conda.ts:9`), and `KiloSessions` automatically syncs full prompts, messages, tool outputs, code diffs, `gitUrl`, and `gitBranch` to `https://ingest.kilosessions.ai` on every Kilo-authenticated session (`kilo-sessions.ts:289,776-953,1878-1929`). | **Severe / Material** |
| **`packages/kilo-docs/pages/getting-started/settings/product-telemetry.md:73`:** Claims *"KILO_TELEMETRY_LEVEL takes precedence: `off` disables product telemetry and `all` enables it even when global config says `false`."* | `Telemetry.level` computes `configured && allowed && ...` (`packages/kilo-telemetry/src/telemetry.ts:150`). `experimental.openTelemetry: false` **blocks** `KILO_TELEMETRY_LEVEL=all`, as explicitly verified by unit test `"a config opt-out wins over a client that sets KILO_TELEMETRY_LEVEL=all"` (`packages/kilo-telemetry/src/__tests__/identity-delivery.test.ts:83-86`). | **High** |
| **`product-telemetry.md:17,34,40`:** Claims VS Code has no Kilo-only switch and *"Setting only `experimental.openTelemetry: false` in Kilo config or inheriting `KILO_TELEMETRY_LEVEL=off` does not override an enabled VS Code telemetry preference."* | VS Code **does** provide a Kilo-only **"Enable telemetry"** switch (`AboutKiloCodeTab.tsx:268-289` -> `POST /telemetry/level`), and `Telemetry.level` (`telemetry.ts:150`) honors **both** `experimental.openTelemetry: false` and `KILO_TELEMETRY_LEVEL=off` even when `vscode.env.isTelemetryEnabled` is `true`. | **High** |
| **`product-telemetry.md:19,88-92`:** Claims *"Product telemetry is currently always on in the JetBrains plugin. The CLI settings above and the IDE's data-sharing preferences do not turn it off."* | JetBrains **now implements** a **"Send usage data"** opt-out toggle in **Settings > Tools > Kilo Code > Advanced** (`AdvancedConfigurable.kt`; `AdvancedSettingsUi.kt:39,50-54`; `KiloBundle.properties:763-768`), wired to `POST /telemetry/level`, and respects `experimental.openTelemetry: false` and `KILO_TELEMETRY_LEVEL=off`. (It still ignores IntelliJ's built-in data-sharing preference.) | **Medium (Stale Docs)** |
| **`product-telemetry.md:10,18,77-86,94-96`:** Documents **Kilo Desktop** and **Mobile** settings/crash reporting. | Neither Kilo Desktop nor Mobile source code exists in `Kilo-Org/kilocode`; only CLI/TUI/HTTP server, VS Code, JetBrains, and docs reside in this repository (`packages/*`). | **Informational / Scope Gap** |
| **`product-telemetry.md:98-103` & `cli.md:491-519`:** Fail to disclose `KiloSessions` (`ingest.kilosessions.ai` / `KILO_DISABLE_SESSION_INGEST`), `~/.conda/aau_token` join-key extraction, plaintext email `distinct_id` + `$create_alias`, unconditional `telemetry-id` creation, or `X-KILOCODE-MACHINEID` / `X-KILOCODE-PROJECTID` headers. | `KILO_SESSION_INGEST_URL` appears only in contributor docs (`CONTRIBUTING.md:266`). User-facing docs leave users unaware that opting out of product telemetry does not stop `ingest.kilosessions.ai` transcript/diff uploads or `X-KILOCODE-MACHINEID` transmission. | **Severe** |

---

## 7. Zero-Telemetry / Negative-Check Verification

Read-only `rg` checks across `packages/`, `script/`, `install`, and `package.json` verified the following **clean negatives**:
- **Crash/APM SDKs absent in shipping runtime code:** Case-insensitive search for `\b(sentry|mixpanel|amplitude|bugsnag|rollbar|datadog|honeycomb|fullstory|logrocket)\b|@sentry/|segment\.io|@segment/|appinsights|applicationinsights` returned **zero** runtime analytics/crash SDK imports or DSNs in `packages/kilo-telemetry`, `packages/opencode`, `packages/core`, `packages/kilo-vscode`, `packages/kilo-jetbrains`, `packages/kilo-console`, `packages/kilo-indexing`, or `packages/tui`.
  - The only matches were upstream Sentry removal transforms (`script/upstream/transforms/transform-package-json.ts:330-333`), a Sentry icon asset (`packages/ui/src/components/file-icons/types.ts:144`), marketplace/docs mentions of Sentry as a cloud/MCP integration (`packages/kilo-docs/pages/contributing/architecture/cloud-security.md:320` referring to separate `Kilo-Org/cloud` infrastructure), and `DO_NOT_TRACK` in `packages/opencode/test/kilocode/startup-speed.test.ts:52` / `product-telemetry.md:73`.
- **No silent external AI/cloud auth harvesting:** `packages/opencode/src/plugin/` contains zero reads of `~/.aws`, `~/.ssh`, `~/.config/gh`, `~/.gemini`, `~/.codex`, or `~/.claude`; Claude/Codex transcript imports and migration require explicit user flags/commands (`experimental.claudeMigration` defaults `false`; `kilo import` / `/resume-claude` / `/resume-codex`).
- **AI SDK telemetry disabled:** `experimental_telemetry` is explicitly hardcoded `{ isEnabled: false }` in both `packages/opencode/src/session/llm.ts:482` and `packages/opencode/src/kilocode/agent/index.ts:493-495`, so no `ai.*` / `gen_ai.*` LLM prompt/response spans are recorded by default.

---

```json
{
  "slug": "kilocode",
  "platform": "Kilo Code",
  "group": "B",
  "repo": "https://github.com/Kilo-Org/kilocode",
  "sha": "b12e612bd3ad9c4073ea57caaf0197fad7f98d63",
  "commit_date": "2026-11-24T14:57:22-05:00",
  "license": "MIT",
  "status": "Active",
  "default_state": "ON by default: PostHog product telemetry enabled via experimental.openTelemetry=true (packages/core/src/v1/config/config.ts:352-354) and KiloSessions cloud transcript/diff ingest enabled whenever Kilo auth exists (packages/opencode/src/kilo-sessions/kilo-sessions.ts:281-315,776-953). Optional OTLP and SessionExport (enabled=false) are OFF by default.",
  "opt_out_controls": [
    "Global config experimental.openTelemetry: false (packages/core/src/v1/config/config.ts:352-354)",
    "Env var KILO_TELEMETRY_LEVEL=off|0|false (any value other than all|on|true|1; packages/kilo-telemetry/src/telemetry.ts:147-151)",
    "VS Code 'telemetry.telemetryLevel' != 'all' (KILO_TELEMETRY_HOST_DISABLED=1) and Kilo Settings > About Kilo Code 'Enable telemetry' switch (packages/kilo-vscode/webview-ui/src/components/settings/AboutKiloCodeTab.tsx:268-289)",
    "JetBrains Settings > Tools > Kilo Code > Advanced 'Send usage data' toggle (packages/kilo-jetbrains/frontend/src/main/kotlin/ai/kilocode/client/settings/AdvancedSettingsUi.kt:50-54)",
    "Env var KILO_DISABLE_SESSION_INGEST=1 (undocumented for users; disables https://ingest.kilosessions.ai auto-sync)",
    "Env vars KILO_DISABLE_AUTOUPDATE=1 / autoupdate=false and KILO_DISABLE_MODELS_FETCH=1"
  ],
  "do_not_track": "Not supported / ignored in runtime code (documented as unsupported in packages/kilo-docs/pages/getting-started/settings/product-telemetry.md:73).",
  "opt_out_bypasses": [
    "Identity.getMachineId() unconditionally generates and persists UUID v4 telemetry-id even when telemetry is disabled (packages/kilo-telemetry/src/telemetry.ts:141); VS Code and JetBrains pass machine IDs unconditionally.",
    "Persistent machineId is sent via X-KILOCODE-MACHINEID on Kilo Gateway LLM requests (packages/opencode/src/session/llm/request.ts:183-189,261) and to https://auto-routing.kiloapps.io (packages/opencode/src/session/llm.ts:117,126) regardless of telemetry opt-out.",
    "KiloSessions automatically uploads full prompts, replies, tool outputs, diffs, gitUrl, gitBranch, and models to https://ingest.kilosessions.ai on Kilo-authenticated sessions regardless of experimental.openTelemetry=false, KILO_TELEMETRY_LEVEL=off, KILO_DISABLE_SHARE=1, or non-Kilo model provider (packages/opencode/src/kilo-sessions/kilo-sessions.ts:281-315,776-953,1878-1929).",
    "VS Code TelemetryProxy.capture and JetBrains KiloTelemetryService still forward UI events to localhost CLI /telemetry/capture when only Kilo's experimental.openTelemetry=false is set (dropped server-side at Client.capture)."
  ],
  "persistent_ids": [
    "telemetry-id (UUID v4 in <Global.Path.data>/telemetry-id, or KILO_MACHINE_ID from vscode.env.machineId / JetBrains <system>/kilo/machine-id)",
    "Plaintext Kilo account email used as PostHog distinct_id via /api/profile and cached in telemetry-profile.json",
    "kilocodeOrganizationId",
    "External Conda aauClientToken extracted from ~/.conda/aau_token + 'conda info --json --offline' (when telemetry enabled)"
  ],
  "hashing": "None for telemetry-id, email distinct_id, or Conda aauClientToken (sent plaintext to PostHog; $create_alias links telemetry-id to plaintext email). SHA-256 used only locally for telemetry-profile.json token cache key and telemetry-alias-<sha256> dedup marker filenames.",
  "external_auth_reads": [
    "Telemetry join key: ~/.conda/aau_token + 'conda info --json --offline' when telemetry enabled (packages/kilo-telemetry/src/conda.ts:8-49)",
    "Legacy Kilo migration: ~/.kilocode/cli/config.json (packages/kilo-gateway/src/auth/legacy-migration.ts:11), JetBrains ~/.kilocode/secrets.json & globalStorage/ (LegacyV5Sources.kt:13-17), user-triggered Roo import (packages/kilo-vscode/src/roo-import/service.ts:12-58)",
    "Compat/Opt-in: ~/.claude/CLAUDE.md & ~/.claude/skills/ (CLI default on unless KILO_DISABLE_CLAUDE_CODE=1; VS Code/JetBrains default off), ~/.claude/ide/*.lock (TUI editor lock), opt-in ClaudeMigration (~/.claude.json), explicit /resume-claude & /resume-codex (~/.claude/projects/, ~/.codex/sessions/)"
  ],
  "telemetry_endpoints": [
    "https://us.i.posthog.com (posthog-node product telemetry; us.i.posthog.com / us-assets.i.posthog.com via /ingest on kilo-docs)",
    "https://ingest.kilosessions.ai (default-active KiloSessions transcript/diff ingest & WebSocket relay for Kilo-authenticated sessions)",
    "https://api.kilo.ai (/api/profile, /api/user, /api/users/notifications)",
    "https://ai-gateway.kilo.ai/api/v1 & https://auto-routing.kiloapps.io (carry X-KILOCODE-MACHINEID / machineId even when telemetry off)",
    "wss://events.kiloapps.io (Kilo Event Service WebSocket)",
    "https://supermassive-black-hole.kiloapps.io/v1/session-export/batch (hard-disabled enabled=false in source)",
    "User-configured OTEL_EXPORTER_OTLP_ENDPOINT (/v1/logs, /v1/traces; opt-in only)",
    "https://models.dev/api.json, https://registry.npmjs.org/@kilocode%2fcli/<channel>, GitHub Releases / Brew / Chocolatey / Scoop update/download URLs"
  ],
  "ip_geo_handling": "PostHog client explicitly configures disableGeoip: false (packages/kilo-telemetry/src/client.ts:22), enabling server-side IP/GeoIP resolution on all PostHog events. No IP suppression/anonymization on direct HTTPS requests to Kilo endpoints.",
  "custom_model_handling": "Verbatim custom/private modelId and providerID/apiProvider strings transmitted in PostHog LLM Completion (processor.ts:143-144), Feedback Submitted (feedback.ts:41-42; feedback-payload.ts:29-30), Indexing events (indexing.ts:146-203), and JetBrains session/model events (SessionController.kt:3307-3313), linked to plaintext email/telemetry-id/aauClientToken. Auto-routing sends client model list + prompts/tools to auto-routing.kiloapps.io.",
  "docs_discrepancies": [
    "PRIVACY.md (2025-03-07) states Kilo does not have access to or store code, files, prompts, or AI requests, omitting PostHog telemetry, plaintext email identification, telemetry-id, ~/.conda/aau_token, and automatic KiloSessions uploads to ingest.kilosessions.ai.",
    "product-telemetry.md:73 wrongly states KILO_TELEMETRY_LEVEL=all overrides experimental.openTelemetry=false (Telemetry.level uses && so global config false wins).",
    "product-telemetry.md:17,34,40 wrongly state VS Code has no Kilo toggle and ignores experimental.openTelemetry=false / KILO_TELEMETRY_LEVEL=off (VS Code ships About Kilo Code 'Enable telemetry' switch and honors both).",
    "product-telemetry.md:19,88-92 wrongly state JetBrains telemetry is 'currently always on' (JetBrains ships Settings > Tools > Kilo Code > Advanced 'Send usage data' toggle and honors CLI config/env).",
    "User docs omit KILO_DISABLE_SESSION_INGEST / ingest.kilosessions.ai auto-sync, ~/.conda/aau_token reading, plaintext email aliasing, and unconditional machine-ID transmission."
  ],
  "privacy_rating": "F",
  "one_line_verdict": "Severe red flags: PostHog telemetry with GeoIP, raw plaintext email aliasing, and ~/.conda/aau_token harvesting is ON by default (ignoring DO_NOT_TRACK), persistent machine IDs transmit on Gateway/auto-router calls even when opted out, Kilo-authenticated sessions automatically upload full prompts, tool outputs, code diffs, gitUrl, and gitBranch to ingest.kilosessions.ai bypassing experimental.openTelemetry=false, KILO_TELEMETRY_LEVEL=off, and KILO_DISABLE_SHARE=1, and root PRIVACY.md directly contradicts the code by claiming Kilo has no access to or storage of code, commands, or prompts."
}
```
