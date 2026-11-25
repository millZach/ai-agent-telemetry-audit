# Privacy & Telemetry Forensic Audit: OpenHands (`OpenHands/OpenHands`)

## Header Metadata

- **Slug:** `openhands`
- **Platform:** `OpenHands`
- **Group:** `B`
- **Repository:** `OpenHands/OpenHands` (`https://github.com/OpenHands/OpenHands`; `All-Hands-AI/OpenHands` redirects here)
- **License:** `MIT`
- **Status:** `Active (All-Hands-AI/OpenHands redirects here)`
- **Branch:** `main`
- **Commit SHA:** `adf71d626b81a0db00902e0d3edf2b00d2cf917c`
- **Commit Date:** `2026-11-24T23:20:32-05:00`
- **Commit Subject:** `feat: add Feedback button and submission flow (#19272)`

> **Repository Architecture Note (Agent Canvas v1.45.0 Layout at Audited Commit):** At this `main` SHA, `OpenHands/OpenHands` is the **OpenHands Agent Canvas** control center (`@openhands/agent-canvas` v`1.45.0`, `package.json:2-3`), comprising the React Router/Vite SPA and npm library (`src/`), Node.js CLI/stack launchers (`bin/`, `scripts/`), Electron desktop shell (`electron/`), and Docker/Helm packaging (`docker/`, `helm/`). Legacy directories (`openhands/`, `enterprise/`, `frontend/`, `pyproject.toml`) are absent; the Python `openhands-agent-server` (`OpenHands/software-agent-sdk`) and `openhands-automation` (`OpenHands/automation`) backends live in sibling repositories (`README.md:214-222`) and are spawned via `uvx` or installed inside the container image (`docker/Dockerfile`), receiving telemetry environment variables exported by this repository's launchers and entrypoint.

---

## 1. Default State & Opt-Out Mechanism

OpenHands Agent Canvas exhibits **Active Pre-Consent Default Telemetry** backed by PostHog (`posthog-js` `1.450.0`, `package.json:52`) routed by default through OpenHands' own ad-blocker-bypass reverse proxy (`https://z.openhands.dev`, `src/services/telemetry.ts:14-18,74-78`). On Cloud backends, analytics consent is forced ON and UI controls are disabled; across local/CLI/Docker/Desktop deployments, runtime opt-out controls are fragmented and contain multiple bypasses.

### A. Default Telemetry Behavior

1. **Pre-Consent `canvas_install` Beacon (ON Before Prompt):**
   - `src/entry.client.tsx:52-55` wraps the app in `AgentServerUIProviders` with `DEFAULT_AGENT_SERVER_ANALYTICS = { provider: "posthog" }` (`src/components/providers/agent-server-ui-providers.tsx:34-36`), which mounts `TelemetryProvider` (`src/components/providers/agent-server-ui-providers.tsx:112-113`) and `<TelemetryLifecycle />` -> `useTelemetry()` (`src/components/providers/telemetry-provider.tsx:283-286,319-324`).
   - On first mount, `useTelemetry()` (`src/hooks/use-telemetry.ts:77-83`) unconditionally calls `trackInstall()` (`src/services/telemetry.ts:759-801`). Unless `isDoNotTrackEnabled()` is true or `localStorage["openhands-telemetry-first-use"] === "true"`, `trackInstall()` initializes PostHog with capturing enabled (`initializePostHogClient(true)`), temporarily calls `posthog.opt_in_capturing()`, emits `"canvas_install"` with browser/origin metadata (`src/services/telemetry.ts:771-793`), and only *afterward* restores `posthog.opt_out_capturing()` if consent is not `"granted"` (`src/services/telemetry.ts:797-800`).
2. **Pre-Checked Consent Banner & Default-True Settings Switch (Local Backends):**
   - When `user_consents_to_analytics === null` on a connected local backend (`src/root.tsx:254-268`, `src/components/features/analytics/telemetry-consent-banner.tsx:43-47`), `TelemetryConsentBanner` renders the checkbox `"Send anonymous usage data"` (`src/i18n/translation.json:24033`) with **`defaultChecked`** (`src/components/features/analytics/telemetry-consent-banner.tsx:96-102`).
   - In **Settings -> App** (`src/routes/app-settings.tsx:164-168,217-232`), `null` consent is likewise treated as `true` (`// Treat null as true since analytics is opt-in by default`, `settings?.user_consents_to_analytics ?? true`). Note that if a user declines in the banner and saves, `setTelemetryConsent("denied")` sets `localStorage["openhands-telemetry-consent"] = "denied"`, and `useSyncAutomationTelemetryConsent()` (`src/hooks/use-sync-automation-telemetry-consent.ts:90`) relays that choice to the local Automation sidecar via `POST /api/automation/v1/telemetry/consent` (`src/api/automation-service/automation-service.api.ts:487-505`).
3. **Cloud Backends Lock Analytics ON & Override Local Refusal:**
   - `TelemetryConsentBanner` returns `null` whenever `getLockedCloudHost() !== null || backend.kind !== "local"` (`src/components/features/analytics/telemetry-consent-banner.tsx:125`).
   - In **Settings -> App** (`src/routes/app-settings.tsx:51,112-114,217-232`), when `isCloudBackend` is true, the `enable-analytics-switch` is forced toggled ON (`true`), `isDisabled={true}`, omitted from saved settings updates (`...(!isCloudBackend && { user_consents_to_analytics: enableAnalytics })`), and coerced to `enableAnalytics = true`.
   - When hosted on OpenHands Cloud in cookie-auth mode (`getLockedCloudAuthMode() === "cookie"`, `src/api/agent-server-config.ts:85-96,177-179`), `getTelemetryConsent()` **unconditionally returns `"granted"`** (`src/services/telemetry.ts:495-497`), overriding any explicit `"denied"` preference in `localStorage["openhands-telemetry-consent"]`.
