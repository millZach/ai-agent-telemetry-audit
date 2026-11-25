# Privacy & Telemetry Forensic Audit: Aider (`Aider-AI/aider`)

## Header Metadata

- **Slug:** `aider`
- **Platform:** `Aider`
- **Group:** `B`
- **Repository:** `Aider-AI/aider` (`https://github.com/Aider-AI/aider`)
- **License:** `Apache-2.0`
- **Status:** `Active`
- **Branch:** `main`
- **Commit SHA:** `5dc9490bb35f9729ef2c95d00a19ccd30c26339c`
- **Commit Date:** `2026-05-22T07:02:20-07:00`
- **Commit Subject:** `Merge pull request #5173 from Cyberfilo/feat/anthropic_models-list-add-opus-4-5`

---

## 1. Default State & Opt-Out Mechanism

Aider combines a **probabilistic 10% prompted opt-in PostHog telemetry pipeline** with **default-ON outbound update checks and model-pricing metadata fetches**, plus **unconditional on-disk UUID creation** on every run.

### Default State

1. **PostHog Analytics (`aider/analytics.py`, `aider/main.py`):**
   - `args.analytics` defaults to `None` (`aider/args.py:569-573`), representing neither explicitly enabled nor explicitly disabled.
   - On every startup, `aider/main.py:636-641` instantiates `Analytics(...)`, whose constructor (`aider/analytics.py:73-86`) unconditionally calls `self.get_or_create_uuid()` (`aider/analytics.py:81,147-153`), generating and persisting a UUIDv4 in `~/.aider/analytics.json` (`aider/analytics.py:139`).
   - `aider/main.py:642-662` then calls `analytics.need_to_ask(args.analytics)` (`aider/analytics.py:119-135`):
     - **90% of fresh installs (`uuid_str[:6] > "199999"`):** Because `PERCENT = 10` (`aider/analytics.py:15`) and `compute_hex_threshold(10)` returns `"199999"` (`aider/analytics.py:27,51-52`), `need_to_ask(None)` returns `False`. `asked_opt_in` remains falsy (`None`/`False`), so `analytics.enable()` (`aider/analytics.py:88-99`) disables PostHog (`self.ph = None`) and **PostHog analytics remains OFF** for that and subsequent runs unless `--analytics` is passed or `analytics.json` is removed.
     - **10% of fresh installs (`uuid_str[:6] <= "199999"`):** `need_to_ask(None)` returns `True` (`aider/analytics.py:135`). Aider prompts: `"Allow collection of anonymous analytics to help improve aider? (Y)es/(N)o [Yes]: "` (`aider/main.py:649-651`, `aider/io.py:841-842`). Answering `"y"` (or pressing Enter / EOF / `--yes-always`) sets `asked_opt_in = True`, persists `analytics.json`, and initializes PostHog (`aider/analytics.py:102-108`) for this and all future runs; answering `"n"` calls `analytics.disable(permanently=True)` (`aider/main.py:654-655`, `aider/analytics.py:110-117`).
   - **Legacy Mixpanel Stub:** `aider/analytics.py:8,55,101,233-237` imports `MixpanelException` and hardcodes `mixpanel_project_token`, with `# self.mp = Mixpanel(mixpanel_project_token)` commented out (`aider/analytics.py:101`). `self.mp` stays `None`.
2. **PyPI Version Check (`aider/versioncheck.py`): ON by default.**
   - `--check-update` defaults to `True` (`aider/args.py:605-609`, `aider/main.py:737-738`). Once every 24 hours (`VERSION_CHECK_FNAME`, `aider/versioncheck.py:12,65-72`), startup sends an unauthenticated GET to `https://pypi.org/pypi/aider-chat/json` (`aider/versioncheck.py:78`).
3. **Model Pricing & OpenRouter Catalog Fetches (`aider/models.py`, `aider/openrouter.py`): ON by default (when model is not in bundled metadata).**
   - When constructing `Model(args.model)` (`aider/main.py:822`, `aider/models.py:348`), `ModelInfoManager.get_model_from_cached_json_db` checks `local_model_metadata` (`aider/resources/model-metadata.json` + `.aider.model.metadata.json`, `aider/main.py:390-403`). If the model is absent from `local_model_metadata`, Aider fetches LiteLLM's pricing JSON from `raw.githubusercontent.com` (`aider/models.py:162-166,203-233`, 24h cache). For `openrouter/` models not found in LiteLLM/local metadata, `OpenRouterModelManager` fetches `https://openrouter.ai/api/v1/models` (`aider/openrouter.py:30,114-120`, 24h cache), and if still missing, `fetch_openrouter_model_info()` scrapes `https://openrouter.ai/<model_route>` (`aider/models.py:263-290`).
