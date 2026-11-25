# Privacy & Telemetry Forensic Audit: OpenCode (`anomalyco/opencode`)

## Header Metadata

- **Slug:** `opencode`
- **Platform:** `OpenCode`
- **Group:** `B`
- **Repository:** `anomalyco/opencode` (`https://github.com/anomalyco/opencode`; `sst/opencode` redirects here)
- **License:** `MIT`
- **Status:** `Active (sst/opencode redirects here)`
- **Branch:** `dev`
- **Commit SHA:** `4c2977b82ece65dfebc4c25058f76e4f8a241c61`
- **Commit Date:** `2026-11-24T23:52:07-05:00`
- **Commit Subject:** `feat(core): add push notifications for the mobile apps (#66130)`

> **Monorepo Architecture Note (v2 Layout at Audited Commit):** At this `dev` SHA, OpenCode has migrated to its v2 monorepo layout. Legacy directories (`packages/opencode`, `packages/console`, `packages/web`, `packages/mobile`, `packages/slack`) are no longer present. Active components audited below are:
> - **Core CLI, Daemon/Server & Rust CLI:** `packages/cli`, `packages/cli-rs`, `packages/core`, `packages/server`, `packages/util`, `packages/schema`
> - **Desktop Electron Shell & Web/UI App:** `packages/desktop`, `packages/app`, `packages/gui-plugins`, `packages/session-ui`, `packages/tui`
> - **Hosted Services & Docs:** `services/update`, `services/site`, `services/legacy`, `services/data`, `.github/workflows/`

---

## 1. Default State & Opt-Out Mechanism

OpenCode's default outbound telemetry behavior splits sharply between its **Core CLI / Local Server** (minimal default network metadata + strictly opt-in OpenTelemetry), its **Desktop Electron / Web App** (release-baked Sentry crash reporting with a broken prod opt-out filter, un-disableable 10-minute auto-update polling, and automatic Google Favicon fetches), and its **Hosted Website / Update Backend** (`services/update` IP/geo data lake logging and `opencode.ai` first-party funnel analytics).

### A. Core CLI & Local Server (`packages/cli`, `packages/cli-rs`, `packages/core`, `packages/server`)

1. **Product Analytics SDKs: None (Clean).**
   No PostHog, Mixpanel, Amplitude, Segment, Statsig, or Bugsnag SDK exists anywhere in the client or server runtime. (The only `posthog` references are MCP Code Mode header handling when a *user configures* an external PostHog MCP server at `mcp*.posthog.com`, `packages/core/src/plugin/mcp-codemode-defaults.ts:12-33`.)
2. **OpenTelemetry (OTLP): Strictly Opt-In (OFF by default).**
   - In the TypeScript CLI/server (`packages/cli/src/index.ts:164-170`, `packages/util/src/observability.ts:20-52`), OTLP logs (`/v1/logs`) and traces (`/v1/traces`) are exported **only** when `OTEL_EXPORTER_OTLP_ENDPOINT` is set (`packages/util/src/observability/otlp.ts:63-75,96-99`). When unset, `Otlp.loggers()` returns `[]` and `Otlp.tracingLayer()` returns `Layer.empty`.
   - In the Rust native CLI (`packages/cli-rs/crates/cli/src/telemetry.rs:53-54`), OTLP export (`/v1/logs` only, no traces per `packages/cli-rs/parity/deviations/G11-telemetry.md:3-11`) is likewise disabled unless `OTEL_EXPORTER_OTLP_ENDPOINT` is non-empty.
   - Legacy v1 `experimental.openTelemetry` in `opencode.json` was removed in v2 and is stripped as an unsupported key (`packages/core/src/config/normalize.ts:46-52`).
   - Local file logging (`opencode.log` / `opencode-<channel>.log`, rotated at `50 MB` to `25 MB`) remains strictly local (`packages/util/src/observability/logging.ts:9-60`).
3. **Model Catalog Polling (`models.opencode.ai`): ON by default, Opt-Out via Env.**
   - By default (`fetch: !truthy(process.env.OPENCODE_DISABLE_MODELS_FETCH)`, `packages/cli/src/server-process.ts:117-121`), the server polls `https://models.opencode.ai/experimental/v2.0/api.json?type=chat,decision,image,video,realtime&include=openai-codex,openai-chatgpt` immediately on startup and every `5 minutes` (`packages/core/src/models-dev.ts:680-683,733-738,880-883`).
   - **Opt-Out:** `OPENCODE_DISABLE_MODELS_FETCH=1` (or `true`) stops remote fetches and falls back to the bundled snapshot (`packages/core/src/models-dev.ts:696-705,880`); `OPENCODE_MODELS_URL` or `OPENCODE_MODELS_PATH` overrides the source (`packages/cli/src/server-process.ts:118-119`).
4. **CLI Auto-Update Checks (`opencode.ai/update/api`): ON by default, Opt-Out via Env/Config.**
   - By default, the CLI checks `https://opencode.ai/update/api/${channel}/${OPENCODE_ARTIFACT}/${distribution}?current=${OPENCODE_VERSION}` (or `https://formulae.brew.sh/api/formula/opencode.json` for Homebrew Core) immediately and every `10 minutes` (`packages/cli/src/services/updater.ts:315-328`, `packages/cli/src/commands/handlers/default.ts:83-100`).
   - **Opt-Out:** Setting `OPENCODE_DISABLE_AUTOUPDATE=1` (or `true`), setting `config.update = "disable"`, or running a local dev build (`OPENCODE_LOCAL`) skips update checks (`packages/cli/src/services/updater.ts:607-620`).

