# Tool routing — Claude Code / OpenCode / Codex → Venice

How a client's tools reach the Venice web model and how the model's tool calls are mapped back.
Source of truth is the code (`venice_bridge.py`, `venice_adapters.py`); this page describes v0.4.0.

## TL;DR

No client tool is dropped. The adapter forwards **every** well-formed tool 1:1; the bridge describes **all** of them to the model. Venice never receives a `tools` array — the bridge
puts a `[TOOLS]` block in the system prompt (`openai_to_venice_chat` → `_build_tool_instructions`,
`venice_bridge.py`) and the model replies with `{"tool_calls":[…]}`, which the bridge parses and
routes. The two places behaviour varies are: **how much schema detail** each tool gets (an 78 KB budget),
and **how a wrong tool name / argument name the model invents is mapped** to the real client tool.

## 1. Forwarding — adapter  (Anthropic → OpenAI), `messages_to_chat` (`venice_adapters.py`)

| Incoming (Claude Code) | Mapped to | Status |
|---|---|---|
| `name` | `function.name` | verbatim |
| `description` | `function.description` | verbatim |
| `input_schema` | `function.parameters` | verbatim |
| `tools` array | all of them | forwarded; only a tool with **no name** or a non-dict `input_schema` is skipped |
| `tool_choice` | — | not carried (the bridge ignores it anyway) |
| `max_tokens` | `max_tokens` | forwarded when an int |

The bridge does **not** send tools to Venice: `payload["tools"]=[]`, `payload["clientTools"]=[]`
(`venice_bridge.py`). Tools travel as text in the system prompt instead.

## 2. Schema-detail tiers — `_build_tool_instructions` (`venice_bridge.py`), budget `TOOL_BLOCK_BUDGET=78_000`

| Tier | Which tools | Detail given to the model |
|---|---|---|
| **A** (never cut) | tools already called earlier in the conversation · names **without an underscore** (the CC built-ins: `Bash Read Write Edit Glob Grep Task WebFetch WebSearch TodoWrite …`) · `CORE_TOOL_NAMES` = `apply_patch` | **full compact argument schema** |
| **B** (underscore names → mostly `mcp__*`) | everything else | one-liner → (over budget) shorter description → name-only, announced on the last line |

Floor guarantee: header + all of Tier A in full + **every** tool at least by name — nothing disappears. So a plain `claude-venice` with no MCP servers sends essentially **all** tools with full
schema; the budget only degrades Tier B when many `mcp__*` tools are present. The per-request log line
`tool list: A=… B=… names only=… block=… of 78000 … full schema for: …` reports exactly what
each request sent.

## 3. Name routing — model invents a wrong name → real client tool, `TOOL_ALIASES` / `_route_call` (`venice_bridge.py`)

| Client tool | Names the model uses that map to it |
|---|---|
| `bash` | shell, sh, zsh, terminal, execute, exec, run, run_command, run_shell_command, run_terminal_cmd, python, code_interpreter, calculator, container.exec, exec_command, local_shell … |
| `read` | cat, read_file, view, view_file, open, open_file, get_file, head, tail, get_attachment, view_media, hydrate_content |
| `write` | write_file, create_file, save_file, write_to_file, replace_file |
| `edit` | edit_file, str_replace, str_replace_editor, apply_edit, update_file, search_replace |
| `grep` | search, rg, ripgrep, search_files, grep_search, find_in_files, codebase_search, content_search … |
| `glob` | find, find_files, file_search, search_file_names |
| `websearch` | web_search, search_web, google, google_search, bing_search, brave_search … |
| `webfetch` | web_scrape, web_fetch, fetch, fetch_url, browse, open_url, http_get, scrape |
| `task` | agent, sub_agent, spawn_agent, launch_agent, dispatch_agent, delegate, new_task, create_subagent … |
| `skill` | use_skill, load_skill, run_skill, invoke_skill |
| `question` | askuserquestion, ask_user, ask_followup_question, ask, clarify … |
| listing (`ls`, `list_files`, `dir`, `tree` …) | to a declared `list` tool, else `bash: ls -la <path>` |
| SQL (`sql_query`, `sqlite_query`, `db_query`, `execute_sql`, `run_sql`, `sql` …) | `bash: sqlite3 -readonly <db> <sql>` (when both database path and SQL query are given) |
| date/time (`get_current_date`, `get_date`, `current_date`, `get_time`, `date` …) | `bash: date -u '+%Y-%m-%d %H:%M:%S UTC'` |
| deferred `grep` / `glob` (when absent from declared tools) | `bash: grep -rn -- <pat> <where> \| head -n 50` / `find <where> -name <pat> \| head -n 50` |