4. **Streamlit Browser GUI (`--gui`): Stats Explicitly Disabled.**
   - `launch_gui()` passes `--browser.gatherUsageStats=false` (`aider/main.py:249`) and writes an empty `[general]\nemail = ""` to Streamlit's `credentials.toml` (`aider/main.py:217-230,242`) to suppress Streamlit's email prompt and built-in telemetry.

### Opt-Out Controls & Precedence

All CLI flags accept YAML config keys (`.aider.conf.yml` in CWD, git root, or `~/.aider.conf.yml`) and `AIDER_*` environment variables via `configargparse` (`aider/args.py:36-42`, `aider/main.py:464-478`):

- **`--no-analytics` / `AIDER_ANALYTICS=false` / `analytics: false`** (`aider/args.py:569-573`): Skips the opt-in prompt and skips `analytics.enable()` (`aider/main.py:642-662`), keeping `self.ph = None`.
- **`--analytics-disable` / `AIDER_ANALYTICS_DISABLE=true` / `analytics-disable: true`** (`aider/args.py:580-584`, `aider/main.py:515-517,636-641`): Permanently disables PostHog by writing `asked_opt_in: true` and `permanently_disable: true` to `~/.aider/analytics.json` (`aider/analytics.py:114-117`). Note that `--analytics-disable` does **not** exit after printing `"Analytics have been permanently disabled."` (`aider/main.py:515-517`); execution continues normally.
- **`--analytics-posthog-host` / `--analytics-posthog-project-api-key`** (`aider/args.py:586-594`, `aider/analytics.py:77-83,102-104`): Redirects PostHog output to a self-hosted/custom instance when opted in.
- **`--analytics-log <file>`** (`aider/args.py:575-578`, `aider/analytics.py:242-254`): Logs events locally as JSONL. Combined with `--no-analytics`, logs locally with no outbound PostHog traffic.
- **`--no-check-update` / `AIDER_CHECK_UPDATE=false` / `check-update: false`** (`aider/args.py:605-609`, `aider/main.py:737-738`): Disables the 24h PyPI version check.
- **`--no-show-release-notes` / `AIDER_SHOW_RELEASE_NOTES=false`** (`aider/args.py:611-615`, `aider/main.py:1099-1109`): Suppresses the first-run release notes prompt/browser open (`https://aider.chat/HISTORY.html#release-notes`).
- **`--no-detect-urls` / `AIDER_DETECT_URLS=false`** (`aider/args.py:843-847`, `aider/coders/base_coder.py:964-984`): Prevents prompting to scrape URLs found in chat input.
- **`DO_NOT_TRACK`:** **Not supported** (`0` references in `aider/`).

### Opt-Out Bypasses & Failure Modes

1. **Silent Auto-Opt-In via `--yes-always` / Enter / EOF (`aider/main.py:649-658`, `aider/io.py:807-891`):**
   - `io.confirm_ask("Allow collection of anonymous analytics to help improve aider?")` (`aider/main.py:649-651`) leaves `default="y"` and `explicit_yes_required=False` (`aider/io.py:810,812`).
   - In `aider/io.py:866-867`: `if self.yes is True: res = "n" if explicit_yes_required else "y"`. Consequently, if a user passes `--yes-always` (or `AIDER_YES_ALWAYS=true` / `yes-always: true`) without also passing `--no-analytics`, and their fresh UUID lands in the 10% bucket, Aider **silently opts them into PostHog telemetry without any interactive confirmation** and persists `asked_opt_in: true` for future runs.
   - Pressing Enter on an empty prompt or sending `EOF` (`Ctrl+D`) likewise accepts `default="y"` (`aider/io.py:884-891`). Additionally, `aider/main.py:546-547` sets `args.yes_always = True` whenever `return_coder=True` (used by `aider/gui.py:71`).
2. **Unconditional On-Disk UUID Creation When Disabled (`aider/main.py:515-516,636`, `aider/analytics.py:81,147-153`):**
   - Even when `--no-analytics`, `AIDER_ANALYTICS=false`, or `--analytics-disable` is set, `Analytics.__init__()` creates `~/.aider/analytics.json` with a persistent `uuid4` before opt-in or disable checks execute (though `self.ph` remains `None` so it is not transmitted).
3. **Update & Model Metadata Fetches Ignore `--no-analytics`:**
   - `--no-analytics` and `--analytics-disable` do not disable PyPI version checks (`aider/versioncheck.py:78`), LiteLLM pricing JSON fetches (`aider/models.py:208`), or OpenRouter catalog/scraping fetches (`aider/openrouter.py:116`, `aider/models.py:289`).

---

## 2. External Auth / Local Non-Workspace File Reads

