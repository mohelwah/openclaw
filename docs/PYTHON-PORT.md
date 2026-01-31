# Main codebase map and Python port guide

This doc maps the **main OpenClaw codebase** so you can recreate a similar project in Python. It identifies entry points, core modules, and how they connect.

---

## 1. Entry points and flow

| Layer | Path | Role |
|-------|------|------|
| **Binary** | `openclaw.mjs` | Node launcher; loads `dist/entry.js`. |
| **Entry** | `src/entry.ts` | Sets process title, env (profile, NO_COLOR), respawns with NODE_OPTIONS; then imports `src/cli/run-main.js`. |
| **CLI runner** | `src/cli/run-main.ts` | Loads `.env`, builds Commander program via `buildProgram()`, registers plugin CLIs, parses argv. |
| **Program** | `src/cli/program/build-program.ts` | Creates Commander app, registers all commands from `command-registry.ts`. |
| **Gateway** | `src/gateway/server.impl.ts` | `startGatewayServer()` — HTTP + WebSocket server, config, channels, agents, plugins. |

So the **main codebase** is:

- **CLI**: `src/cli/` (entry, run-main, program, command-registry, gateway-cli, etc.)
- **Gateway server**: `src/gateway/` (server.impl, server-methods, ws, config-reload, etc.)
- **Config**: `src/config/` (load/save/validate config)
- **Channels**: `src/channels/` (registry, plugins) + `extensions/*` (WhatsApp, Telegram, etc.)
- **Agents**: `src/agents/` (Pi embedded runner, tools, skills, model selection)
- **Auto-reply / sessions**: `src/auto-reply/`, `src/config/sessions/`, `src/sessions/`
- **Infra**: `src/infra/` (ports, env, binaries, restart), `src/process/`, `src/logging/`

The **published package** runs as:

```text
openclaw.mjs → dist/entry.js → run-main → buildProgram() → parseAsync(argv)
```

Subcommand `gateway` starts the long-lived server:

```text
gateway command → src/cli/gateway-cli/run.ts → startGatewayServer() (src/gateway/server.impl.ts)
```

---

## 2. Main codebase by area

### 2.1 CLI and commands

| Directory / file | Purpose |
|------------------|--------|
| `src/entry.ts` | Process setup, profile, then `run-main`. |
| `src/cli/run-main.ts` | Dotenv, build program, register plugins, parse argv. |
| `src/cli/program/build-program.ts` | One Commander program. |
| `src/cli/program/command-registry.ts` | Registers: setup, onboard, configure, config, maintenance, message, memory, agent, **subclis** (gateway, nodes, plugins, etc.), status/health/sessions, browser. |
| `src/cli/program/register.subclis.ts` | Registers sub-CLIs (gateway, nodes, daemon, plugins, channels, cron, etc.). |
| `src/cli/gateway-cli/` | `gateway` command: run, dev, discover; calls `startGatewayServer`. |
| `src/cli/deps.ts` | Shared CLI deps (config loader, paths, etc.). |

**Python equivalent**: Use `argparse` or `click` (or `typer`). One root parser, subparsers for each top-level command (`setup`, `onboard`, `gateway`, `status`, etc.). Mirror `command-registry` as a list of (name, register_function).

### 2.2 Gateway server (core runtime)

| Directory / file | Purpose |
|------------------|--------|
| `src/gateway/server.impl.ts` | `startGatewayServer(port, opts)`: loads config, migrates legacy, loads plugins/channels, creates HTTP + WebSocket server, cron, discovery, Tailscale, etc. Returns `{ close() }`. |
| `src/gateway/server/` | HTTP listen, TLS, WebSocket connection lifecycle. |
| `src/gateway/server-methods*.ts` | RPC handlers: agent, chat, channels, config, cron, devices, health, logs, models, nodes, sessions, skills, etc. |
| `src/gateway/server-ws-runtime.ts` | WebSocket message dispatch to methods. |
| `src/gateway/server-channels.ts` | Channel manager for gateway. |
| `src/gateway/server-chat.ts` | Creates agent event handler (incoming messages → agent loop). |
| `src/gateway/config-reload.ts` | Reload config without full restart. |
| `src/gateway/protocol/` | Schema/types for gateway API. |

**Python equivalent**: Async HTTP + WebSockets (e.g. `aiohttp` or `FastAPI` + WebSocket). One module for “server lifecycle” (start/close), one for “method handlers” (agent, chat, config, …). Expose the same logical API (e.g. JSON-RPC or REST + WS events).