### B. Desktop Electron & Web UI (`packages/desktop`, `packages/app`, `packages/gui-plugins`, `packages/session-ui`)

1. **Sentry Error & Crash Reporting (`@sentry/solid`): ON in Release Builds, NO Runtime Opt-Out, Broken Prod Filter Bypass.**
   - Desktop releases inject `VITE_SENTRY_DSN: ${{ vars.WEB_SENTRY_DSN }}`, `VITE_SENTRY_ENVIRONMENT`, and `VITE_SENTRY_RELEASE` at build time (`.github/workflows/release.yml:354-361`).
   - On renderer startup (`packages/desktop/src/renderer/index.tsx:23` -> `packages/desktop/src/renderer/startup/sentry.ts:7-29`), `@sentry/solid` `init()` executes unconditionally whenever `VITE_SENTRY_DSN` is truthy. There is **no UI toggle in Settings (`packages/app/src/settings/general/general.tsx`, `packages/gui-plugins/src/updater/section.tsx`), no `opencode.json` setting, no CLI/env flag, and no `DO_NOT_TRACK` check** to disable Sentry.
   - **Critical Prod Opt-Out Bypass Bug:** Comments in `packages/app/src/app.tsx:62-64` claim `"Prod crashes reach Sentry only when someone clicks Report; other channels send them as they happen"`, gating React `ErrorBoundary` automatic capture on `VITE_OPENCODE_CHANNEL`. However, both `packages/desktop/src/renderer/startup/sentry.ts:19-27` and `packages/app/src/entry.tsx:71-76` attempt to strip Sentry's automatic `GlobalHandlers` and `BrowserApiErrors` integrations using:
     ```ts
     import.meta.env.OPENCODE_CHANNEL === "prod"
     ```
     In Vite configs (`packages/desktop/electron.vite.config.ts:47,110` and `packages/app/vite.js:113`), `import.meta.env.OPENCODE_CHANNEL` is only defined in Electron's **main** process; renderer and web builds only define `import.meta.env.VITE_OPENCODE_CHANNEL`. Consequently, `import.meta.env.OPENCODE_CHANNEL === "prod"` evaluates to `undefined === "prod"` (`false`) in production builds, silently keeping automatic `GlobalHandlers` and `BrowserApiErrors` crash transmission enabled in prod!
   - By contrast, Electron's native Crashpad crash reporter starts with `uploadToServer: false` (local dumps only, `packages/desktop/src/main/native/logging.ts:261`), and `AppRecordFatalRendererError` writes only to local logs (`packages/desktop/src/main/ipc-handlers/app.ts:49-50`). Build-time sourcemap uploads via `sentryVitePlugin` explicitly set `telemetry: false` (`packages/app/vite.config.ts:10-21`).
2. **Desktop Auto-Update Polling (`electron-updater`): ON by default, NO Runtime Opt-Out.**
   - In packaged Desktop builds that contain `app-update.yml`, the updater starts unconditionally (`packages/gui-plugins/src/updater/main.ts:17`), sets `User-Agent: opencode/${channel === "prod" ? "latest" : channel}/${app.getVersion()}/desktop` (`packages/gui-plugins/src/updater/platform.ts:17,26`), and polls `https://opencode.ai/update/api/{dev,beta,latest}/desktop/opencode/` (`packages/desktop/electron-builder.config.ts:248,267,291`) immediately on launch and every `10 minutes` (`packages/gui-plugins/src/updater/machine.ts:207-211`).
   - Unlike the CLI updater, Desktop's updater reads **neither `OPENCODE_DISABLE_AUTOUPDATE` nor `config.update = "disable"`**, and `UpdatesSection` exposes only a "Release notes" switch and manual "Check for updates" button — **no toggle to disable automatic update polling** (`packages/gui-plugins/src/updater/section.tsx:23-64`, `packages/gui-plugins/src/updater/main.ts:11-36`). Only distro packages that strip `app-update.yml` disable it (`packages/gui-plugins/src/updater/main.ts:15-17`).
3. **Other Automatic / UI-Triggered Desktop & Web Network Calls:**
   - **What's New Changelog (`opencode.ai/changelog.json`):** On version upgrades, fetches `https://opencode.ai/changelog.json` once (`packages/gui-plugins/src/updater/whats-new.tsx:8,55-68`, `packages/gui-plugins/src/updater/renderer.tsx:130-182`). Has an opt-out UI switch (`releaseNotes.enabled` in `packages/gui-plugins/src/updater/section.tsx:25-40`).
   - **Markdown Link Favicons (`google.com/s2/favicons`):** Any rendered public `http(s)` link in markdown chat messages (excluding `github.com`, IP addresses, and private suffixes) automatically injects an `<img>` pointing to `https://www.google.com/s2/favicons?domain=${encodeURIComponent(url.host)}&sz=32` (`packages/session-ui/src/components/markdown.tsx:640-669`) with **no opt-out**.
   - **Desktop Notification Icon (`opencode.ai`):** macOS/Linux notifications load `icon: "https://opencode.ai/favicon-96x96-v3.png"` (`packages/desktop/src/renderer/platform/notifications.ts:49`, `packages/app/src/runtime/platform/web.ts:32`).
   - **About Page GitHub API (`api.github.com`):** Opening Settings -> About fetches `https://api.github.com/repos/anomalyco/opencode/contributors?anon=1&per_page=1` (`packages/app/src/settings/about/contributors.ts:15-24`).