**Clean on third-party agent auth harvesting:** Aider does **not** search for or read credentials from `~/.claude`, `~/.codex`, `~/.cursor`, `~/.ssh`, `~/.aws`, or `~/.config/github-copilot/apps.json` (documented only as manual copy-paste instructions in `aider/website/docs/llms/github.md:36-50`).

Verified local non-workspace file reads and writes across `aider/`:

- **Aider Analytics State:** `Path.home() / ".aider" / "analytics.json"` (`aider/analytics.py:139`) — read/written unconditionally in `Analytics.__init__()` (`aider/analytics.py:81,155-184`).
- **Aider OAuth Key Store:** `Path.home() / ".aider" / "oauth-keys.env"` (`aider/main.py:370-376`, `aider/onboarding.py:363-367`) — read via `load_dotenv` at startup (`aider/main.py:370-382`) and appended when user completes OpenRouter OAuth PKCE onboarding.
- **Aider Config, Dotenv & Model Overrides:** `~/.aider.conf.yml` (`aider/main.py:476`), `~/.env` (`aider/main.py:307,363-367`), `~/.aider.model.settings.yml` (`aider/main.py:307,336-338`), `~/.aider.model.metadata.json` (`aider/main.py:307,397-399`).
- **Aider Install Tracking & Caches:** `~/.aider/installs.json` (`aider/main.py:1185-1216`, stores `(__version__, sys.executable)` keys locally), `~/.aider/caches/versioncheck` (`aider/versioncheck.py:12`), `~/.aider/caches/model_prices_and_context_window.json` (`aider/models.py:169-170`), `~/.aider/caches/openrouter_models.json` (`aider/openrouter.py:34-35`), and `~/.aider/caches/help.<version>` (`aider/help.py:93`).
- **Streamlit Credentials (`--gui` only):** `Path(get_streamlit_file_path()) / "credentials.toml"` (`aider/main.py:217-230`) — writes `[general]\nemail = ""` if missing so Streamlit skips its onboarding prompt.
- **Local Git Config:** `repo.git.config("--get", "user.name" | "user.email")` (`aider/main.py:132,137`, `aider/repo.py:291`) — used strictly locally for commit attribution/warnings; never transmitted in analytics.
- **GitHub Copilot & AWS Credentials:** Copilot token exchange (`aider/models.py:951-984`) reads **only** `os.environ["GITHUB_COPILOT_TOKEN"]` (never disk files); `AWS_PROFILE` / `AWS_ACCESS_KEY_ID` checks (`aider/models.py:751-765`) inspect `os.environ` and delegate Bedrock auth to `boto3`/`litellm` only when a Bedrock model is configured.

---

## 3. Persistent Device / Install IDs & Hashing

- **Persistent User/Install ID (`uuid` in `~/.aider/analytics.json`):**
  - Generated via `str(uuid.uuid4())` (`aider/analytics.py:152`) on the first run of Aider—regardless of whether analytics is enabled, prompted, or disabled via `--no-analytics` / `--analytics-disable` (`aider/main.py:516,636`, `aider/analytics.py:81,147-153`).
  - Note: The file path is `~/.aider/analytics.json` (`aider/analytics.py:139`), **not** `analytics.uuid` (zero references to `analytics.uuid` exist in the repository).
  - When PostHog is enabled, `self.user_id` is sent as `distinct_id` on every `self.ph.capture(...)` call (`aider/analytics.py:240`), and written as `user_id` in `--analytics-log` (`aider/analytics.py:246`).
  - Also used locally for deterministic 10% sampling (`is_uuid_in_percentage(self.user_id, 10)` checking `uuid_str[:6] <= "199999"`, `aider/analytics.py:30-52,135`).
- **No Hardware/OS Fingerprinting:** Aider does not read MAC addresses, hostnames, usernames, machine-ids, or hardware UUIDs (`rg` confirmed 0 occurrences of `gethostname`, `getnode`, `machine_id`, `/etc/machine-id`, `uuid1`/`uuid3`/`uuid5`, `getuser`, `getlogin`).
- **Hashing:**
  - `hashlib.sha256` is used strictly for OpenRouter OAuth PKCE `S256` `code_challenge` generation (`aider/onboarding.py:166-171`).
  - `hashlib.sha1` (`aider/models.py:1019`) hashes LLM request parameter dictionaries locally for in-memory deduplication/tracking (`chat_completion_call_hashes` in `aider/coders/base_coder.py:380`).
  - No telemetry identifier or repository/user data is hashed with MD5/SHA-1/SHA-256.

---

## 4. Exact Outbound Telemetry / Analytics / Crash / OTEL / Relay / Update Endpoints