### 2.3 Config

| Directory / file | Purpose |
|------------------|--------|
| `src/config/config.ts` | Re-exports: loadConfig, readConfigFileSnapshot, writeConfigFile, migrateLegacyConfig, paths (CONFIG_PATH, etc.). |
| `src/config/io.js` | Actual load/save (JSON5), snapshot, hash. |
| `src/config/validation.js` | Validate config object (with plugin schemas). |
| `src/config/sessions/` | Session key resolution, store paths, main session. |

**Python equivalent**: Single config module: load from file (JSON/JSON5/YAML), validate with Pydantic or a schema lib, expose `load_config()`, `write_config()`, and session paths. Keep the same logical structure (gateway, agents, channels, plugins) so you can share or port config files.

### 2.4 Channels (WhatsApp, Telegram, etc.)

| Directory / file | Purpose |
|------------------|--------|
| `src/channels/plugins/index.ts` | `listChannelPlugins()`, `getChannelPlugin(id)` — from plugin registry. |
| `src/channels/registry.ts` | Channel IDs, ordering. |
| `extensions/whatsapp/`, `extensions/telegram/`, etc. | Each extension: channel impl, gateway methods, CLI. |

Messages enter via channel adapters (extensions), then the gateway routes them into the **agent** (see below). So channels are “ingress”; the core is **gateway + agent**.

**Python equivalent**: Define a “channel” interface (connect, send, subscribe to messages). Implement one adapter per channel (e.g. WhatsApp via a Baileys port or REST API, Telegram via python-telegram-bot or similar). Register them in a channel manager used by the gateway.

### 2.5 Agents (LLM loop, tools, skills)

| Directory / file | Purpose |
|------------------|--------|
| `src/agents/pi-embedded.ts` | Pi embedded runner entry. |
| `src/agents/pi-embedded-runner/` | Run loop: stream to LLM, handle tool calls. |
| `src/agents/pi-embedded-subscribe.*` | Handlers for lifecycle, messages, tools. |
| `src/agents/pi-tools*.ts` | Tool definitions, policy, schema. |
| `src/agents/model-*.ts` | Model catalog, selection, failover. |
| `src/agents/skills/` | Skills loading, workspace, refresh. |
| `src/agents/auth-profiles/` | Auth (API keys, OAuth) for models. |
| `src/auto-reply/` | Auto-reply logic, templates. |

**Python equivalent**: One “agent” package: run loop (call LLM, parse tool calls, execute tools, stream back). Use OpenAI SDK or similar for chat completions; implement the same tool schema. Separate modules: model selection, skills loader, auth/store for keys.

### 2.6 Infra and support

| Directory / file | Purpose |
|------------------|--------|
| `src/infra/ports.ts` | Port availability, bind. |
| `src/infra/env.js` | Env normalization, accepted options. |
| `src/infra/dotenv.js` | Load `.env`. |
| `src/logging/` | Subsystem loggers, capture. |
| `src/process/` | Exec, child process. |
| `src/plugins/` | Plugin loader, registry, CLI registration. |

**Python equivalent**: Use `python-dotenv`, a small env module, and a logging pattern (e.g. structlog or standard logging with a single “gateway” logger). For plugins, either dynamic import of Python modules or a simple registry of callables.

---

## 3. Data flow (simplified)

```text
User / app
    │
    ▼
openclaw <command> [args]     →  CLI (src/cli)  →  Commander  →  command handler
    │
    └─ gateway   →  startGatewayServer()
                        │
                        ├─ HTTP server (health, control UI, optional OpenAI-style endpoints)
                        ├─ WebSocket server (RPC: chat, agent, config, channels, …)
                        ├─ Channel plugins (WhatsApp, Telegram, …)  →  incoming messages
                        ├─ Agent (Pi embedded)  ←  messages  →  LLM + tools
                        └─ Config, cron, discovery, Tailscale, etc.
```

So the **main codebase** for a minimal “same project” in Python is:

1. **CLI** – one entry script, one program with subcommands (gateway, status, config, …).
2. **Config** – load/write/validate one config file (same shape as OpenClaw).
3. **Gateway** – one HTTP+WS server; dispatch WS messages to “methods” (agent, chat, channels, health, …).
4. **Channels** – interface + at least one implementation (e.g. WhatsApp or Telegram).
5. **Agent** – one loop: receive message → LLM (with tools) → send back; same tool/skill concepts.
6. **Infra** – ports, env, logging.