### C. `DO_NOT_TRACK` & Hosted Website (`services/site`)

- **CLI / Desktop / Web App:** Zero support for `DO_NOT_TRACK` (`rg` returned zero matches across `packages/`).
- **Hosted Website (`opencode.ai`, `services/site`):** Loads a first-party tracker `/console/funnel/v1.js` on marketing and docs pages after first pointer/key input or 1s idle (`services/site/src/lib/funnel.ts:44-67`, `funnel-config.ts:33-58`). The Privacy Policy explicitly states:
  - **DNT Ignored:** `"the Services do not support 'Do Not Track' requests sent from a browser at this time"` (`services/site/src/lib/legal/privacy-policy.ts:476`).
  - **Partial Opt-Out / GPC Only:** When `navigator.globalPrivacyControl === true` or the user sets `oc_optout=1` (`services/site/src/component/site/analytics-choice.ts:38-57`), OpenCode strips `oc_vid`, `oc_sid`, `oc_attr`, and `opencode-console.funnel.seen`, **but explicitly continues recording page views, clicks, and account/workspace events without linking them to browser IDs** (`services/site/src/lib/legal/privacy-policy.ts:583-586`).

---

## 2. External Auth / Local Non-Workspace File Reads

OpenCode does **not** harvest or exfiltrate credentials from external AI coding tools, nor does it transmit any discovered local file contents over telemetry. However, for interoperability and Desktop's embedded browser feature, it reads several non-workspace paths on the user's machine:

| Local Path / Source | Purpose & Scope | Opt-Out / Condition | Telemetry Transmission | Source Citation |
| :--- | :--- | :--- | :--- | :--- |
| `~/.local/share/opencode/auth.json` | One-time migration of OpenCode v1 credentials into v2 SQLite DB on migration | Runs once when migrating/initializing DB | None (local DB only) | `packages/core/src/database/migration/20260805200742_import_legacy_credentials.ts:39,46-80` |
| `~/.claude/CLAUDE.md` | Fallback global instruction file when `~/.config/opencode/AGENTS.md` is missing | Opt-out via `OPENCODE_DISABLE_CLAUDE_CODE=1`, `OPENCODE_DISABLE_CLAUDE_CODE_PROMPT=1`, or `global: false` | Sent only to user's LLM provider as prompt context | `packages/core/src/config/plugin/instruction.ts:42-45,82-90`, `packages/cli/src/server-process.ts:135-140` |
| `~/.claude/skills` & ancestor `.claude/skills` | Interoperability skill discovery alongside `.agents/skills` | Opt-out via `OPENCODE_DISABLE_CLAUDE_CODE=1` or `OPENCODE_DISABLE_CLAUDE_CODE_SKILLS=1` | None | `packages/core/src/config/discovery.ts:81,163-171`, `packages/core/src/config/plugin/compatibility.ts:45-57`, `packages/cli/src/server-process.ts:128-131` |
| `~/.claude/ide/*.lock` | Active IDE WebSocket discovery (`port`, `authToken`, `workspaceFolders`) matched to CWD | Only when workspace folder matches CWD | None (local IPC only) | `packages/tui/src/editor.ts:41-81` |
| Zed SQLite DB (`~/Library/Application Support/Zed/db/0-stable/db.sqlite`, `~/.local/share/zed/db/0-stable/db.sqlite`) | Active editor buffer/selection context in TUI (`OPENCODE_ZED_DB` override) | Only when workspace path matches CWD | None (prompt context if attached) | `packages/tui/src/editor-zed.ts:187-195` |
| `claude` binary / `CLAUDE_CONFIG_DIR` | Claude Agent provider spawns local `claude` CLI / SDK & checks account/transcripts | Opt-in: only when user connects/uses `Claude Agent` provider | None (direct Anthropic/Claude Code session) | `packages/core/src/plugin/provider/claude-agent/claude/io.ts:22-51`, `probe.ts:44-70`, `query-options.ts:49-51` |
| `~/.aws/config`, `~/.aws/credentials` (or `AWS_CONFIG_FILE` / `AWS_SHARED_CREDENTIALS_FILE`) | Parses INI `[profile ...]` section names **only** (never reads secret keys) | Local Bedrock plugin discovery | None | `packages/core/src/plugin/provider/amazon-bedrock.ts:27-48` |
| `~/.config/gcloud/{active_config,configurations/*,application_default_credentials.json}` | Extracts `project`, `region`, and `quota_project_id` metadata for Vertex AI selector | Local Vertex plugin discovery | None | `packages/core/src/plugin/provider/google-vertex.ts:62-102` |
| `/Library/Managed Preferences/[<user>/]ai.opencode.managed.plist`, `/etc/opencode`, `/Library/Application Support/opencode`, `%ProgramData%\opencode` | Enterprise MDM / system-wide managed config and policy enforcement | Documented managed config (`options.system`) | None | `packages/core/src/config/discovery.ts:141-151`, `packages/core/src/config.ts:256-279`, `services/site/src/docs/content/config.mdx:31-65` |
| Installed Browser Profiles (Chrome, Edge, Brave, Arc, Vivaldi, Opera, Chromium, Firefox, Zen, LibreWolf, Floorp, Safari, Helium) | **Desktop embedded browser import:** lists profiles, cookie host domains, cookies (with OS keychain/Safe Storage decryption if requested), and browsing history | **User-triggered only** via Desktop Browser import UI (`sources`, `sites`, `importBrowser`, `readHistory`); nothing read at startup | Local Electron `persist:opencode-browser` partition (or user-linked remote OpenCode server) | `packages/gui-plugins/src/browser/profile/sources.ts:60-100`, `packages/gui-plugins/src/browser/profile/index.ts:67-96,440-491`, `packages/gui-plugins/src/browser/main.ts:45-83` |
| `~/.codex`, `~/.cursor`, `~/.gemini`, `~/.windsurf`, `~/.copilot`, `claude_desktop_config.json` | **NOT READ** (verified negative across `packages/core`, `packages/cli`, `packages/cli-rs`, `packages/desktop`) | N/A | None | Negative `rg` check (Section 7) |