Mapping only fires when the client **does not** declare that exact name; a declared name always wins and
is left alone. `SAME_TOOL` resolves the same role across client versions (CLI `bash` vs desktop
`shell`, `task` vs `agent`), and `ARG_ALIASES` maps camelCase/generic keys to Claude Code's snake_case
parameters (`file_path`, `old_string`, `new_string`, `replace_all`, `run_in_background`, `skill`).
Unless `:web` or `:xsearch` is requested, `_to_chat_payload` explicitly sends `webEnabled: false`,
`webScrapeEnabled: false`, and `xSearchEnabled: false` so Venice does not inject automatic web-search
results into tool-calling turns.

## 4. Venice's own server tools → client tool, `VENICE_SERVER_TOOLS` (`venice_bridge.py`)

| Venice server tool | Routed to |
|---|---|
| view_media · get_attachment · hydrate_content | `read` (only when the id is a real file path) |
| web_search | `websearch` / `webfetch`; web_scrape · web_fetch → `webfetch` |
| calculator · code_interpreter · python | `bash` |

A call Venice **swallows** (runs itself) arrives with the name only, no arguments, so it cannot be
forwarded — it is asked for again by name, in the right format.

## 5. Argument-name aliasing — `ARG_ALIASES` / `_fit_arguments` (`venice_bridge.py`)

Before renaming, `_fit_arguments` unwraps envelopes the model adds around the real arguments: `{"arguments": {...}}`, `{"name": ..., "arguments": {...}}`, a JSON string in `arguments`, and `{"input": {...}}` / `{"parameters": {...}}`, up to three levels and only when the tool's own schema has no field of that name. Keys already present at the outer level win. A bare string reply (`{"content": "..."}`) goes into the tool's
one required string parameter when the tool has no `content` parameter of its own.

| Expected arg | Names the model may use |
|---|---|
| `filePath` / `path` | path, file, file_path, filename, file_name, target_file, dir, directory, folder, cwd, absolute_path |
| `command` | cmd, script, code, shell_command, input, expression |
| `oldString` / `newString` | old_string/old/old_text/search/find · new_string/new/new_text/replace |
| `content` | contents, text, file_text, data, body |
| `pattern` / `query` | query/regex/glob/q/search/name · q/search/pattern/text/prompt |
| `url` | link, uri, href, address |
| `prompt` | task, instructions, message, request, query |
| `agent` | subagent_type, agent_type, subagent, agent_name, role, mode |
| `limit` · `include` · `replaceAll` · `workdir` | max_results/max/count · glob/file_pattern/globs · replace_all/all/global · cwd/dir/working_directory |

Rule: only an argument the tool **expects but did not receive** is filled, and only from a name the tool
**does not know**.

## 6. Value fixes + expansions — `_coerce_arguments`, `_expand_call`

| Transform | What it does |
|---|---|
| command list → string | Codex `["bash","-lc","…"]` → one command |
| url extraction | pulls `https?://…` out of `prompt`/`query`/`text` |
| agent/description defaults | fills the sub-agent tool's required fields (default `general`) |
| questions normalize | reshapes to OpenCode's question-tool schema |
| MultiEdit / Cline SEARCH-REPLACE diff / read_many_files | one model call → **several** `edit`/`read` calls |

## 7. Deliberately NOT routed, and why

| Thing | Why |
|---|---|
| `patch` · `replace_in_file` · `apply_diff` | their diff/patchText can't be turned into `oldString`/`newString`, so the model gets a clean **"no such tool"** instead of a broken edit |
| a call Venice swallowed | only the name streams back, never the arguments → can't be forwarded |
| arguments not in the tool's schema | CC/OpenCode tools are `additionalProperties:false`, so extras are **dropped** (log `dropped args not in schema`, ) |
| a Venice media call whose id isn't a file | no file-read equivalent → not mapped (returns None, ) |

---

Verify against the code before relying on any row — the alias tables change as new clients appear. The
per-request `tool list: …` stderr line is the ground truth for what a given request actually sent.