| Endpoint / URL | Purpose / Channel | Default State & Trigger | Opt-Out | Source Reference |
| :--- | :--- | :--- | :--- | :--- |
| `https://us.i.posthog.com` | PostHog analytics & `$exception` autocapture (project key `phc_99T7muzafUMMZX15H8XePbMSreEUzahHbtWjy3l5Qbv`) | OFF for 90% of fresh installs; 10% cohort prompted (`[Yes]` default, `--yes-always` auto-opts in); ON if `--analytics` + confirmed | `--no-analytics`, `--analytics-disable`, `AIDER_ANALYTICS=false`; override via `--analytics-posthog-host` / `--analytics-posthog-project-api-key` | `aider/analytics.py:56-57,102-108,239-240`; `aider/main.py:636-662` |
| Mixpanel (token `6da9a43058a5d1b9f3353153921fb04d`) | Legacy Mixpanel analytics (dormant) | **Dormant / unreachable** (`# self.mp = Mixpanel(...)` commented out; `self.mp` is always `None`) | N/A | `aider/analytics.py:8,55,62,101,233-237` |
| `https://pypi.org/pypi/aider-chat/json` | Startup version check (24h cached in `~/.aider/caches/versioncheck`) | **ON by default** (`--check-update` defaults to `True`) | `--no-check-update` / `AIDER_CHECK_UPDATE=false` | `aider/versioncheck.py:12,64-80`; `aider/args.py:605-609`; `aider/main.py:737-738` |
| `https://raw.githubusercontent.com/BerriAI/litellm/main/model_prices_and_context_window.json` | Model pricing & context window cache (24h TTL) | **ON by default** whenever selected model is not in `aider/resources/model-metadata.json` or `.aider.model.metadata.json` | None (only bypassed if cached <24h or model exists in local metadata) | `aider/models.py:162-170,203-247`; `aider/main.py:390-403` |
| `https://openrouter.ai/api/v1/models` | OpenRouter model catalog cache (24h TTL) | **ON by default** when using an `openrouter/` model missing from LiteLLM/local metadata | None (bypassed if cached <24h) | `aider/openrouter.py:30-35,114-120`; `aider/models.py:263-267` |
| `https://openrouter.ai/<model_route>` | HTML scraping fallback for OpenRouter model pricing/context | **ON by default** when `openrouter/<model_route>` is missing from both LiteLLM & OpenRouter `/api/v1/models` caches | None | `aider/models.py:269-290` |
| `https://openrouter.ai/api/v1/auth/key` | Check free vs paid OpenRouter tier | Triggered when `--model` is omitted and `OPENROUTER_API_KEY` is set | Specify `--model` explicitly | `aider/onboarding.py:18-41,53-60` |
| `https://openrouter.ai/auth`, `https://openrouter.ai/api/v1/auth/keys`, `http://localhost:8484-8584/callback/aider` | OpenRouter OAuth PKCE onboarding | Prompted only when no `--model` and no API keys exist (or missing `OPENROUTER_API_KEY` for `openrouter/` model) | Decline prompt or set API key / `--model` | `aider/onboarding.py:79-113,175-186,214-356` |
| `https://api.github.com/copilot_internal/v2/token` | GitHub Copilot OAuth token exchange (`Editor-Version: aider/<ver>`, `Copilot-Integration-Id: vscode-chat`) | Triggered on LLM completion only when `GITHUB_COPILOT_TOKEN` is in `os.environ` | Unset `GITHUB_COPILOT_TOKEN` | `aider/models.py:934-984,1027-1035` |
| `https://aider.chat` (OpenRouter attribution headers) | Sets `OR_SITE_URL="https://aider.chat"` and `OR_APP_NAME="Aider"` in `os.environ` | Set unconditionally at `aider/llm.py` import (used by LiteLLM on OpenRouter requests) | Override env vars | `aider/llm.py:9-13` |
| `https://github.com/Aider-AI/aider/issues/new?...` | Uncaught crash / `/report` pre-filled GitHub issue URL | **User-confirmed browser open only** (never automatic HTTP upload) | Answer `n` at prompt | `aider/report.py:37-92,151`; `aider/commands.py:1555-1567` |
| `https://aider.chat/HISTORY.html#release-notes` (and doc URLs in `aider/urls.py`) | Release notes & help doc links | Prompted via `io.offer_url()` (auto-opened if `--yes-always` is set!) | `--no-show-release-notes` | `aider/main.py:1099-1109`; `aider/urls.py:1-17` |
| `aider.chat` website PostHog (`head_custom.html`) | Website JS PostHog snippet (`phc_99T7muzafUMMZX15H8XePbMSreEUzahHbtWjy3l5Qbv`, `https://us.i.posthog.com`) | Website config currently has `analytics.enabled: false` (gated behind opt-in cookie consent if enabled) | Website `enabled: false` | `aider/website/_config.yml:5-8`; `aider/website/_includes/head_custom.html:83-143` |