---

## 3. Persistent Device / Install IDs & Hashing

OpenCode generates **zero persistent device or install telemetry IDs** in its CLI, core server, or Desktop application, though first-party update/web services derive identifiers from IP/cookies:

1. **Telemetry Run ID (`runID()` — Ephemeral per-process):**
   - TS CLI/Server: `crypto.randomUUID().slice(0, 8)` generated lazily once per process (`packages/util/src/observability/shared.ts:5-8`), stamped onto local file logs (`packages/util/src/observability/logging.ts:22`) and, **only when `OTEL_EXPORTER_OTLP_ENDPOINT` is configured**, onto OTLP `opencode.run` and `service.instance.id` (`packages/util/src/observability/otlp.ts:57-58`).
   - Rust CLI: identical per-process 8-char `run_id()` (`packages/cli-rs/crates/cli/src/telemetry.rs:47,381-382`).
2. **Update Rollout Hash (`services/update` — Server-Side IP Hash):**
   - `services/update/src/index.ts:116-123` takes the client's raw IP (`CF-Connecting-IP`) and computes `SHA-256("${resolved}:${ip}")`, taking the first 32 bits (`getUint32(0)`) to assign deterministic progressive rollout buckets across releases (`"The same IP keeps its position across releases"`). Separately, raw `ip` is logged to `env.EVENTS` (`services/update/src/index.ts:68`).
3. **Local / Functional Non-Telemetry Persistent IDs & Hashing:**
   - **Project ID (`packages/core/src/project.ts:317-351,453-458`):** Computed locally via SHA-1 (`Hash.fast`, `packages/util/src/hash.ts:4-6`) of `git-remote:${normalized}` (lowercased host/path), git root commit, or `directory:${directory}` and stored in local SQLite (`ProjectTable`). Not emitted to OpenCode telemetry.
   - **Peer Machine Identity (`packages/core/src/machine/identity.ts:11-62`):** Local 32-byte Ed25519 seed (`machine/home-key` in global data) used solely as the Iroh P2P endpoint identity for user-paired remote OpenCode machines (`packages/core/src/machine/machines.ts`).
   - **Mobile Push Device ID (`packages/core/src/push/list.ts:13-27`, `crypto.ts:14-39`):** Stored locally when a user explicitly pairs a mobile app; encrypted notifications bind `id` as HKDF salt and AES-256-GCM AAD (`OpenCode mobile notifications v2:${id}`).
   - **Hosted Site Web Cookies (`services/site`):** Sets persistent 1-year first-party cookies `oc_vid` (visitor ID), `oc_sid` (session ID), and `oc_attr` (arrival attribution) via `/console/funnel/v1.js` unless GPC or `oc_optout=1` is active (`services/site/src/lib/legal/privacy-policy.ts:570-586`, `analytics-choice.ts:49-54`).
   - **Negative Check:** No hardware UUID, MAC address, `node-machine-id`, `/etc/machine-id`, `IOPlatformUUID`, or `MachineGuid` lookup exists anywhere in the repository.

---

## 4. Exact Outbound Telemetry / Analytics / Crash / OTEL / Relay / Update Endpoints