4. **Release Build Key Baking & Local Python Backend Propagation:**
   - Unconfigured source builds fall back to a staging key (`phc_kBtz5nKmxVRRQ7HtPwr2QX9eMC5j65zE86QKocVNwb4U`, `config/defaults.json:47`, `src/services/telemetry.ts:70-72`), whereas release workflows inject production `VITE_POSTHOG_API_KEY` (`vars.POSTHOG_PROD_KEY`) and `VITE_POSTHOG_SURVEY_ID` (`vars.POSTHOG_SURVEY_ID`) into npm bundles and `config/defaults.json` (`.github/workflows/npm-publish.yml:65-93`), Docker builds (`.github/workflows/docker.yml:108-109,174-205`, `docker/Dockerfile:46-54,73-74`), and Electron macOS/Linux/Windows builds (`.github/workflows/desktop-macos.yml:90-91,173-174`, `.github/workflows/desktop-linux.yml:66-67`, `.github/workflows/desktop-windows.yml:60-61`).
   - Local launchers and the container entrypoint automatically export `OH_TELEMETRY_EXPORTER="posthog"` + `OH_TELEMETRY_POSTHOG_API_KEY` (`scripts/dev-safe.mjs:813-854`, `docker/entrypoint.sh:474-492`) and `AUTOMATION_POSTHOG_API_KEY` (`scripts/dev-with-automation.mjs:987-1003`, `docker/entrypoint.sh:459-468`) using the bundled `config/defaults.json:46-49` key/host when not overridden.

### B. Opt-Out Controls & Fragmentation / Bypasses

| Control / Flag | Intended Scope | Actual Enforcement & Gaps | Source Citation |
| :--- | :--- | :--- | :--- |
| Consent Banner / Settings Switch | Ongoing PostHog events (`trackEvent`, `useTracking`, local Automation) | Banner checkbox is `defaultChecked`; does NOT stop pre-consent `canvas_install`; does NOT stop `posthog.init()` network fetch; **forced ON & disabled on Cloud backends**; no UI button calls `clearTelemetryData()` (`src/services/telemetry.ts:1004-1051`, 0 UI callers). | `src/components/features/analytics/telemetry-consent-banner.tsx:96-102,125`; `src/routes/app-settings.tsx:112-114,217-232` |
| `VITE_DO_NOT_TRACK=1` | Full opt-out (install + session + backends) | **Build-time only for Vite frontend** (`.env.sample:28`); **zero effect at runtime on prebuilt `npx @openhands/agent-canvas`, Electron desktop app, or Docker image frontend**. When set at launch time, `dev-safe.mjs`, `dev-with-automation.mjs`, and `docker/entrypoint.sh` do skip Python backend telemetry keys. | `src/services/telemetry.ts:370-375`; `.env.sample:28`; `scripts/dev-safe.mjs:814-833`; `scripts/dev-with-automation.mjs:988-992`; `docker/entrypoint.sh:462,474-476` |
| `DO_NOT_TRACK=1` | Standard Node / CLI / OS opt-out | Checked by SSR (`process.env?.DO_NOT_TRACK === "1"`), `submitPrivateFeedback()`, `dev-safe.mjs` (`agent-server`), and `docker/entrypoint.sh` (`agent-server`). **IGNORED by prebuilt SPA in the browser, `dev-with-automation.mjs:988` (`AUTOMATION_POSTHOG_API_KEY`), and `docker/entrypoint.sh:462` (`AUTOMATION_POSTHOG_API_KEY`).** | `src/services/telemetry.ts:383-385,962`; `scripts/dev-safe.mjs:815`; `scripts/dev-with-automation.mjs:988`; `docker/entrypoint.sh:462,481` |
| `AGENT_CANVAS_DISABLE_TELEMETRY=1` / `--disable-telemetry` | Runtime opt-out for prebuilt `static-server.mjs` HTML injection (`window.__AGENT_CANVAS_DO_NOT_TRACK__=true`) | **Undocumented** in `README.md`, `.env.sample`, `bin/agent-canvas.mjs --help`, and `docs/`. Works only when passed to `static-server.mjs`; **NOT forwarded by `docker/entrypoint.sh:459-492`, `dev-safe.mjs:814-815`, or `dev-with-automation.mjs:988` to Python backends**, and ignored unless `static-server.mjs` serves `index.html`. | `scripts/static-server.mjs:82-86,104,280-285,441-443`; `src/services/telemetry.ts:143-154,378-380` |
| `navigator.doNotTrack === "1"` / `window.doNotTrack === "1"` | Browser DNT header/API | Disables `isDoNotTrackEnabled()` in `src/services/telemetry.ts:388-395`, stopping `canvas_install` and `initializePostHogClient()`. **Bypassed by:** `submitPrivateFeedback()` (`#19272`, which checks `isTelemetryHardDisabled()`, `VITE_DO_NOT_TRACK`, and `process.env.DO_NOT_TRACK`, omitting `navigator.doNotTrack`), `reportCanvasAuthenticated()`, npm update checks, and Python backends. | `src/services/telemetry.ts:388-395,959-965`; `src/api/main-app-auth.ts:32-47` |
| `configureTelemetry(false)` / `analytics={false}` | Library consumer programmatic disable | Disables `TelemetryProvider` and PostHog calls in embedded host apps (`AgentServerUIProvidersProps.analytics`). | `src/services/telemetry.ts:290-298`; `src/components/providers/agent-server-ui-providers.tsx:104-113` |

### C. Specific Opt-Out Bypasses Verified in Source