*Self-hosted / build-only note:* `scripts/homepage.py:192` (`api.github.com/repos/...`), `aider/scrape.py` (user-initiated `/web` or `--detect-urls`, plus `pypandoc.download_pandoc()` fallback at `aider/scrape.py:223`), and `aider/help.py:139` (`BAAI/bge-small-en-v1.5` via HuggingFace on explicit `/help <question>`) do not run automatically during normal CLI startup.

---

## 5. Exact Data & Properties Collected

When PostHog analytics is active (`self.ph` initialized in `aider/analytics.py:102-108`), the following data is transmitted to `https://us.i.posthog.com` (or custom `--analytics-posthog-host`):

1. **Identifier & Super-Properties (attached to every event):**
   - `distinct_id`: `self.user_id` (UUIDv4 from `~/.aider/analytics.json`, `aider/analytics.py:152,240`).
   - `super_properties` (`get_system_info()`, `aider/analytics.py:107,186-193`):
     - `python_version` (e.g., `sys.version.split()[0]`)
     - `os_platform` (`platform.system()`)
     - `os_release` (`platform.release()`)
     - `machine` (`platform.machine()`)
     - `aider_version` (`__version__`)
2. **IP / GeoIP Handling:**
   - **No IP suppression:** `Posthog(...)` is instantiated without `disable_geoip=True` (`aider/analytics.py:102-108`). Direct HTTPS connections to `https://us.i.posthog.com` expose the client IP address for PostHog server-side GeoIP resolution (`aider/website/docs/legal/privacy.md:21`). PyPI, GitHub raw, and OpenRouter requests also directly expose client IP without an anonymizing proxy.
3. **Custom / Private Model Name Handling (`_redact_model_name`, `aider/analytics.py:195-204`):**
   - Events receiving `main_model` (`cli session`, `model warning`, `message_send`) attach `main_model`, `weak_model`, and `editor_model` via `_redact_model_name()`:
     - Calls `model_info_manager.get_model_from_cached_json_db(model.name)` (`aider/analytics.py:199`).
     - **Verbatim leak vector for locally registered custom models:** Because `get_model_from_cached_json_db()` checks `self.local_model_metadata` first (`aider/models.py:224-226`), **any private/custom model defined in a user's `.aider.model.metadata.json` is considered "known" and sent VERBATIM** (`aider/analytics.py:200-201`).
     - Models absent from both `local_model_metadata` and LiteLLM's cached JSON DB have the suffix after the first `/` replaced with `REDACTED` (`model.name.split("/")[0] + "/REDACTED"`, `aider/analytics.py:202-203`), **still exposing the provider/prefix** (e.g., `ollama/REDACTED`, `azure/REDACTED`, `vllm/REDACTED`), or return `None` (serialized as `"None"`, `aider/analytics.py:204,231`) if there is no `/`.
     - `aider/onboarding.py:135` calls `analytics.event("auto_model_selection", model=model)` bypassing `_redact_model_name()`, though `try_to_select_default_model()` (`aider/onboarding.py:44-76`) only selects from 7 hardcoded public model names.
     - Outside PostHog, unrecognized `openrouter/<model_route>` names are leaked in HTTP GET URL paths to `https://openrouter.ai/<model_route>` (`aider/models.py:284-289`).
4. **Events & Behavioral Profile Fields Collected:**
   - **Lifecycle & Mode:** `launched`, `gui session`, `cli session` (`main_model`, `weak_model`, `editor_model`, `edit_format`), `copy-paste mode`, `exit` (`reason`, e.g., `"Completed main CLI coder.run"`, `"/exit"`, `"Control-C"`, `"Unknown edit format"`, etc.) (`aider/main.py:664-1163`, `aider/commands.py:1057`, `aider/coders/base_coder.py:995`).
   - **Repository & Git Config Metadata:** `repo` (`num_files` = tracked file count, `aider/main.py:930-931`), `no-repo` (`aider/main.py:933`), `auto_commits` (`enabled`, `aider/main.py:970`).
   - **LLM Usage & Cost (`message_send`):** `main_model`, `weak_model`, `editor_model`, `edit_format`, `prompt_tokens`, `completion_tokens`, `total_tokens`, `cost`, `total_cost`, plus `message_send_starting` (`aider/coders/base_coder.py:1420,2113-2122`).
   - **Commands & Features:** `command_run` (`!`), `command_<command>` (e.g., `command_ask`, `command_code`, `command_undo` — command name only, no args; `aider/commands.py:314,323,327`), `interactive help` (`aider/commands.py:1126`), `ai-comments file-add` / `ai-comments execute` (`aider/watch.py:194,210`), `model warning` (`aider/main.py:893`).
   - **Onboarding (`aider/onboarding.py:97-382`):** `auto_model_selection` (`model`), `oauth_flow_initiated`, `oauth_flow_code_received`, `oauth_flow_success`, `oauth_flow_failure`, `oauth_flow_failed` (`provider`, `reason`), `oauth_flow_save_failed` (`provider`, `reason`).