| Subsystem / Component | Exact Endpoint / URL | Trigger & Frequency | Default State | Opt-Out Control | Source Citation |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Desktop / Web Sentry Crash Reporting** | `VITE_SENTRY_DSN` (`${{ vars.WEB_SENTRY_DSN }}` injected in release CI) | Renderer startup (`init()`), unhandled errors (`GlobalHandlers`/`BrowserApiErrors` even in prod due to `OPENCODE_CHANNEL` bug), manual "Report" button | **ON** in Desktop releases; build-time conditional in Web | **None** at runtime (no UI/config/env/DNT control) | `.github/workflows/release.yml:359`, `packages/desktop/src/renderer/startup/sentry.ts:7-29`, `packages/app/src/entry.tsx:61-78` |
| **Desktop Auto-Updater** | `https://opencode.ai/update/api/{dev,beta,latest}/desktop/opencode/` | Startup + every **10 minutes** in packaged builds with `app-update.yml` | **ON** in packaged Desktop | **None** in UI, env, or `opencode.json` (only deleting `app-update.yml`) | `packages/desktop/electron-builder.config.ts:248,267,291`, `packages/gui-plugins/src/updater/main.ts:17`, `machine.ts:207-211` |
| **CLI Auto-Updater & Installer** | `https://opencode.ai/update/api/{channel}/{artifact}/{distribution}?current={ver}` (or `https://formulae.brew.sh/api/formula/opencode.json`); `https://opencode.ai/v2/install`; `https://registry.npmjs.org/@opencode%2fcli-...` | CLI startup + every **10 minutes**; `opencode upgrade`; `install` script | **ON** in non-local CLI | `OPENCODE_DISABLE_AUTOUPDATE=1|true` or `config.update="disable"` | `packages/cli/src/services/updater.ts:321-324,542,608-620`, `packages/cli/src/commands/handlers/default.ts:83-99`, `install:179,195,212` |
| **Update Worker Data Lake Backend** | Receives requests at `opencode.ai/update/*` | Every CLI / Desktop / `install` update request | **Always ON** server-side | N/A (server ingest) | `services/update/src/index.ts:58-74` |
| **Model Catalog Polling** | `https://models.opencode.ai/experimental/v2.0/api.json?type=chat,decision,image,video,realtime&include=openai-codex,openai-chatgpt` | Server startup + every **5 minutes** | **ON** by default | `OPENCODE_DISABLE_MODELS_FETCH=1|true`, `OPENCODE_MODELS_URL`, `OPENCODE_MODELS_PATH` | `packages/core/src/models-dev.ts:680-683,761-765,880-883`, `packages/cli/src/server-process.ts:117-121` |
| **CLI/Server OTLP (TS & Rust)** | `${OTEL_EXPORTER_OTLP_ENDPOINT}/v1/logs` and `/v1/traces` (TS); `/v1/logs` only (Rust) | Batched logs & Effect/AI SDK spans during operation | **OFF** by default (Opt-In) | Omit/unset `OTEL_EXPORTER_OTLP_ENDPOINT` | `packages/util/src/observability/otlp.ts:63-101`, `packages/cli-rs/crates/cli/src/telemetry.rs:53-54` |
| **Desktop "What's New" Changelog** | `https://opencode.ai/changelog.json` | Renderer startup when version changes | **ON** by default on upgrade | Settings -> General -> Release notes switch (`releaseNotes.enabled`) | `packages/gui-plugins/src/updater/whats-new.tsx:8,55-68`, `renderer.tsx:130-182`, `section.tsx:25-40` |
| **Markdown Link Favicons** | `https://www.google.com/s2/favicons?domain=${host}&sz=32` | Rendering public `http(s)` links in chat markdown (non-GitHub, non-private/IP) | **ON** by default | **None** | `packages/session-ui/src/components/markdown.tsx:640-666` |
| **Desktop Notification Icon** | `https://opencode.ai/favicon-96x96-v3.png` | macOS/Linux/Web desktop notification triggered | **ON** when notifications fire | Disable OS notifications | `packages/desktop/src/renderer/platform/notifications.ts:47-51`, `packages/app/src/runtime/platform/web.ts:32` |
| **Settings About Contributors** | `https://api.github.com/repos/anomalyco/opencode/contributors?anon=1&per_page=1` | Opening Settings -> About tab | Lazy on UI view | Do not open About tab | `packages/app/src/settings/about/contributors.ts:15-24` |
| **Mobile Push Relay** | `https://opencode-notify-relay.anomalyco.workers.dev`, `https://notify.dev.opencode.ai` (or device `callbackURL`) | Session idle, permission prompt, question, clear | **Opt-In** (only if user pairs mobile device) | Unpair device / pause notifications | `packages/core/src/push/list.ts:84`, `packages/core/src/push/sender.ts:84-121` |
| **OpenCode Console / Auth / Inference (User-Connected)** | `https://opencode.ai/console` (`/auth/device/token`, `/api/user`, `/api/orgs`, `/api/v2/skills/...`), `https://opencode.ai/inference/openai/v1`, `https://opencode.ai/oauth/opencode/client.json` | OAuth login / signed-in Console workspace / live voice / remote MCP client metadata | Opt-In (or free tier unless blocked by policy) | Disconnect / `OPENCODE_DISABLE_FREE_MODELS=1` / enterprise deny policy | `packages/core/src/console-access.ts:5-22`, `packages/core/src/plugin/provider/opencode.ts:766,806,829-830`, `voice.ts:167`, `mcp/oauth.ts:36` |
| **Remote Access Tunnel (`@opentunnel/client`)** | User-enabled remote tunnel route (`https://${route}.${tunnel.hostname}`) | `opencode service set remote true` | **OFF** by default | `disabled` / unset remote | `packages/cli/src/services/remote-tunnel.ts:15-73`, `service-config.ts:16-22` |
| **Desktop Browser Extension Stores** | `https://clients2.google.com/service/update2/crx?...`, `https://edge.microsoft.com/extensionwebstorebase/v1/crx?...` | User explicitly installs Chrome/Edge extension into Desktop browser | **Opt-In** (user action) | Do not install store extensions | `packages/gui-plugins/src/browser/profile/extensions.ts:249-256` |
| **Hosted Website Funnel Analytics** | `/console/funnel/v1.js` (served on `opencode.ai` & docs) | Web page visit + pointer/key/1s idle | **ON** on web site | GPC or `oc_optout=1` (strips IDs/cookies only; events still recorded) | `services/site/src/lib/funnel.ts:49-67`, `services/site/src/lib/legal/privacy-policy.ts:583-586` |
| **Legacy API / Retired Share Host** | `api.opencode.ai/exchange_github_app_token` (GitHub Action OIDC); `opncd.ai/api/share*` (discontinued no-op) | GitHub Action run; retired v1 share links | Legacy / CI only (sharing discontinued) | N/A (returns HTTP 410 HTML; zero storage) | `services/legacy/src/api.ts:17-65`, `services/legacy/src/share.ts:9,161-171` |

---

## 5. Exact Data & Properties Collected