1. **`posthog.init()` Remote Config/Flags Fetch Even When Opted Out:**
   `TelemetryProvider` (`src/components/providers/telemetry-provider.tsx:311-317`) unconditionally runs `initializePostHogClient()` on mount whenever `analyticsEnabled` is true (default), **even when `getTelemetryConsent()` is `"pending"` or `"denied"`**. As acknowledged in `src/services/telemetry.ts:320-321` (*"initializing one anyway would still fetch remote config and flags"*), unless `isDoNotTrackEnabled()` returns true, PostHog still initializes against `https://z.openhands.dev` on page load and calls `posthog.opt_out_capturing()` afterward (`src/services/telemetry.ts:438-468`).
2. **Main-App Cookie-Auth `canvas_authenticated` Event Ignores All Consent & DNT:**
   `authenticateWithMainAppCookie()` (`src/api/main-app-auth.ts:49-56`) calls `reportCanvasAuthenticated()` (`src/api/main-app-auth.ts:32-47`), which issues `POST /api/analytics/events` with `{ event_type: "canvas_authenticated", client_version: AGENT_CANVAS_CLIENT_VERSION }` and `withCredentials: true` without checking `isDoNotTrackEnabled()`, `getTelemetryConsent()`, or `navigator.doNotTrack`.
3. **Private Feedback Submission Flow (PR `#19272`) Bypass Scope:**
   `<FeedbackButton />` (`src/routes/home.tsx:21-23`, `src/components/features/feedback/feedback-button.tsx:16-167`) is strictly user-initiated, requires `VITE_POSTHOG_SURVEY_ID` (injected in CI releases), and calls `submitPrivateFeedback()` (`src/services/telemetry.ts:955-999`). By design, it ignores `openhands-telemetry-consent` (*"This does not enable analytics"*, `src/i18n/translation.json:41925`), uses `credentials: "omit"` and `referrerPolicy: "no-referrer"`, sets `ip: false` and `$process_person_profile: false` with a fresh `randomUUID()`, and checks `isTelemetryHardDisabled()`, `VITE_DO_NOT_TRACK`, and `process.env.DO_NOT_TRACK` (`src/services/telemetry.ts:959-965`) — **however, it fails to check `navigator.doNotTrack`**.

---

## 2. External Auth / Local Non-Workspace File Reads

OpenHands does **not** harvest, upload, or transmit external coding tools' credentials or secrets over telemetry. Its external auth inspection is limited to explicit **local-backend-only** ACP CLI status detection and optional user-initiated DigitalOcean MARS storage:

| Non-Workspace Source / Command | Trigger & Gate | Data Read / Transmitted | Telemetry Impact | Source Citation |
| :--- | :--- | :--- | :--- | :--- |
| `claude auth status --json` (`~/.claude/.credentials.json` / macOS Keychain via Claude CLI) | Runs **only** on `backend.kind === "local"` when onboarding ACP credential step is active or Settings -> Agent is open (`enabled && isLocal && !!providerKey`). | Parses JSON `loggedIn` boolean; returns `"authenticated" \| "unauthenticated" \| "unknown"`. Raw credentials/tokens are never read or returned. | Never sent to telemetry (only drives UI banner `"already signed in"` vs API-key form). | `src/api/acp-service/acp-service.api.ts:34-46,76-79,111-118`; `src/hooks/query/use-acp-auth-status.ts:66-99`; `docs/ACP_AGENTS.md:75-78` |
| `codex login status` (`~/.codex/auth.json` via Codex CLI) | Same local-only UI gate (`useAcpAuthStatus`). | Matches `"not logged in"` / `"logged in"` text on stdout+stderr. | None. | `src/api/acp-service/acp-service.api.ts:52-57,80-83` |
| `$HOME/.gemini/oauth_creds.json` | Same local-only UI gate (`test -f "$HOME/.gemini/oauth_creds.json" && echo present \|\| echo absent`). | File existence check only (`"present"` / `"absent"`); file content never read. | None. | `src/api/acp-service/acp-service.api.ts:63-71,84-88` |
| `${PI_CODING_AGENT_DIR:-$HOME/.pi/agent}/auth.json` | Same local-only UI gate (`grep -qs '"type"' ...`). | Key presence check (`$ANTHROPIC_API_KEY` or `"type"` substring); returns `"present"` / `"absent"`, never extracts secrets. | None. | `src/api/acp-service/acp-service.api.ts:63-71,89-94` |
| OS Keychain (`security` / `secret-tool`, service `"openhands-agent-canvas"`) or `~/.openhands/agent-canvas/digitalocean/{secrets.json,mars-credentials.json}` | **Only** when DigitalOcean MARS integration is enabled (`MARS_WEB=1`, `MARS_TOKEN`, or Electron desktop app). | Stores/reads user's DigitalOcean PAT/OAuth tokens (never exposed to renderer; encrypted via AES-256-GCM or `safeStorage` when not in keychain) + metadata `deviceId`. | Never sent to PostHog; `deviceId` sent only in `X-Device-UUID` header to `api.digitalocean.com`. | `scripts/mars-secret-store.mjs:1-21,41-46`; `scripts/mars-credentials.mjs:25,36-38,78-85`; `scripts/mars-api.mjs:14,26` |

> **Containerized ACP Note:** On remote/Docker backends, `useAcpAuthStatus` disables host probing (`isSupported = backend.kind === "local"`, `src/hooks/query/use-acp-auth-status.ts:72-73`) and prompts the user to manually paste container secrets (`setup-acp-secrets-step.tsx:37-50`, `acp-auth-status-banner.tsx:11-40`).

---

## 3. Persistent Device / Install IDs & Hashing

### A. Persistent Identifiers & Cross-Boundary Identity Stitching

