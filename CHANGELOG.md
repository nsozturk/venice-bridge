# Changelog

All notable changes to venice-bridge. Versions follow [Semantic Versioning](https://semver.org/).
Releases: <https://github.com/nsozturk/venice-bridge/releases> · PyPI: <https://pypi.org/project/venice-bridge/>

## 0.4.0 — 2026-11-28

### Added
- **Image tools from the Venice web app** — generation straight through `/api/inference/image`
  (`/v1/images/generations`, `/v1/image/generate`, `n`/`variants` 1–4, style presets, negative prompt, steps, cfg,
  seed, aspect ratio, resolution, optional prompt enhance), image edit / combine with optional mask
  (`/v1/images/edits`, `/v1/image/edit`, Venice `multi-edit`), **upscale** (`/v1/image/upscale`, Venice `enhance`,
  scale 1–4), **background removal** (`/v1/image/background-remove`), and `GET /v1/image/styles`. `/v1/models`
  lists image and edit models. Only 0-credit models are used unless `VENICE_ALLOW_CREDIT_IMAGES=1`, and
  `acceptCreditFallback` is always sent as `false`.
- `/stats` endpoint counters: `image_edits`, `image_upscales`, `image_background_removals`.

- **Adapter `/venice-api/...` routes** and **`claude-venice --venice-api` / `--claude`** — run Claude Code on the
  official Venice API or the official Anthropic API with your own key (both metered; opt-in only).

### Security
- `claude-venice <backend> --models` reads API keys from a temporary `0600` header file (`curl -H @file`) instead of
  the command line, where `ps` could see them.
- Multipart requests to Venice send text fields with `curl --form-string`; with `-F`, a prompt starting with `@` or
  `<` would have made curl upload a local file.

## 0.3.0 — 2026-11-22

### Added
- **Embeddings endpoint (`POST /v1/embeddings`)** — OpenAI-compatible embeddings backed by Venice's `text-embedding-bge-m3` (`1024`-dim float vectors).
- **Thinking mode on tool-calling turns** — models requested with `:think` suffix or `VENICE_REASONING=1` now preserve reasoning output (`reasoning_content`) on both streaming and non-streaming tool-call turns.
- **Anonymous web search mode** — sets `webSearchProviderMode: "anon"` when web search is enabled.
- **Multi-backend `claude-venice` launcher** — added `--unquant`, `--zai`, and `--go` / `--opencode-go` flags with isolated `0600` local key files (`~/.config/venice-bridge/*-api-key.txt`) and `--models` discovery for each backend.

### Fixed
- **High-concurrency `ConnectionResetError(54)` / 502 fix** — raised `ThreadingHTTPServer.request_queue_size` from Python's default `5` to `512` in both `venice_bridge.py` and `venice_adapters.py`, added exponential-backoff retries (5 attempts) for transient socket resets in `venice_adapters._upstream()`, and cleanly caught `(BrokenPipeError, ConnectionResetError)` on aborted client SSE streams.

## 0.2.6 — 2026-11-21

### Fixed

- **Double-encoded `Bash.command` with unescaped inner double quotes.** Moved double-encoded `{"command": "..."}` unwrapping in `_fit_arguments` to run after `wrap_key` / `ARG_ALIASES` and added a regex fallback when unescaped inner double quotes cause strict JSON/lenient parsing to fail (fixes residual `Exit code 127: command not found: command:` errors).
- **Control characters in `Bash.command` from JSON-decoded regex escapes.** When models emit `\b` or `\f` instead of `\\b` / `\\f` inside JSON `Bash` arguments, JSON decodes them to ASCII backspace (`\x08`) / formfeed (`\x0c`), which Claude Code's Zod validator rejects (`"command contains control characters that would be hidden in the approval dialog"`). `_fit_arguments` now restores `\x08` → `\b` and `\x0c` → `\f` and strips any other stray C0 control characters (preserving `\n`, `\r`, `\t`).

## 0.2.5 — 2026-11-21

### Fixed

- **Explicit `webEnabled: false`, `webScrapeEnabled: false`, `xSearchEnabled: false` by default.** Previously omitting these keys caused Venice `/api/inference/chat` to default to `auto` web search and inject unrelated web articles into coding turns unless `:web` / `:xsearch` was requested. Setting them explicitly to `false` eliminates web-search hijacking on tool-calling turns.
- **Claude Code `2.1.329` tool-schema parity (`ARG_ALIASES` & `SAME_TOOL`).** Added Claude Code's snake_case argument names (`file_path`, `old_string`, `new_string`, `replace_all`, `run_in_background`, `skill`) to `ARG_ALIASES` (fixing dropped `path`/`filePath` arguments on `Read`/`Edit`/`Write`) and added `"agent"` to `SAME_TOOL["task"]`.
- **New `_route_call` aliases for hallucinated and deferred tools:**
  - `SQL_ALIASES` (`sql_query`, `sqlite_query`, `db_query`, `execute_sql`, `run_sql`, `query_db`, `sql`, etc.) → read-only `sqlite3 -readonly <db> <sql>` via `Bash`/`shell` when both database path and SQL query are provided.
  - `DATE_ALIASES` (`get_current_date`, `get_date`, `current_date`, `get_time`, `current_time`, `get_current_time`, `date`) → `date -u '+%Y-%m-%d %H:%M:%S UTC'` via `Bash`/`shell`.
  - Deferred `grep`/`search` and `glob`/`find_files` (when `Grep`/`Glob` are deferred and absent from declared tools) → bounded `grep -rn ... | head -n 50` and `find ... | head -n 50` via `Bash`/`shell`.
- **Double-encoded `command` unwrap in `_fit_arguments`.** Unwraps `{"command": '{"command": "..."}'}` and salvaged `{"content": '{"command": "..."}'}` payloads so shells no longer fail with `command not found: command:`.
- **`TOOL_REMINDER` cleanup.** Removed the literal `"example_tool"` placeholder from the reminder prompt so models stop hallucinating `example_tool` calls.
- **`claude-venice` `bypassPermissions` on resumed sessions.** The launcher now passes `--permission-mode bypassPermissions` alongside `--dangerously-skip-permissions` by default (and pairs `--dangerously-skip-permissions` whenever `--permission-mode bypassPermissions` is explicitly passed), so `--resume <id>` sessions originally saved in `default` mode do not fall back to interactive permission prompts.

## 0.2.4 — 2026-11-21

### Added

- **Proactive Clerk JWT refresh.** The session token is refreshed in the background when fewer than
  `VENICE_BRIDGE_PROACTIVE_REFRESH_S` seconds (default 30) remain, instead of on the request path
  (7.5% of measured upstream opens had waited ≥ 0.5 s for auth).
- **`GET /stats`** (JSON, or `?format=html`): live request, model, TTFT, retry, cache, auth, pool and endpoint counters. Counts only; no prompts, cookies or tokens.
- **`tool_choice`** (`auto`, `none`, `required`, named) and **`parallel_tool_calls: false`** on
  `/v1/chat/completions`; Anthropic `tool_choice` and `disable_parallel_tool_use` and Responses
  `parallel_tool_calls` are forwarded by the adapters.
- **`response_format`** (`json_object`, `json_schema`) and Responses **`text.format`**: the model is told to answer in JSON and fenced or padded JSON is cleaned up.
- **`stop` / `stop_sequences`** (no-tools path): output stops before the first match, safe across stream chunks; the Anthropic adapter reports `stop_reason: "stop_sequence"`.
- **Anthropic thinking blocks**: reasoning is streamed as `thinking` content blocks on `/v1/messages`.
- **Opt-in multi-account cookie pool** (`VENICE_BRIDGE_COOKIE_POOL`, a 0600 file outside the repo, one cookie per line; see `cookie-pool.example.txt`). When the active account hits its quota (HTTP 429) the bridge moves to the next account until the reset. Unset = single-account behavior, unchanged. Accounts appear only as `acct#N` in logs and `/stats`.
- **Multimodal image & document attachments** across OpenAI `/v1/chat/completions` (`image_url`, `file`), Anthropic `/v1/messages` (`image`, `document` base64), and OpenAI `/v1/responses` (`input_image`, `input_file`). Requests carrying attachments are sent as `multipart/form-data` to `/api/inference/chat/agents/ad-hoc`; text-only requests stay unchanged on the plain chat route. Enforces a 17-type MIME allowlist (png/jpg/gif/webp + txt/md/csv/rtf/json/yaml/xml/pdf/docx/xls/xlsx), rejects remote `http(s)://` URLs with HTTP 400 (`remote URLs not supported, send base64`), enforces per-file (10 MB) and total (20 MB) size limits with HTTP 413 (`VENICE_BRIDGE_MAX_ATTACHMENT_BYTES`, `VENICE_BRIDGE_MAX_TOTAL_ATTACHMENT_BYTES`), skips response cache for multimodal turns, and never logs raw bytes/base64.
  - *Known limitation:* Because attachments require `/api/inference/chat/agents/ad-hoc`, combining an image attachment with client tool calls in the same turn can collide with Venice's own server tools (measured: `2/5` `bash` tool calls survived on `deepseek-v4-1-flash-uncensored` with an attached PNG; text-only tool turns are unaffected). Only attachments sent after the last assistant turn are uploaded; older ones become a placeholder, so the rest of the conversation returns to the plain chat route.
- **OpenAI-compatible Speech-to-Text (`POST /v1/audio/transcriptions`)** backed by Venice `/api/inference/asr` (`nvidia/parakeet-tdt-0.6b-v3` default; `json`, `text`, and `verbose_json` formats; mp3/wav/m4a/webm/ogg/flac allowlist; 25 MB cap via `VENICE_BRIDGE_MAX_AUDIO_BYTES` → HTTP 413; 0 chat/image quota spent).
- **OpenAI-compatible Image Generation (`POST /v1/images/generations`)** backed by Venice in-chat `veniceai:image_generation.image_chunk` (`n=1`, `b64_json` default or `url` as `data:` URI, no file hosting; decrements both `chat` and `image` quota by 1). Both endpoints add count-only counters under `GET /stats` (`endpoints.audio_transcriptions`, `endpoints.image_generations`) and never log or cache raw audio/image bytes.

## 0.2.3 — 2026-11-21

### Added

- **`claude-venice --lean`** launcher flag: disables all MCP servers via `--strict-mcp-config --mcp-config '{"mcpServers":{}}'` to keep the `[TOOLS]` block compact when a project carries `.mcp.json` (default behavior unchanged).

### Fixed

- **Runaway repetition loops cut early.** In tool mode, replies exceeding 40 tool-call envelopes or 150,000
  characters are cut at the 41st envelope / cap and parsed as complete instead of hanging for minutes
  (`VENICE_BRIDGE_LOOP_MAX_CALLS`, `VENICE_BRIDGE_LOOP_MAX_CHARS`).
- **Code written instead of run.** A reply with no tool call that holds a fenced code block while a shell tool
  is declared now goes to the judge, which can ask the model to run the command instead of showing it.
- `bash_tool` is routed to the client's shell tool.

## 0.2.2 — 2026-11-21

### Fixed

- **Bare-string tool calls are no longer dropped.** When a model answers a tool call with a bare
  string, it reaches the bridge as `{"content": "..."}`. If the tool has no `content` parameter and
  exactly one required string parameter, the string now goes into that parameter instead of being
  dropped as `dropped args not in schema: ['content']`.

### Docs

- This changelog, and a "What's new" section on the documentation site.

## 0.2.1 — 2026-11-21

### Fixed

- **Double-nested tool arguments are unwrapped**: `{"arguments": {...}}`, `{"name": ..., "arguments": {...}}`,
  a JSON string in `arguments`, and `{"input": {...}}` / `{"parameters": {...}}`, up to three levels and
  only when the tool's schema has no field of that name.
- The format reminder tells the model to put parameters directly in `arguments` and to use a heredoc for
  multi-line SQL or Python.

### Added

- Installable from PyPI: `pipx install venice-bridge` gives the `venice-bridge`, `venice-adapters` and
  `claude-venice` commands.

## 0.2.0 — 2026-11-21

### Added

- `venice_adapters.py` (127.0.0.1:18892): Anthropic Messages and OpenAI Responses in front of the bridge,
  so Claude Code and Codex CLI work. Standard library only.
- `claude-venice`: runs Claude Code against the adapter in an isolated profile.
- Local models by prefix (`ollama/...`, `lmstudio/...`, others via `backends.json`).
- `venice-auto`: Venice normally, a local model while the daily web quota is used up.
- Opt-in Venice web search (`:web`, `:xsearch`) and reasoning on no-tool requests (`:think`).
- `temperature` and `top_p` forwarded; credit-only and e2ee models hidden from `/v1/models`;
  in-memory cache for byte-identical repeat requests.

### Fixed

- A local model asked through the adapter never falls back to Venice when its backend is down.
- `reasoning_effort` no longer turns reasoning on for requests that declare tools.
- `venice-bridge/venice-auto` resolves like `venice-auto`.

## 0.1.0

- First open-source release: OpenAI-compatible bridge for the venice.ai web plan with emulated tool calls,
  plain chat route by default, native call reader and tool-name / argument routing.