5. **Prompts, Code, File Paths, and Exception / Stack Trace Exposure:**
   - **Prompts & Source Code:** Normal chat messages, user file contents, and repository filenames/paths are **not** sent in `analytics.event(...)` calls.
   - **Exception Autocapture & Verbatim Exception Strings (Leak Surface):**
     - `Posthog(..., enable_exception_autocapture=True)` (`aider/analytics.py:106`) enables the PostHog Python SDK's automatic uncaught-exception capture (`$exception` events with tracebacks/frames).
     - `aider/coders/base_coder.py:1511` emits `self.event("message_send_exception", exception=str(err))` verbatim.
     - `aider/onboarding.py:343,378` emit `reason=server_error` and `reason=str(e)` verbatim (which can include local OS paths like `key_file` in `oauth_flow_save_failed` or socket error strings).
     - Note: All non-numeric event properties are coerced via `str(value)` (`aider/analytics.py:227-231`).

---

## 6. Docs / Privacy Policy vs Code Discrepancies

1. **Wrong Default PostHog Host in Docs (`aider/website/docs/more/analytics.md:113` vs `aider/analytics.py:57`):**
   - `analytics.md:113` states: `Set a custom PostHog host (default is app.posthog.com)`.
   - Actual code (`aider/analytics.py:57`): `posthog_host = "https://us.i.posthog.com"`.
2. **"Opt-In Only" & Sampling Narrative vs `--yes-always` / `[Yes]` Default (`analytics.md:14,53-55,69` vs `aider/main.py:649-658`, `aider/io.py:810-891`):**
   - `analytics.md:14` claims: *"Analytics are only collected if you agree and opt-in."* Meanwhile `analytics.md:53-55` confusingly states *"aider will enable analytics for a random subset of users. Such randomly selected users will be asked if they wish to opt-in to analytics."*
   - In code, the 10% prompt defaults to `default="y"` with `explicit_yes_required=False`. Pressing Enter, sending EOF, or running non-interactively with `--yes-always` / `AIDER_YES_ALWAYS=true` **silently opts the user into PostHog permanently without explicit agreement** (`aider/io.py:866-891`).
3. **Sample Analytics Log Omits System Super-Properties, Exception Autocapture, and IP/GeoIP (`analytics.md:79-82` vs `aider/analytics.py:102-108,186-193,239-254`):**
   - `analytics.md:79-82` tells users to inspect `aider/website/assets/sample-analytics.jsonl` to see what data is collected.
   - However, `sample-analytics.jsonl` is generated from `--analytics-log` (`scripts/update-docs.sh:13`), which only writes `event`, `properties`, `user_id`, and `time` (`aider/analytics.py:242-254`). It omits the `super_properties` (`python_version`, `os_platform`, `os_release`, `machine`, `aider_version`) attached to every PostHog capture (`aider/analytics.py:107`), omits PostHog `enable_exception_autocapture=True` `$exception` stack traces (`aider/analytics.py:106`), and omits PostHog IP/GeoIP enrichment (`disable_geoip` is unset).
4. **Custom Model Redaction Claim vs `.aider.model.metadata.json` (`scripts/my_models.py:89` vs `aider/analytics.py:199-201`, `aider/models.py:224-226`):**
   - `scripts/my_models.py:89` states: `"Aider's analytics only records the names of 'well known' LLMs."`
   - In reality, `_redact_model_name()` sends any model present in `local_model_metadata` (including user-authored `.aider.model.metadata.json`) **verbatim**, and sends `<provider>/REDACTED` (retaining the provider prefix) for other slashed model names (`aider/analytics.py:199-203`).
5. **Privacy Policy vs Analytics Page on Device/IP Data (`privacy.md:21` vs `analytics.md:15-16`):**
   - `aider/website/docs/legal/privacy.md:21` discloses collecting *"IP address, device type, device identifiers, web browser type and version, and operating system version"*, whereas `analytics.md:15-16` claims Aider *"never collects your ... personal info"* without disclosing that CLI analytics transmits OS/machine/Python metadata (`super_properties`) and exposes client IP without `disable_geoip=True`.