1. **PostHog Persistent Client IDs (`localStorage["ph_agent-canvas_posthog"]`):**
   - `posthog.init()` uses `persistence: "localStorage"`, `persistence_name: "agent-canvas"`, `consent_persistence_name: "agent-canvas-consent"`, and `person_profiles: "always"` (`src/services/telemetry.ts:444-447`). This persists a PostHog `distinct_id` and `$device_id` across sessions, generated even when `canvas_install` fires pre-consent (`src/services/telemetry.ts:759-793`).
   - `localStorage` also stores `openhands-telemetry-consent`, `openhands-telemetry-first-use`, `openhands-telemetry-consent-pending-cloud-sync`, `openhands-telemetry-consent-pending-local-revocation`, and `posthog_bootstrap:consumed_nonces`; `sessionStorage` stores `openhands-telemetry-session` and `posthog_bootstrap` (`src/services/telemetry.ts:55-63`, `src/components/providers/telemetry-provider.tsx:13-16`).
2. **Website-to-App Bootstrap Handoff (`oh_ph_handoff` / `distinct_id` / `session_id`):**
   - `TelemetryProvider` parses `oh_ph_handoff` (Base64URL JSON containing `distinct_id`, `session_id`, `nonce`, `exp`, and `attribution`) or raw `distinct_id` + `session_id` URL query/hash params (`src/components/providers/telemetry-provider.tsx:174-238`), seeds `pendingBootstrap` (`src/services/telemetry.ts:406-412,451`), and strips them from the address bar via `history.replaceState` (`src/components/providers/telemetry-provider.tsx:150-172`) so PostHog merges marketing website sessions (`openhands.dev`) directly with Agent Canvas activity.
3. **Anonymous-to-Cloud Identity Aliasing (`posthog.identify`):**
   - When connected to a Cloud backend, `useTelemetryIdentity()` (`src/hooks/use-telemetry-identity.ts:54-99`) resolves the user's `userId`, `email` (`settings?.email || settings?.git_user_email`), and `orgId`, calling `setTelemetryCloudContext({ userId, email, orgId })` and `setTelemetryIdentity(userId, email ? { email } : {})`.
   - When consent is `"granted"`, `applyDesiredTelemetryIdentity()` (`src/services/telemetry.ts:249-273`) invokes `posthog.identify(userId, { email })`, retroactively stitching prior anonymous `canvas_install` / website-handoff events to the identified Cloud account (`.agents/skills/telemetry-analytics/references/guide.md:11`).
4. **Frontend-to-Local-Backend ID Stitching:**
   - When consent is granted, `AgentServerConversationService` fetches `getTelemetryDistinctId()` and sends it as **`user_id`** inside local `createConversation()` payloads (`src/api/conversation-service/agent-server-conversation-service.api.ts:796-805`).
   - `AutomationService` attaches PostHog's `distinct_id` on every local automation HTTP request via header **`X-OpenHands-Telemetry-Distinct-Id`** (`src/api/client-source.ts:14-15`, `src/api/automation-service/automation-service.api.ts:205-223`) and passes `frontend_distinct_id` in `POST /api/automation/v1/telemetry/consent` (`src/api/automation-service/automation-service.api.ts:493-499`).
5. **Ephemeral Feedback UUID (`#19272`):**
   - `submitPrivateFeedback()` generates a fresh per-submission `randomUUID()` for `distinct_id` with `$process_person_profile: false` (`src/services/telemetry.ts:983,989`), preventing linkage to the PostHog tracking profile.
6. **DigitalOcean MARS Device UUID:**
   - `mars-credentials.mjs:81` generates `deviceId: randomUUID()` persisted in `~/.openhands/agent-canvas/digitalocean/mars-credentials.json` (`scripts/mars-credentials.mjs:25`) and transmitted only as `X-Device-UUID` to `https://api.digitalocean.com` (`scripts/mars-api.mjs:14,26`) when MARS is actively used.

### B. Cryptographic Hashing

- **Zero Unsalted Cross-Provider Telemetry Hashing:** Agent Canvas performs no SHA-1/SHA-256 hashing of git remotes, machine IDs, emails, or repo paths for telemetry (`hashing` for telemetry is none).
- **Local-Only Token Fingerprint:** `scripts/mars-credentials.mjs:74-76` computes an unsalted SHA-256 prefix (`createHash("sha256").update(token).digest("hex").slice(0, 16)`) solely inside local `mars-credentials.json` to recognize duplicate DigitalOcean PAT entries without storing plaintext tokens alongside metadata.

---

## 4. Exact Outbound Telemetry / Analytics / Crash / OTEL / Relay / Update Endpoints