1. **IP Address & Geolocation:**
   - **Update Service (`services/update/src/index.ts:58-74`):** Explicitly captures and forwards raw client IP (`request.headers.get("cf-connecting-ip")`), Cloudflare country (`request.cf?.country`), Cloudflare PoP colo (`request.cf?.colo`), `User-Agent`, HTTP `method`, `path`, and ISO `timestamp` to `env.EVENTS.send([...])` (Cloudflare Pipelines data lake) on **every** CLI, Desktop, and installer request—with **zero IP anonymization or suppression**. It also hashes `${resolved}:${ip}` (`SHA-256` -> 32-bit uint) for persistent per-IP rollout bucketing (`services/update/src/index.ts:116-120`).
   - **Sentry (`packages/desktop/src/renderer/startup/sentry.ts:10-28`, `packages/app/src/entry.tsx:62-77`):** `sendDefaultPii` is not enabled in code, and `Breadcrumbs` are filtered out (`sentry.ts:22`, `entry.tsx:74`). Standard Sentry HTTP ingest still sees client IP/User-Agent headers unless stripped server-side by Sentry project settings; payload explicitly carries `environment`, `release` (`desktop@${version}` / `web@${version}`), and `tags.platform` (`"desktop"` / `"web"`).
2. **Custom / Private Model Names & Provider IDs:**
   - **Default State (OTLP OFF):** Not sent in telemetry. `models.opencode.ai` fetches only send `GET` with `User-Agent: opencode/${channel}/${version}/${name}` and optional `If-None-Match` ETag (`packages/core/src/models-dev.ts:761-765`, `packages/core/src/app.ts:22-24`)—no user model list is uploaded.
   - **When OTLP is Enabled (`OTEL_EXPORTER_OTLP_ENDPOINT` set):** `packages/core/src/session/runner/step.ts:83-91,135-136` attaches **verbatim, unhashed** `"gen_ai.provider.name"`, `"gen_ai.request.model"`, `"gen_ai.agent.name"`, and `"opencode.retry.attempt"` to Effect spans (`SessionStep.attempt`, `LLM.stream`), sending custom/private provider & model IDs to the user-configured OTLP collector.
3. **Prompts, Code, File Paths, & Stack Traces:**
   - **Sentry (Desktop/Web):** Sends JS exception messages, component error details, and JS stack traces (augmented in Desktop by `Document-Policy: include-js-call-stacks-in-crash-reports` / sourcemaps uploaded at build time, `.github/workflows/release.yml:359-361`, `packages/app/vite.config.ts:8-22`). `Breadcrumbs` are disabled (`sentry.ts:22`, `entry.tsx:74`), reducing incidental UI history, though unhandled exception messages/stacks can include URLs, local paths, or error strings.
   - **Markdown Favicons (`packages/session-ui/src/components/markdown.tsx:658`):** Leaks every public external domain hostname rendered inside LLM responses or user prompts (`?domain=${encodeURIComponent(url.host)}&sz=32`) to Google (`www.google.com/s2/favicons`) along with the user's IP/browser headers.
   - **OTLP (Opt-In Only):** Sends structured Effect logs and spans (`SessionStep`, `@ai-sdk` spans via global `AsyncLocalStorageContextManager`, `packages/util/src/observability/otlp.ts:88-91`) to the user's OTLP endpoint, along with `opencode.client`, `deployment.environment.name`, `service.version`, `opencode.run`, and `service.instance.id`.
   - **Mobile Push Relay (`packages/core/src/push/sender.ts:53-67,84-96`, `crypto.ts:34-39`):** Notification JSON (`id`, `type`, `sessionID`, `title`, truncated `body`, `data`) is **end-to-end encrypted** (`AES-256-GCM`) with a P-256 ECDH + HKDF-SHA256 key before leaving the host; the relay receives `{ box }` (and optional `silent: true`).
4. **Behavioral Profile Fields:**
   - Client binaries (`CLI`, `CLI-RS`, `Desktop`) collect **no** behavioral profile metrics (no session count, duration, tool usage histograms, or token counts sent home; `opencode stats` queries only the local server DB via `client.experimental.session.stats`, `packages/cli/src/commands/handlers/stats.ts:31-42`).
   - Only `opencode.ai` website visitors encounter funnel behavioral tracking (page views, section views, clicks, copy actions, download OS match via `funnelConfig` + `/console/funnel/v1.js`, `services/site/src/lib/funnel.ts:1-6`, `funnel-config.ts:77-84`).

---

## 6. Docs / Privacy Policy vs Code Discrepancies

1. **Broken Production Crash Suppression vs Code Comments (`sentry.ts` & `app.tsx`):**
   - `packages/app/src/app.tsx:62-64` explicitly documents: *"Prod crashes reach Sentry only when someone clicks Report; other channels send them as they happen."*
   - In reality, `packages/desktop/src/renderer/startup/sentry.ts:24` and `packages/app/src/entry.tsx:74` check `import.meta.env.OPENCODE_CHANNEL === "prod"` instead of `import.meta.env.VITE_OPENCODE_CHANNEL`, which is always `undefined` in Vite renderer/web bundles (`packages/desktop/electron.vite.config.ts:110`, `packages/app/vite.js:113`). Production builds therefore still keep Sentry `GlobalHandlers` (and Desktop `BrowserApiErrors`) enabled automatically without clicking Report.
2. **Misleading `username` Telemetry Schema Description (`packages/schema/src/config.ts:52-54`):**
   - The `username` config schema field states: `"Username displayed in conversations and used for telemetry identity"`.
   - Forensic check across `packages/` confirms `.username` is **never** passed to Sentry (`setUser`), OTLP (`resource` attributes), updater requests, or `models.dev` requests.