6. **Undisclosed Default Outbound Calls & Unconditional UUID Creation (`analytics.md:14,34-40` vs `aider/analytics.py:81`, `aider/versioncheck.py:78`, `aider/models.py:208`):**
   - `analytics.md` does not disclose that `~/.aider/analytics.json` is created with a persistent UUIDv4 even when `--no-analytics` or `--analytics-disable` is used, nor that `--no-analytics` leaves PyPI version checks (`pypi.org`) and GitHub/OpenRouter model metadata fetches active.
7. **Unused Mixpanel Dependency Shipped (`analytics.md:110` vs `requirements/requirements.in:23`, `aider/analytics.py:8,55`):**
   - `analytics.md:110` states Aider uses PostHog, yet `mixpanel==5.0.0` remains a required runtime dependency (`requirements/requirements.in:23`, `requirements.txt:204`) with a hardcoded `mixpanel_project_token` in `aider/analytics.py:55`.

---

## 7. Zero-Telemetry / Negative-Check Verification

Aider is **not** a zero-telemetry repository (it bundles PostHog, PyPI update checks, and model-catalog fetches). Targeted read-only `rg` checks across `aider/` (excluding `aider/website/`) verified the absence of the following specific patterns:

- **Other Telemetry/Crash SDKs & GeoIP Suppression (`0` matches):**
  - Command: `rg -n "DO_NOT_TRACK|analytics\.uuid|sentry|opentelemetry|otel|amplitude|segment|bugsnag|rollbar|datadog|scarf|disable_geoip|anonymize" aider/ --glob "!aider/website/**"`
  - Result: Only matched `last_segment` in `aider/models.py:462-463`; **0 matches** for `DO_NOT_TRACK`, `analytics.uuid`, Sentry, OpenTelemetry, Amplitude, Segment, Bugsnag, Rollbar, Datadog, Scarf, `disable_geoip`, or `anonymize`.
- **Competitor / Host Auth File Harvesting (`0` matches):**
  - Command: `rg -n "\.claude|\.codex|\.cursor|\.copilot|github-copilot|\.aws|\.ssh|\.kube|\.netrc|\.git-credentials" aider/ --glob "!aider/website/**"`
  - Result: Only matched `ModalCursorShapeConfig`/`cursor_position` in `aider/io.py:16,610` and `bedrock/` model names in `aider/resources/model-settings.yml`; **0 reads** of external agent or cloud credential files on disk.
- **Hardware / Machine / Hostname Fingerprinting (`0` matches):**
  - Command: `rg -n "hostname|gethostname|getnode|machine_id|machine-id|/etc/machine-id|mac_address|uuid1|uuid3|uuid5|getuser|getlogin" aider/ --glob "!aider/website/**"`
  - Result: **0 matches** (exit code 1).

---