| Endpoint / Host | Purpose | Default State & Trigger | Opt-Out Condition | Source Citation |
| :--- | :--- | :--- | :--- | :--- |
| **`https://z.openhands.dev`** | OpenHands reverse proxy forwarding to PostHog US (`DEFAULT_POSTHOG_HOST`), explicitly built to **bypass ad blockers** (`api_host`) + `#19272` `/api/surveys/?token=...` and `/capture/` | **ON by default** for `canvas_install` (pre-consent), SDK init/flags fetch on load, consented events, and user-clicked Feedback (`#19272`) | `VITE_DO_NOT_TRACK=1` (build), `AGENT_CANVAS_DISABLE_TELEMETRY=1` / `--disable-telemetry` (static-server runtime), `DO_NOT_TRACK=1` (SSR/feedback), `navigator.doNotTrack=1` (blocks SDK init, **not** feedback), or custom `VITE_POSTHOG_HOST` / `configureTelemetry()` | `src/services/telemetry.ts:14-18,74-78,438-455,927-930,970-995` |
| **`https://us.i.posthog.com`** | Direct PostHog US ingestion host (`config/defaults.json:48`), injected by launchers/Docker into local Python `openhands-agent-server` (`OH_TELEMETRY_POSTHOG_HOST`) and `openhands-automation` (`AUTOMATION_POSTHOG_HOST`) | **ON by default** in launched local/Docker backends when PostHog key resolves from `config/defaults.json` or build env | `VITE_DO_NOT_TRACK=1` or explicit `OH_TELEMETRY_EXPORTER` / `DO_NOT_TRACK=1` (note: `DO_NOT_TRACK=1` does **not** disable `AUTOMATION_POSTHOG_API_KEY` in `dev-with-automation.mjs` or `docker/entrypoint.sh`) | `config/defaults.json:46-49`; `scripts/dev-safe.mjs:58-61,835-851`; `scripts/dev-with-automation.mjs:116-117,987-1003`; `docker/entrypoint.sh:459-492` |
| **`https://us.posthog.com`** | Default PostHog UI host (`DEFAULT_POSTHOG_UI_HOST`) | Configured into `posthog.init({ ui_host })` | Same as `z.openhands.dev` SDK init | `src/services/telemetry.ts:80-83,442` |
| **`POST /api/analytics/events`** (same-origin Cloud main app) | Emits `canvas_authenticated` + `client_version` (`withCredentials: true`) | **Automatic** in Cloud cookie-auth mode (`shouldUseMainAppCookieAuth()`) upon `/api/authenticate` success | **None** (does not check telemetry consent or DNT) | `src/api/main-app-auth.ts:5-6,22-24,32-56` |
| **`https://registry.npmjs.org/@openhands/agent-canvas/latest`** | Checks latest published `@openhands/agent-canvas` version (`AgentCanvasUpdateCard`) | **Automatic** on Settings mount (`SettingsDesktopSidebar`, mobile drawer/hub) whenever `getLockedCloudHost() === null`, cached 1h | **None** in UI/env (suppressed only when `getLockedCloudHost()` is non-null) | `src/api/agent-canvas-updates.ts:9-37`; `src/hooks/query/use-latest-agent-canvas-version.ts:12-26`; `src/components/features/settings/agent-canvas-update-card.tsx:308-311`; `src/components/features/settings/settings-desktop-sidebar.tsx:41` |
| **`https://fonts.googleapis.com/css2?...`** (`Outfit`, `IBM Plex Mono`) | CSS `@import` stylesheet loads (which chain to `fonts.gstatic.com` fonts) | **Unconditional** on every app load | **None** | `src/index.css:1-2` |
| **`POST /api/automation/v1/telemetry/consent`** (local sidecar) | Relays browser `consent_granted` + `frontend_distinct_id` to local Automation backend | **Automatic** whenever local backend consent resolves or is revoked (`useSyncAutomationTelemetryConsent`) | Disabled when connected to Cloud backend | `src/api/automation-service/automation-service.api.ts:487-505`; `src/hooks/use-sync-automation-telemetry-consent.ts:48-90` |
| **`X-OpenHands-Client` / `X-OpenHands-Client-Version` HTTP Headers** | Coarse client identification sent to Agent Server / Cloud / Automation (`Datadog` facets per `guide.md:19`) | Included on Cloud & Automation API calls | **None** (coarse version/client label only; documented non-PII) | `src/api/client-source.ts:3-20`; `.agents/skills/telemetry-analytics/references/guide.md:17-19` |

*(No Electron auto-updater (`electron-updater` / `autoUpdater`), Sentry, Segment, or OTLP collector endpoint exists in this repository; see Section 7.)*

---

## 5. Exact Data & Properties Collected

### A. IP Address & Geolocation Handling

- **No IP Suppression on Main PostHog SDK:** `posthog.init()` (`src/services/telemetry.ts:438-455`) sets neither `ip: false` nor `property_blacklist` to drop `$ip` / GeoIP properties. All client telemetry hits OpenHands' reverse proxy **`https://z.openhands.dev`** (`src/services/telemetry.ts:77-78`) before forwarding to PostHog US, exposing the user's client IP and derived geolocation to both OpenHands infrastructure and PostHog.
- **Feedback Endpoint Exception (`#19272`):** `submitPrivateFeedback()` (`src/services/telemetry.ts:975-995`) sets `"ip": false` in the `/capture/` JSON body (instructing PostHog to discard GeoIP) along with `credentials: "omit"` and `referrerPolicy: "no-referrer"`, though the HTTPS transport request still connects directly to `z.openhands.dev`.

### B. Custom & Private Model Names

- **Verbatim Unsanitized Transmission (`LLM_MODEL`):** When a user saves LLM settings in the settings modal (`src/components/shared/modals/settings/settings-form.tsx:41-50`), `trackSettingsSaved()` (`src/hooks/use-tracking.ts:268-285`) emits event `"settings_saved"` containing **`LLM_MODEL: llmModel`** (`agentLlm.model`) verbatim without redaction, hashing, or known-provider bucketing (`openai/*` vs custom/self-hosted model identifiers), alongside `LLM_API_KEY_SET` (`"SET" | "UNSET"`), `SEARCH_API_KEY_SET`, `REMOTE_RUNTIME_RESOURCE_FACTOR`, and `current_url`. (API keys and base URLs themselves are not included in `trackSettingsSaved`.)

### C. Prompts, Code, Paths, URLs & Stack Traces

- **Prompts & Code:** User prompt strings and workspace file contents are **not** transmitted to telemetry. `trackInitialQuerySubmitted` and `trackUserMessageSent` (`src/hooks/use-tracking.ts:235-262`, `src/components/features/chat/chat-interface.tsx:562-572`) capture only numeric character lengths (`query_character_length`, `current_message_length`, `replay_json_size`, `session_message_count`).
- **Full Page URLs & Origins (`current_url`, `url_origin`, `referrer`):**
  - Every event emitted through `useTracking()` (`src/hooks/use-tracking.ts:70-82`) and `trackError()` (`src/utils/error-handler.ts:37-46`) attaches **`current_url: window.location.href`** verbatim — leaking self-hosted hostnames, paths (such as `/conversations/<id>`), and any non-stripped URL query parameters.
  - Pre-consent `canvas_install` (`src/services/telemetry.ts:783-793`) attaches `url_origin: window.location.origin` and `document.referrer`.
