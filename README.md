# Personal Agent

> **Language / 语言**: [English](#english) · [中文](#中文)

<a id="english"></a>

A from-scratch Python agent built on the ReAct loop. Three faces, one core:

- **CLI** — terminal REPL with streaming Markdown, tool calls, and a delete-permission gate.
- **Web** — a draggable orange tabby pixel cat (`web/`) that opens a chat bubble; FastAPI + WebSocket bridge in `server/`.
- **Feishu/Lark bot** — long-connection bot in `src/agent/feishu/`; one ConversationState per user.

The agent carries a **markdown-native memory** at `~/assistant-memory/`: per-turn two-stage retrieval picks the right files; a background curator extracts facts from the user message and `[memory note: ...]` markers in the assistant reply.

## Requirements

- Python 3.11+
- [uv](https://docs.astral.sh/uv/)
- One of: `DEEPSEEK_API_KEY` · `GPT_AK` (ByteDance) · `OPENAI_API_KEY`
- Node 18+ (only if you run the web frontend)

## Setup

```bash
uv sync
```

Create `.env`:

```
DEEPSEEK_API_KEY=...        # or GPT_AK=...  or OPENAI_API_KEY=...
```

Auto-detect priority: **DeepSeek → ByteDance → OpenAI**. Force with `USE_DEEPSEEK=true` or `USE_BYTEDANCE=true`.

### Optional env vars

| Variable | Default | Notes |
|---|---|---|
| `OPENAI_MODEL` | `gpt-4o-mini` | OpenAI model |
| `GPT_MODEL` | `gpt-5.2-2025-12-11` | ByteDance model |
| `DEEPSEEK_MODEL_PRO` | `deepseek-v4-pro` | Main responses + curator |
| `DEEPSEEK_MODEL_FLASH` | `deepseek-v4-flash` | Two-stage retrieval |
| `DEEPSEEK_BASE_URL` | `https://api.deepseek.com` | DeepSeek endpoint |
| `OPENAI_MAX_TOKENS` | `4096` | Max completion tokens |
| `ASSISTANT_MEMORY_DIR` | `~/assistant-memory` | Memory store root |
| `LOG_LEVEL` / `LOG_DEBUG` | `DEBUG` / `false` | Logging |
| `LOG_SERVER_PORT` | `9999` | Live debug log server port |
| `FEISHU_APP_ID` / `FEISHU_APP_SECRET` | — | Required for the Feishu bot |

## Run

### CLI

```bash
uv run agent              # REPL
uv run agent -memlog      # also stream retrieval/curator events as orange MEMORY panels
uv run log-server         # (separate terminal) live debug logs
```

`quit` / `exit` / Ctrl+C to leave. Double Ctrl+C force-exits and kills any in-flight curator/digest threads.

### Web (cat pet)

Two terminals:

```bash
# Backend
uv run uvicorn server.main:app --reload --reload-dir server --port 8000

# Frontend
cd web && npm install && npm run dev
```

Open <http://localhost:5173>. The browser shows a draggable pixel cat plus a right-side action panel that streams every tool call and curator memory write in real time.

In web mode the end-of-turn `propose_writes` LLM call is suppressed — only `[memory note: ...]` markers, intercepted mid-stream, fire the curator. Trivial turns cost zero curator tokens. The marker text is stripped before tokens reach the browser.

### Feishu bot

```bash
uv run feishu-bot
```

Long-connection (WebSocket outbound) bot — no public IP or domain required. State is keyed per `open_id`.

## Tools

Defined in `src/agent/tools/`, registered in `tools/registry.py`:

| Tool | Category |
|---|---|
| `read_file`, `list_dir` | Read |
| `write_file`, `str_replace`, `file_rewrite`, `make_dir` | Write |
| `delete_file`, `delete_dir` | Delete (gated) |
| `bash` | Exec (tier-classified) |
| `web_search` | Utility |
| `beautify`, `read_skill`, `check_permissions`, `echo` | Utility |

**Delete gate** (`permissions/gates.py`): on the first delete touching a path, an interactive panel offers *grant for session* / *delete once* / *cancel*. Grants cover all children. Non-TTY → cancel.

**Edit safety**: writes go to a temp file then atomic rename; Python/JSON files get a syntax check after edit; paths are confined to cwd.

## Assistant memory

Layout under `~/assistant-memory/` (override with `ASSISTANT_MEMORY_DIR`):

```
_index.md                       # always-loaded top-level manifest
identity/
  agent.md                      # the agent's "soul"; frontmatter immutable_core is locked
  me.md                         # user identity (user-maintained)
people/<name>.md
preferences/<topic>.md          # food, coffee, gifts, …
life/<aspect>.md                # long-term aspirations, ongoing work
threads/<topic>.md              # open conversational threads, unresolved TODOs
events/<slug>.md                # time-anchored events
context/current.md              # this week's state — always injected
log/YYYY-MM/YYYY-MM-DD.md       # daily session digests
<scope>/_manifest.md            # pipe-delimited hooks for retrieval
```

**Per-turn read** — `assistant_memory/retrieval.py`:
1. `_index.md` + recent turns + user query → Stage 1 (`flash` model) picks scopes.
2. Selected scope manifests → Stage 2 (`flash` model) picks ≤5 files.
3. Selected files + `context/current.md` are injected as a transient system suffix for that turn only — never persisted into history.

`signal_detector.py` extracts entity hints (people/places) from the user message to bias Stage 2.

**Background write** — `assistant_memory/curator.py`:
- After each assistant turn, a daemon thread runs the curator (`pro` model).
- It scans the user message for declarative facts and the assistant reply for `[memory note: ...]` markers.
- Writes are tier-classified (`tiers.py`): Layer 1 silent appends; Layer 2 confirm (UI not wired — silently skipped).
- Manifest lines are updated incrementally.

**On exit** — consolidation pass + per-day session digest run in a daemon thread (the agent prints `Dreaming...`). The most recent per-turn curator is joined for up to 2s first so consolidation reads fresh manifests.

**Soul** — `identity/agent.md`. The frontmatter `immutable_core` is locked and always appended to the system prompt. The body is mutable but only via curator Layer 2 (gated by `[memory note: ...]`). Never edited with `str_replace`.

### Slash commands

| Command | Description |
|---|---|
| `/memory list <scope>` | List files in a scope |
| `/memory show <scope>/<file>` | Print a memory file |
| `/memory rebuild <scope>` | Regenerate a scope manifest |
| `/memory current` | Show `context/current.md` |
| `/memory help` | List all memory commands |
| `/skills` | List installed skills |
| `/<skill-name>` | Load a skill's instructions into context |

### Migrating from the old JSON store

```bash
uv run python -m scripts.migrate_to_assistant_memory
```

Converts `agent-memory/personality.json` → `identity/agent.md`, projects + digests → markdown, and renames the legacy dir to `agent-memory.legacy/`.

## Project layout

```
src/agent/
├── cli/            app.py · display.py            REPL + Rich UI
├── core/           loop.py · state.py             ReAct loop, ConversationState (transient suffix)
├── llm/            client.py                      provider-agnostic chat client
├── config/         settings.py                    .env loading, backend selection
├── assistant_memory/
│   ├── manager.py        public lifecycle (on_startup/turn/exit, retrieve_for_query, handle_command)
│   ├── retrieval.py      two-stage retrieval pipeline
│   ├── curator.py        extract facts, classify, apply writes, summarize sessions
│   ├── store.py          markdown + frontmatter IO, atomic writes
│   ├── schema.py         dataclasses, get_memory_dir()
│   ├── prompts.py        Stage 1/2/main/curator templates
│   ├── signal_detector.py  entity hints for Stage 2
│   ├── tiers.py          write-tier classification
│   ├── page.py           manifest pagination
│   └── templates.py      scope/manifest skeletons
├── tools/          read/write/edit/delete + bash, web_search, beautify, read_skill, registry
├── permissions/    gates.py                       session-scoped delete grants
├── skills/         discovery.py · loader.py · manager.py · models.py
├── feishu/         server.py · client.py          long-connection Feishu bot
├── memory/         models.py                      legacy — kept for the migration script
├── logger.py · log_server.py                       socket logger + TCP debug server
server/             main.py + bridge               FastAPI + WebSocket bridge for the web pet
web/                Vite + React + TS frontend; cat sprites in web/public/cat-sprites
cat/                PixelLab sprite source assets
scripts/            migrate_to_assistant_memory.py
tests/              pytest suite (memory, curator, retrieval, tools, permissions, skills, …)
```

Top-level entry points (from `pyproject.toml`):

- `agent` → `agent.cli.app:main`
- `log-server` → `agent.log_server:main`
- `feishu-bot` → `agent.feishu.server:main`

## Testing

```bash
uv run pytest -q
```

Covers store, retrieval, curator, manager, slash commands, delete permissions, tiers, signal detector, skills, compaction, the loop, the bash tool, and the display layer's `[memory note: ...]` filter.

## License

MIT

---

<a id="中文"></a>

# 中文

从零开始用 Python 写的 ReAct agent。一套核心，三种入口：

- **CLI** — 终端 REPL，流式 Markdown 渲染、工具调用、删除权限确认。
- **Web** — 可拖动的橘色像素虎斑猫桌宠（`web/`），点击弹出对话气泡；后端 FastAPI + WebSocket 桥接（`server/`）。
- **飞书机器人** — 长连接 Bot（`src/agent/feishu/`），按 `open_id` 维护独立会话。

Agent 自带 **markdown 原生记忆库**：每轮两阶段检索挑选相关文件，后台 curator 从用户消息和助手回复中的 `[memory note: ...]` 标记提取事实。

## 环境要求

- Python 3.11+
- [uv](https://docs.astral.sh/uv/)
- 三选一：`DEEPSEEK_API_KEY` · `GPT_AK`（字节跳动）· `OPENAI_API_KEY`
- Node 18+（仅 web 前端需要）

## 安装

```bash
uv sync
```

新建 `.env`：

```
DEEPSEEK_API_KEY=...        # 或 GPT_AK=... / OPENAI_API_KEY=...
```

自动检测优先级：**DeepSeek → ByteDance → OpenAI**。可用 `USE_DEEPSEEK=true` / `USE_BYTEDANCE=true` 强制。

可选环境变量见英文版表格。

## 运行

### CLI

```bash
uv run agent
uv run agent -memlog      # 实时显示橙色 MEMORY 面板（检索/整理事件）
uv run log-server         # 独立终端的调试日志服务
```

`quit` / `exit` / Ctrl+C 退出。连按两次 Ctrl+C 强制退出，立即中止后台 curator/digest 线程。

### Web 桌宠

两个终端：

```bash
uv run uvicorn server.main:app --reload --reload-dir server --port 8000
cd web && npm install && npm run dev
```

打开 <http://localhost:5173>，右侧面板实时显示每次工具调用和 curator 写入。

Web 模式下 turn 末尾的 `propose_writes` LLM 调用被关闭 —— 只有流中途出现的 `[memory note: ...]` marker 才会触发 curator。Marker 文本在转发到浏览器前剥离，用户看不到。

### 飞书机器人

```bash
uv run feishu-bot
```

长连接（WebSocket 出站），不需要公网 IP 或域名。

## 工具一览

| 工具 | 类别 |
|---|---|
| `read_file`、`list_dir` | 读 |
| `write_file`、`str_replace`、`file_rewrite`、`make_dir` | 写 |
| `delete_file`、`delete_dir` | 删（带权限确认） |
| `bash` | 执行（按 tier 分级） |
| `web_search` | 工具 |
| `beautify`、`read_skill`、`check_permissions`、`echo` | 工具 |

删除首次触碰路径会弹出确认面板：**授权本会话** / **仅这一次** / **取消**。父路径授权覆盖子路径。非 TTY 默认取消。文件写入采用临时文件 + 原子 rename；Python/JSON 文件写后做语法校验；路径限定在 cwd。

## 助手记忆

目录布局（默认 `~/assistant-memory/`）、读写流程、灵魂演化机制详见英文版。要点：

- **读侧**：两阶段检索（`flash` 模型选 scope，再选 ≤5 个文件），结果作为本轮 system 后缀注入，**不进入对话历史**。
- **写侧**：后台 daemon 线程跑 curator（`pro` 模型），解析事实陈述与 `[memory note: ...]` marker。Layer 1 静默 append，Layer 2 待 UI（暂时跳过）。
- **退出**：consolidate + 当日 digest 在守护线程中跑（屏幕显示 `Dreaming...`）。
- **灵魂**：`identity/agent.md` frontmatter 的 `immutable_core` 不可变，每次启动直接拼到 system prompt；body 只能通过 marker 写。永远不要用 `str_replace` 改它。

### 斜杠命令

| 命令 | 说明 |
|---|---|
| `/memory list <scope>` | 列出 scope 文件 |
| `/memory show <scope>/<file>` | 显示某个记忆文件 |
| `/memory rebuild <scope>` | 重新生成 manifest |
| `/memory current` | 显示 `context/current.md` |
| `/memory help` | 列出所有记忆命令 |
| `/skills` | 列出已安装 skill |
| `/<skill-name>` | 显式加载某个 skill |

### 旧 JSON 存储迁移

```bash
uv run python -m scripts.migrate_to_assistant_memory
```

## 测试

```bash
uv run pytest -q
```

## 许可证

MIT