```json
{
  "slug": "aider",
  "platform": "Aider",
  "group": "B",
  "repo": "Aider-AI/aider",
  "sha": "5dc9490bb35f9729ef2c95d00a19ccd30c26339c",
  "commit_date": "2026-05-22T07:02:20-07:00",
  "license": "Apache-2.0",
  "status": "Active",
  "default_state": "Mixed: PostHog analytics is OFF for 90% of fresh installs while a random 10% cohort is prompted to opt in (with [Yes] default and silent auto-accept under --yes-always/EOF); PyPI version check (--check-update) and GitHub/OpenRouter model metadata fetches are ON by default; a persistent UUIDv4 is created in ~/.aider/analytics.json on every first launch.",
  "opt_out_controls": [
    "--no-analytics / AIDER_ANALYTICS=false / analytics: false (disables PostHog for session)",
    "--analytics-disable / AIDER_ANALYTICS_DISABLE=true / analytics-disable: true (permanently disables PostHog)",
    "--analytics-posthog-host and --analytics-posthog-project-api-key (redirect PostHog to custom instance)",
    "--no-check-update / AIDER_CHECK_UPDATE=false / check-update: false (disables PyPI version check)",
    "--no-show-release-notes / AIDER_SHOW_RELEASE_NOTES=false (disables first-run release notes prompt)",
    "--no-detect-urls / AIDER_DETECT_URLS=false (disables URL scraping prompts)"
  ],
  "do_not_track": false,
  "opt_out_bypasses": [
    "--yes-always / AIDER_YES_ALWAYS=true / yes-always: true (and return_coder=True in main.py:546-547) silently auto-accepts the 10% analytics opt-in prompt (io.confirm_ask has default='y' and explicit_yes_required=False), permanently enabling PostHog without user consent",
    "Empty Enter or EOF (Ctrl+D) at the 10% opt-in prompt defaults to 'y' and enables PostHog (io.py:884-891)",
    "Analytics.__init__() unconditionally generates and writes a persistent UUIDv4 to ~/.aider/analytics.json even when --no-analytics, AIDER_ANALYTICS=false, or --analytics-disable is specified (analytics.py:81,147-153)",
    "--no-analytics and --analytics-disable do not stop PyPI versioncheck (versioncheck.py:78) or GitHub/OpenRouter model metadata fetches (models.py:208,289; openrouter.py:116)"
  ],
  "persistent_ids": [
    "~/.aider/analytics.json -> uuid (random UUIDv4 generated unconditionally on first launch; sent to PostHog as distinct_id when opted in)"
  ],
  "hashing": "No telemetry ID hashing; SHA-256 is used only for OpenRouter OAuth PKCE S256 (onboarding.py:168-170) and SHA-1 only for local LLM request parameter deduplication keys (models.py:1019).",
  "external_auth_reads": "None (0 reads of ~/.claude, ~/.codex, ~/.cursor, ~/.ssh, ~/.aws, or ~/.config/github-copilot/apps.json; reads only Aider's own ~/.aider/* files, ~/.aider.conf.yml, ~/.env, local git user.name/user.email, and GITHUB_COPILOT_TOKEN from os.environ).",
  "telemetry_endpoints": [
    "https://us.i.posthog.com (PostHog capture & $exception autocapture; 10% prompted opt-in / --analytics; project key phc_99T7muzafUMMZX15H8XePbMSreEUzahHbtWjy3l5Qbv)",
    "https://pypi.org/pypi/aider-chat/json (24h cached startup version check; default ON)",
    "https://raw.githubusercontent.com/BerriAI/litellm/main/model_prices_and_context_window.json (24h cached pricing JSON when model missing from local metadata; default ON)",
    "https://openrouter.ai/api/v1/models and https://openrouter.ai/<model_route> (24h cached catalog + HTML scrape fallback for unbundled openrouter/ models)",
    "https://openrouter.ai/api/v1/auth/key, https://openrouter.ai/auth, https://openrouter.ai/api/v1/auth/keys (OpenRouter onboarding/OAuth only)",
    "https://api.github.com/copilot_internal/v2/token (Copilot token exchange only when GITHUB_COPILOT_TOKEN env var is set)",
    "Mixpanel token 6da9a43058a5d1b9f3353153921fb04d present in analytics.py:55 but client instantiation is commented out (dormant)"
  ],
  "ip_geo_handling": "No IP suppression or anonymizing proxy; Posthog() is initialized without disable_geoip=True (analytics.py:102-108), exposing client IP to PostHog (us.i.posthog.com) for GeoIP processing (disclosed in privacy.md:21) as well as to PyPI, GitHub, and OpenRouter.",
  "custom_model_handling": "Verbatim leak vector for locally defined models: _redact_model_name() (analytics.py:195-204) treats any model in local_model_metadata (.aider.model.metadata.json) as known and sends model.name verbatim; unknown '<provider>/<name>' models leak '<provider>/REDACTED'; models without '/' send 'None'; unrecognized openrouter/<route> models are sent in GET paths to openrouter.ai/<route> (models.py:284-289).",
  "docs_discrepancies": [
    "analytics.md:113 states default PostHog host is app.posthog.com, whereas analytics.py:57 uses https://us.i.posthog.com",
    "analytics.md:14 claims analytics are collected only if you agree and opt-in, omitting that --yes-always/Enter/EOF silently auto-opts in the 10% cohort (io.py:866-891)",
    "analytics.md:79-82 presents sample-analytics.jsonl (--analytics-log) as representative, omitting PostHog super_properties (OS/machine/Python/aider versions), enable_exception_autocapture=True $exception traces, and IP/GeoIP collection",
    "scripts/my_models.py:89 claims only 'well known' LLMs are recorded, whereas .aider.model.metadata.json models are sent verbatim and slashed unknown models retain '<provider>/REDACTED'",
    "analytics.md omits unconditional creation of ~/.aider/analytics.json UUID when disabled and omits default-on PyPI/GitHub/OpenRouter fetches unaffected by --no-analytics",
    "analytics.md:110 mentions only PostHog while mixpanel==5.0.0 and a hardcoded Mixpanel token remain in requirements.in and analytics.py:55"
  ],
  "privacy_rating": "D",
  "one_line_verdict": "PostHog is gated behind a 10% prompt with no competitor auth harvesting, but fails clean privacy standards due to silent --yes-always/default-Yes opt-in bypasses, unconditional UUID persistence even when disabled, missing PostHog GeoIP suppression, exception autocapture and verbatim error strings, verbatim leakage of .aider.model.metadata.json custom models, ignored DO_NOT_TRACK, default-on PyPI/catalog requests, and multiple docs discrepancies."
}
```