- **Error Diagnostics (`error_outcome`):** `trackError()` (`src/utils/error-handler.ts:17-47`), called from WebSocket conversation/agent error handlers (`src/contexts/conversation-websocket-context.tsx:859-888,1154-1183`), intentionally drops raw error messages and stack traces, transmitting `current_url`, `error_source`, `error_kind`, `error_id`, `error_telemetry` (`"diagnostic" | "outcome"`), and metadata such as `errorCode`, `toolName`, and `toolCallId`.

### D. Behavioral, Environment & Account Profile Fields

- **`before_send` Global Properties (`addCanvasEventProperties`, `src/services/telemetry.ts:211-231`):** Appends to *every* PostHog event:
  - Package/Client: `client_source` (`"agent_canvas"`), `client_version`, `package_name`, `package_version` (`src/services/telemetry.ts:168-173`).
  - Backend Context (`src/services/telemetry-context.ts:43-67`): `backend_kind`, `deployment_kind` (`"local" | "remote"`), `agent_server_version`, `automation_sdk_version`, `backend_version`, `connection_method` (`"manual" | "cloud_login" | "cloud_cookie"`).
  - Cloud Identity Context (`src/services/telemetry-context.ts:69-77`): **`cloud_user_id`**, **`cloud_user_email`** (`settings?.email || settings?.git_user_email` via `use-telemetry-identity.ts:63-64,84`), and **`cloud_org_id`** (populated while Cloud backend is active, plus `email` on `posthog.identify()`).
  - Website Attribution (`src/components/providers/telemetry-provider.tsx:29-37`, capped at 80 chars): `utm_source`, `utm_medium`, `utm_campaign`, `landing_page_category`, `cta_id`, `cta_surface`, `referring_domain_category`.
- **Pre-Consent `canvas_install` (`src/services/telemetry.ts:783-793`):** `platform`, `user_agent`, `referrer`, `url_origin`, `embedded` (iframe bool).
- **Behavioral Events (`src/hooks/use-tracking.ts:84-490`, `src/services/cloud-funnel-analytics.ts:33-63`):**
  - `canvas_new_session` (`is_first_use`)
  - `login_button_clicked` (`provider`), `backend_added` (`has_api_key`, `source`), `cloud_device_authorization_started`, `cloud_device_authorization_succeeded`, `cloud_conversation_ready` (`$insert_id`, `task_id`, `conversation_id`)
  - `conversation_start_requested` (`conversation_id`, `task_id`, `is_start_task`, `has_repository`, `git_provider`, `has_workspace`, `workspace_mode`, `has_initial_query`, `agent_type`, `has_parent_conversation`, `entry_point`)
  - `push_button_clicked`, `pull_button_clicked`, `create_pr_button_clicked`, `download_via_vscode_button_clicked`, `download_trajectory_button_clicked`, `conversation_exported` (`format`), `mcp_config_updated` (`has_mcp_config`, `sse_servers_count`, `stdio_servers_count`)
  - `onboarding_started`, `onboarding_step_viewed` / `onboarding_skipped` (`step`, `step_index`, `total_steps`, `agent`), `onboarding_completed` (`agent`), `onboarding_link_clicked`
  - Automation funnel (`prebuilt_automation_enabled`, `automation_setup_{opened,blocked,validated,created,failed}`, `automation_{created_button,executed,deleted,disable_button,edited,exported,activity_log_exported,imported}`, `git_sync_{config_updated,triggered}`)
  - Manual Feedback (`#19272`, `src/services/telemetry.ts:980-994`): `$survey_id`, `$survey_questions`, `$survey_response` (user free text), `$survey_completed`, `$process_person_profile: false`, `feedback_source: "agent_canvas"`, optional `feedback_email`.

---

## 6. Docs / Privacy Policy vs Code Discrepancies

1. **Zero Telemetry or Privacy Documentation in User-Facing Docs:**
   - Negative check across `README.md`, `README.windows.md`, `docs/`, and `helm/agent-canvas/README.md` (`rg -n -i "(privacy|telemetry|posthog|do_not_track|donottrack|disable-telemetry|AGENT_CANVAS_DISABLE_TELEMETRY|z\.openhands\.dev|feedback)"`) returned only `README.md:47` (Slack feedback link) and **zero `PRIVACY*` files** in the repository.
   - Users of `npx @openhands/agent-canvas`, Docker, Helm, or the Desktop app are never informed in `README.md`, `docs/SELF_HOSTING.md`, or `bin/agent-canvas.mjs --help` (`bin/agent-canvas.mjs:58-129`) about PostHog telemetry, the pre-consent `canvas_install` event, `https://z.openhands.dev`, npm registry update checks, Google Fonts loads, or `AGENT_CANVAS_DISABLE_TELEMETRY=1` / `--disable-telemetry`.
2. **Banner Claims *"No personal information is collected"* vs `cloud_user_email`, `cloud_user_id` & IP Collection:**
   - `TELEMETRY$CONSENT_DESCRIPTION` in `src/i18n/translation.json:23997` states: *"We collect anonymous usage data to improve the product. No personal information is collected."*
   - In code, `useTelemetryIdentity()` attaches `cloud_user_email` and `cloud_user_id` via `before_send` and calls `posthog.identify(userId, email ? { email } : {})` with `person_profiles: "always"` (`src/hooks/use-telemetry-identity.ts:84-85`, `src/services/telemetry-context.ts:73-74`, `src/services/telemetry.ts:267-270,447`), linking earlier anonymous `canvas_install` events to the user's account (`.agents/skills/telemetry-analytics/references/guide.md:11`). Moreover, client IP addresses are exposed without `ip: false` on `posthog.init()`.