---

## 4. How to create the same project in Python

### 4.1 Suggested layout

```text
openclaw_py/
  pyproject.toml / requirements.txt
  openclaw/
    __main__.py          # python -m openclaw
    cli/
      main.py            # argparse/click root, register subcommands
      gateway.py         # gateway start/dev
    config/
      load.py            # load_config(), paths
      validation.py      # validate config
    gateway/
      server.py          # start_gateway_server(), HTTP + WS
      methods/           # agent, chat, channels, health, ...
    channels/
      base.py            # ChannelPlugin protocol
      manager.py         # list/get plugins
      # implementations in separate packages or extensions/
    agents/
      runner.py          # one-turn or loop: message → LLM + tools → response
      tools.py           # tool registry, schema
      models.py          # model selection, API clients
    infra/
      env.py
      ports.py
      logging_config.py
  extensions/            # optional: whatsapp, telegram, ...
  scripts/
    openclaw             # wrapper script or use python -m
```

### 4.2 Order of implementation

1. **Config** – Load JSON/YAML, validate, expose `load_config()` and paths. No CLI yet.
2. **CLI** – Entry point (`__main__.py` or script), root parser, subcommand `gateway` that only prints “starting gateway on port X” (no server yet).
3. **Gateway server** – HTTP (e.g. one health route) + WebSocket; one stub method (e.g. `health`) that returns OK. No channels or agent.
4. **One channel** – Implement one channel (e.g. Telegram or a “debug” channel that reads stdin). Register it in the gateway; when a message arrives, log it.
5. **Agent** – Single function: “given message + session, call LLM (OpenAI or similar), return response.” No tools first. Wire it to the gateway so that channel messages trigger the agent and responses go back.
6. **Tools and skills** – Add tool schema and execution (e.g. same list as OpenClaw: send_message, read_file, etc.); then skills as config or files.
7. **More channels** – Add WhatsApp, etc., reusing the same channel interface and gateway wiring.
8. **Rest of CLI** – status, configure, onboard, plugins, etc., calling into config/gateway/agents as needed.

### 4.3 Library equivalents (Python)

| OpenClaw (Node) | Python |
|-----------------|--------|
| Commander | `argparse` or `click` or `typer` |
| HTTP + WebSocket | `aiohttp` or `FastAPI` + WebSocket |
| Config (JSON5) | `json` + optional `pyyaml`; validation: `pydantic` |
| WhatsApp (Baileys) | Check for “Baileys Python” or REST gateway; or use a different channel first |
| Telegram | `python-telegram-bot` or `aiogram` |
| OpenAI API | `openai` (official SDK) |
| Logging | `structlog` or `logging` |
| Env | `os.environ` + `python-dotenv` |

### 4.4 What to port first from this repo

- **Protocol / API shape** – From `src/gateway/protocol/` and `server-methods-list.ts`: method names, request/response shapes. Replicate in Python so existing clients (e.g. macOS app) can talk to your server.
- **Config schema** – From `src/config/` (types, validation): same keys and structure so config files can be shared or migrated.
- **Agent tool schema** – From `src/agents/pi-tools*.ts` and tool definitions: same tool names and arguments so skills behave the same.

---

## 5. Summary: where is the main code?

| What | Where |
|------|--------|
| **Entry** | `src/entry.ts` → `src/cli/run-main.ts` |
| **CLI** | `src/cli/` (run-main, program, command-registry, gateway-cli, subclis) |
| **Gateway** | `src/gateway/server.impl.ts` + `server/` + `server-methods*.ts` + `server-ws-runtime.ts` |
| **Config** | `src/config/` (config.ts, io, validation, sessions) |
| **Channels** | `src/channels/` (plugins, registry) + `extensions/*/` (whatsapp, telegram, …) |
| **Agents** | `src/agents/` (pi-embedded*, pi-tools*, model-*, skills/) |
| **Auto-reply / sessions** | `src/auto-reply/`, `src/config/sessions/`, `src/sessions/` |
| **Infra** | `src/infra/`, `src/process/`, `src/logging/` |

Creating “the same project” in Python means reimplementing this flow: **CLI → gateway server → channels + agent**, with the same config shape and a compatible gateway API so that UIs and tools can stay the same.