3. **V2 Documentation Omits Opt-Out Flags & Desktop Telemetry:**
   - Searching V2 docs (`services/site/src/docs/content/`) yields **zero** documentation for `OPENCODE_DISABLE_MODELS_FETCH`, `OPENCODE_DISABLE_AUTOUPDATE`, `OTEL_EXPORTER_OTLP_ENDPOINT`, Desktop Sentry crash reporting, Desktop's un-disableable 10-minute updater polling, or Google favicon leaks (`rg` across `services/site/src/docs/content/` found only `OPENCODE_DISABLE_CLAUDE_CODE*`, `OPENCODE_DISABLE_FREE_MODELS`, `OPENCODE_DISABLE_PROJECT_CONFIG`, and `migrate-v1.mdx:480` noting v1's `experimental.openTelemetry` removal). Meanwhile, `network.mdx:90` tells corporate TLS users to allow `opencode.ai` and `*.opencode.ai` without documenting the background update/changelog/favicon/Sentry traffic.
4. **Update Worker Raw IP Collection vs Website Privacy Retention Text:**
   - `services/site/src/lib/legal/privacy-policy.ts:592` highlights that archived web analytics copies exclude email/name/IP, and `privacy-policy.ts:583-586` documents that GPC/opt-out still records unlinked web events. However, the Desktop/CLI update worker (`services/update/src/index.ts:68-71`) logs **raw** `cf-connecting-ip`, `cf_country`, `cf_colo`, and `User-Agent` into `env.EVENTS` (Cloudflare Pipelines) for every CLI and un-disableable Desktop 10-minute update check.

---

## 7. Zero-Telemetry / Negative-Check Verification

Targeted read-only `rg` checks across `packages/`, `services/`, and `.github/` verified the absence of:

- **Third-Party Analytics / Feature Flags:** Zero occurrences of Mixpanel, Bugsnag, Rollbar, Datadog, Plausible, Umami, Statsig, LaunchDarkly, Segment (`segment.io` / `@segment`), or Amplitude SDKs (`amplitude` matched only UI waveform math and a Figma reference doc; `posthog` matched only MCP proxy header handling in `packages/core/src/plugin/mcp-codemode-defaults.ts:12-33` for user-configured PostHog MCP servers).
- **`DO_NOT_TRACK` Support:** Zero occurrences of `DO_NOT_TRACK` in `packages/`, `services/`, or `.github/`.
- **Hardware / OS Machine Fingerprinting:** Zero occurrences of `node-machine-id`, `IOPlatformUUID`, `MachineGuid`, `/etc/machine-id`, or `macaddress`.
- **Competitor Agent Auth Harvesting:** Zero reads of `~/.codex`, `~/.cursor`, `~/.gemini`, `~/.windsurf`, `~/.copilot`, or `claude_desktop_config` across `packages/core/src`, `packages/cli/src`, `packages/cli-rs/crates`, and `packages/desktop/src` (`.cursor` matched only CSS cursor classes / pagination cursors and prompt instructions in `packages/core/src/plugin/command/initialize.txt:14`).
- **Session Sharing Exfiltration:** `opncd.ai` session sharing is discontinued and replaced with an HTTP 410 tombstone and zero-storage no-op endpoints (`services/legacy/src/share.ts:9,161-171`).

---