3. **Code Comment Claims Analytics Is *"opt-in by default"* While UI & `canvas_install` Default ON:**
   - `src/routes/app-settings.tsx:165-166` states `// Treat null as true since analytics is opt-in by default` (inverting the definition of opt-in while pre-toggling analytics ON), and `TelemetryConsentBanner` (`src/components/features/analytics/telemetry-consent-banner.tsx:99`) renders its consent checkbox `defaultChecked` after `canvas_install` has already fired (`src/hooks/use-telemetry.ts:78-83`).
4. **`.env.sample` Documents Only `VITE_DO_NOT_TRACK="1"` and Omits Runtime / Automation Controls:**
   - `.env.sample:28` implies `VITE_DO_NOT_TRACK="1"` disables all telemetry, omitting that it is a **build-time** Vite constant ineffective on prebuilt npm/Docker/Desktop frontend artifacts, and omitting `AGENT_CANVAS_DISABLE_TELEMETRY`, `DO_NOT_TRACK`, `OH_TELEMETRY_*`, and `AUTOMATION_POSTHOG_*`.

---

## 7. Zero-Telemetry / Negative-Check Verification

OpenHands is **not** a zero-telemetry codebase (`posthog-js` `1.450.0` is active in `package.json:52` and `src/services/telemetry.ts`). Targeted negative checks confirm the absence of other commercial telemetry, crash-reporting, or OpenTelemetry SDKs, as well as Electron auto-updaters:

- **Sentry / Segment / OpenTelemetry / Bugsnag / Rollbar / Mixpanel / Amplitude / LogRocket / FullStory / Clarity / gtag / Plausible / Matomo Check (Negative — 0 Production SDK Usages):**
  - Command: `rg -n -i "\b(sentry|segment|opentelemetry|otel|bugsnag|rollbar|mixpanel|amplitude|logrocket|fullstory|clarity|gtag|plausible|matomo)\b" src electron scripts docker helm config .github docs package.json`
  - Result: **0 telemetry/crash/OTEL SDK dependencies or runtime callers** (`package.json` contains none). The only non-UI-word matches were synthetic automation fixture text `"Sentry Spike Explainer"` / `source: "sentry"` (`scripts/seed-automation-ux-data.mjs:526-533`) and Cisco DefenseClaw third-party documentation (`docs/DefenseClaw.md:227`); `segment` matched only URL/git path segments, `SegmentedToggle`, and `--oh-segment` CSS tokens.
- **Electron Auto-Updater Check (Negative — 0 Matches):**
  - Command: `rg -n "(autoUpdater|electron-updater)" src electron`
  - Result: **0 matches** (exit code 1). Electron does not include `electron-updater`; update checks occur only in the web UI via `fetchLatestAgentCanvasVersion()` (`src/api/agent-canvas-updates.ts:9-27`).
- **Privacy Policy Docs Check (Negative — 0 Files):**
  - File search for `*PRIVACY*` across repo root and `docs/` (depth 2) returned **0 files**.
- **Prompt / Source-Code Exfiltration & Unsalted Hash Stitching Check (Negative):**
  - Audited all `trackEvent`, `trackError`, `useTracking`, and `telemetry.ts` callers (`rg -n "(trackSettingsSaved|trackCloudDeviceAuthorization|trackCloudConversationReady|trackError\(|trackEvent\()" src`): no prompt contents, file contents, or unsalted hashes of git remotes/paths/emails are sent over telemetry.

---