```json
{
  "slug": "opencode",
  "platform": "OpenCode",
  "group": "B",
  "repo": "anomalyco/opencode",
  "sha": "4c2977b82ece65dfebc4c25058f76e4f8a241c61",
  "commit_date": "2026-11-24T23:52:07-05:00",
  "license": "MIT",
  "status": "Active (sst/opencode redirects here)",
  "default_state": "Mixed: Core CLI/server has zero analytics SDKs and opt-in OTLP, but defaults ON for 5m models.opencode.ai and 10m opencode.ai/update polling (which logs raw IP/geo). Desktop release builds enable Sentry with no runtime opt-out, 10m auto-update polling with no opt-out, What's New changelog fetches, and Google Favicon fetches for markdown links.",
  "opt_out_controls": [
    "OPENCODE_DISABLE_AUTOUPDATE=1|true or config.update='disable' (CLI only; ignored by Desktop updater)",
    "OPENCODE_DISABLE_MODELS_FETCH=1|true or OPENCODE_MODELS_URL / OPENCODE_MODELS_PATH",
    "OTEL_EXPORTER_OTLP_ENDPOINT unset by default (strictly opt-in OTLP)",
    "Desktop Settings -> General -> Release notes switch (disables changelog.json)",
    "OPENCODE_DISABLE_CLAUDE_CODE=1 / _PROMPT=1 / _SKILLS=1 (disables ~/.claude file reads)",
    "opencode.ai web only: GPC or oc_optout=1 cookie (strips IDs/cookies; still logs unlinked events)",
    "None for Desktop/Web Sentry crash reporting, Desktop auto-update polling, or Markdown Google Favicon loads"
  ],
  "do_not_track": "Ignored / unsupported across CLI, Desktop, and opencode.ai web (explicitly disclaimed in services/site/src/lib/legal/privacy-policy.ts:476).",
  "opt_out_bypasses": [
    "Sentry prod suppression bug: packages/desktop/src/renderer/startup/sentry.ts:24 and packages/app/src/entry.tsx:74 check import.meta.env.OPENCODE_CHANNEL === 'prod' instead of VITE_OPENCODE_CHANNEL, evaluating undefined === 'prod' (false) and keeping GlobalHandlers/BrowserApiErrors active in prod despite packages/app/src/app.tsx:62-64 claiming prod crashes only send when clicking Report.",
    "Desktop updater (packages/gui-plugins/src/updater/machine.ts:207-211, main.ts:17) ignores OPENCODE_DISABLE_AUTOUPDATE and config.update='disable' with no UI opt-out.",
    "opencode.ai web analytics still records page views, clicks, and account/workspace events when GPC or oc_optout=1 is active, merely omitting oc_vid/oc_sid/oc_attr cookies (privacy-policy.ts:583-586)."
  ],
  "persistent_ids": "No persistent device/machine telemetry ID in CLI/Desktop (OTLP uses ephemeral per-process 8-hex-char runID()). Server-side services/update computes deterministic 32-bit IP rollout hash SHA-256(channel:ip) and logs raw cf-connecting-ip. Local-only IDs: Project.ID (SHA-1 of git remote/dir), Ed25519 machine/home-key (P2P Iroh), opt-in mobile Push DeviceID. Website sets 1y cookies oc_vid/oc_sid/oc_attr.",
  "hashing": "SHA-256('${resolved}:${ip}') truncated to uint32 in services/update/src/index.ts:119 for rollout bucketing; SHA-1 (Hash.fast) for local Project.ID and cache/port derivation; SHA-256 for catalog digest.",
  "external_auth_reads": "No cross-tool auth harvesting (.codex/.cursor/.gemini/.windsurf/.copilot not read). Reads ~/.local/share/opencode/auth.json once on v1->v2 DB migration; ~/.claude/CLAUDE.md, .claude/skills, ~/.claude/ide/*.lock (opt-out via OPENCODE_DISABLE_CLAUDE_CODE*); Zed SQLite DB; ~/.aws/{config,credentials} profile names only; ~/.config/gcloud/{active_config,configurations,application_default_credentials.json} project/region only; macOS MDM ai.opencode.managed.plist; and user-initiated Desktop Browser cookie/history imports from Chrome/Edge/Brave/Arc/Vivaldi/Opera/Firefox/Safari/etc.",
  "telemetry_endpoints": [
    "VITE_SENTRY_DSN (Desktop releases & Web if configured; no runtime opt-out)",
    "https://opencode.ai/update/api/{channel}/{artifact}/{distribution} (CLI & Desktop 10m update checks -> services/update Cloudflare Pipelines)",
    "https://formulae.brew.sh/api/formula/opencode.json (CLI Homebrew Core update check)",
    "https://models.opencode.ai/experimental/v2.0/api.json?... (5m model catalog fetch; opt-out via OPENCODE_DISABLE_MODELS_FETCH)",
    "https://opencode.ai/changelog.json (Desktop What's New on version upgrade; UI switch)",
    "https://www.google.com/s2/favicons?domain=...&sz=32 (Markdown public link favicons; no opt-out)",
    "https://opencode.ai/favicon-96x96-v3.png (Desktop/Web notification icon)",
    "https://api.github.com/repos/anomalyco/opencode/contributors?anon=1&per_page=1 (Settings -> About view)",
    "${OTEL_EXPORTER_OTLP_ENDPOINT}/v1/logs & /v1/traces (strictly opt-in OTLP)",
    "https://opencode-notify-relay.anomalyco.workers.dev & https://notify.dev.opencode.ai (opt-in E2EE mobile push)",
    "/console/funnel/v1.js (opencode.ai web/docs funnel analytics)"
  ],
  "ip_geo_handling": "Update worker (services/update/src/index.ts:68-71) logs raw cf-connecting-ip, cf_country, cf_colo, user-agent, and path to Cloudflare Pipelines (env.EVENTS.send) with zero IP suppression and hashes raw IP for rollout bucketing. Client Sentry does not set sendDefaultPii and strips Breadcrumbs, but Sentry HTTP ingest still sees client IP unless stripped server-side.",
  "custom_model_handling": "Private/custom model names are NOT sent in default CLI/Desktop network requests. When user opts into OTLP (OTEL_EXPORTER_OTLP_ENDPOINT), verbatim unhashed gen_ai.provider.name and gen_ai.request.model are emitted on spans (packages/core/src/session/runner/step.ts:83-91,135-136).",
  "docs_discrepancies": [
    "packages/app/src/app.tsx:62-64 claims prod Sentry crashes only send when clicking Report, contradicted by undefined import.meta.env.OPENCODE_CHANNEL === 'prod' checks in sentry.ts:24 and entry.tsx:74.",
    "packages/schema/src/config.ts:53 documents username as 'used for telemetry identity', though .username is unused in all telemetry paths.",
    "V2 docs (services/site/src/docs/content/) omit OPENCODE_DISABLE_MODELS_FETCH, OPENCODE_DISABLE_AUTOUPDATE, OTEL_EXPORTER_OTLP_ENDPOINT, Desktop Sentry/10m updater, and Google favicon requests.",
    "privacy-policy.ts:592 emphasizes no IP in archived web analytics, while services/update/src/index.ts:68-70 logs raw cf-connecting-ip and cf_country/cf_colo to Cloudflare Pipelines on every CLI/Desktop update check."
  ],
  "privacy_rating": "D",
  "one_line_verdict": "Core CLI/server has clean opt-in OTLP and no analytics SDKs, but Desktop/Web ships un-disableable Sentry with a broken prod opt-out filter, un-disableable 10m update checks hitting a raw-IP/geo update data lake, automatic Google favicon leaks on markdown links, and V2 docs/DNT gaps."
}
```