```json
{
  "slug": "openhands",
  "platform": "OpenHands",
  "group": "B",
  "repo": "OpenHands/OpenHands",
  "sha": "adf71d626b81a0db00902e0d3edf2b00d2cf917c",
  "commit_date": "2026-11-24T23:20:32-05:00",
  "license": "MIT",
  "status": "Active (All-Hands-AI/OpenHands redirects here)",
  "default_state": "ON pre-consent in browser and ON by default for local Python backends: fires canvas_install before consent on first mount (src/hooks/use-telemetry.ts:78-83), initializes PostHog on load unless DNT is set (telemetry-provider.tsx:311-316), pre-checks consent banner (defaultChecked), treats null consent as true in Settings, locks Cloud backends ON (cookie mode unconditionally forces getTelemetryConsent() === 'granted'), and passes PostHog keys to local agent-server and automation backends.",
  "opt_out_controls": [
    "Consent banner checkbox (pre-checked) and Settings -> App toggle (disabled & forced ON on Cloud backends)",
    "VITE_DO_NOT_TRACK=1 (build-time Vite constant for source builds; also stops backend key injection in launchers/Docker)",
    "AGENT_CANVAS_DISABLE_TELEMETRY=1 or --disable-telemetry on scripts/static-server.mjs (undocumented; injects window.__AGENT_CANVAS_DO_NOT_TRACK__=true)",
    "DO_NOT_TRACK=1 (stops SSR, feedback submission, and agent-server telemetry; ignored by automation backend and prebuilt SPA)",
    "Browser navigator.doNotTrack === '1' / window.doNotTrack === '1' (stops PostHog SDK init and install event)",
    "configureTelemetry(false) / analytics={false} on AgentServerUIProviders (library consumers)",
    "clearTelemetryData() exported API (no UI caller exists)"
  ],
  "do_not_track": "Partially supported: navigator.doNotTrack=1 stops PostHog SDK init and canvas_install, and DO_NOT_TRACK=1 stops agent-server/SSR/feedback, but prebuilt browser bundles require undocumented AGENT_CANVAS_DISABLE_TELEMETRY=1, dev-with-automation.mjs and docker/entrypoint.sh ignore DO_NOT_TRACK=1 for AUTOMATION_POSTHOG_API_KEY, and submitPrivateFeedback() omits navigator.doNotTrack.",
  "opt_out_bypasses": [
    "Pre-consent canvas_install fires on first mount before consent UI choice (src/services/telemetry.ts:759-801)",
    "posthog.init() runs on every mount even when consent is 'pending' or 'denied' unless DNT is enabled, fetching PostHog remote config/flags via z.openhands.dev (src/components/providers/telemetry-provider.tsx:311-316, src/services/telemetry.ts:320-321)",
    "Cookie-auth Cloud mode unconditionally returns 'granted' in getTelemetryConsent(), overriding localStorage denial and locking Settings toggle ON (src/services/telemetry.ts:495-497, src/routes/app-settings.tsx:112-114,217-232)",
    "authenticateWithMainAppCookie() sends POST /api/analytics/events (canvas_authenticated) without checking consent or DNT (src/api/main-app-auth.ts:32-56)",
    "submitPrivateFeedback() (#19272) intentionally bypasses telemetry consent and fails to check navigator.doNotTrack (src/services/telemetry.ts:955-965)",
    "DO_NOT_TRACK=1 is ignored by buildAutomationTelemetryEnv() in scripts/dev-with-automation.mjs:988 and docker/entrypoint.sh:462",
    "Settings npm update check (registry.npmjs.org) and Google Fonts CSS @import (fonts.googleapis.com) have no opt-out toggle"
  ],
  "persistent_ids": [
    "PostHog localStorage persistence ('agent-canvas' -> distinct_id & $device_id), seeded pre-consent and joined to Cloud userId + email via posthog.identify() (person_profiles: 'always')",
    "Website handoff bootstrap oh_ph_handoff / distinct_id / session_id URL params + UTM attribution bridging openhands.dev -> Agent Canvas",
    "PostHog distinct_id forwarded to local agent-server as user_id on conversation creation and to local automation via X-OpenHands-Telemetry-Distinct-Id header and frontend_distinct_id body",
    "Cloud account context properties (cloud_user_id, cloud_user_email, cloud_org_id) injected via before_send",
    "DigitalOcean MARS deviceId (randomUUID in mars-credentials.json sent only as X-Device-UUID to api.digitalocean.com when MARS is used)"
  ],
  "hashing": "No telemetry hashing or unsalted cross-provider ID stitching via hashes; local-only truncated SHA-256 token fingerprint (sha256(token)[:16]) in scripts/mars-credentials.mjs:74-76 for DigitalOcean PAT deduplication.",
  "external_auth_reads": "No cross-tool secret exfiltration or telemetry transmission of external auth files. Local-backend-only ACP login status probe (src/api/acp-service/acp-service.api.ts:75-118) runs 'claude auth status --json', 'codex login status', 'test -f \"$HOME/.gemini/oauth_creds.json\"', and 'grep -qs '\"type\"' \"$HOME/.pi/agent/auth.json\"' via local bash API solely to classify UI status ('authenticated'|'unauthenticated'|'unknown') on onboarding/settings. User-initiated DigitalOcean MARS stores DO tokens in OS Keychain or ~/.openhands/agent-canvas/digitalocean/secrets.json.",
  "telemetry_endpoints": [
    "https://z.openhands.dev (default PostHog ad-blocker-bypass reverse proxy for client SDK events, flags/config, and #19272 surveys)",
    "https://us.i.posthog.com (default PostHog US direct ingest host passed to local/Docker agent-server and automation backends)",
    "https://us.posthog.com (default PostHog UI host)",
    "/api/analytics/events (same-origin Cloud cookie-mode canvas_authenticated beacon)",
    "https://registry.npmjs.org/@openhands/agent-canvas/latest (Settings update check when not locked to Cloud)",
    "https://fonts.googleapis.com/css2 (Outfit & IBM Plex Mono stylesheet loads)",
    "/api/automation/v1/telemetry/consent (local automation sidecar consent & distinct_id sync)"
  ],
  "ip_geo_handling": "No IP suppression on main PostHog SDK init (telemetry.ts:438-455 omits ip:false and property_blacklist) and routes through OpenHands reverse proxy z.openhands.dev, exposing client IP and GeoIP to both OpenHands and PostHog. Only manual submitPrivateFeedback() (#19272) sets ip:false in the JSON payload.",
  "custom_model_handling": "Verbatim unredacted LLM model strings sent on settings save: settings-form.tsx:44-45 passes agentLlm.model raw to trackSettingsSaved({ llmModel }) -> LLM_MODEL (use-tracking.ts:280). Every useTracking/trackError event also includes verbatim current_url (window.location.href).",
  "docs_discrepancies": [
    "Zero privacy/telemetry documentation in README.md, README.windows.md, docs/, or bin/agent-canvas.mjs --help (no PRIVACY file; omits PostHog, pre-consent canvas_install, z.openhands.dev proxy, npm update check, Google Fonts, and AGENT_CANVAS_DISABLE_TELEMETRY=1)",
    "Telemetry banner claims 'No personal information is collected' (src/i18n/translation.json:23997) while collecting cloud_user_email/cloud_user_id via posthog.identify() + before_send (with person_profiles:'always') and exposing client IP",
    "app-settings.tsx:165 comment claims 'analytics is opt-in by default' while setting null ?? true, pre-checking the banner checkbox, and firing canvas_install before consent",
    ".env.sample:28 implies VITE_DO_NOT_TRACK=1 disables all telemetry without noting it fails at runtime on prebuilt npm/Docker/Desktop bundles"
  ],
  "privacy_rating": "D",
  "one_line_verdict": "Ships pre-consent install beacons and pre-checked analytics routed through an ad-blocker-bypass PostHog proxy (z.openhands.dev) without IP suppression, leaks verbatim custom model names and full URLs, fractures opt-out flags across prebuilt bundles and backends, locks Cloud telemetry ON, and claims no PII is collected despite identifying users by email."
}
```
