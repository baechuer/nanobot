# 01 — Repository Map

## Repo Tree Overview

```
nanobot/
├── nanobot/                    # Main Python package
│   ├── __main__.py             # ★ Entry point (python -m nanobot)
│   ├── __init__.py             # Version + logo
│   ├── agent/                  # ★★★ Core agent runtime
│   │   ├── loop.py             # ★★★ AgentLoop — the control center (511 lines)
│   │   ├── context.py          # ★★ ContextBuilder — prompt assembly (196 lines)
│   │   ├── memory.py           # ★★ MemoryStore + MemoryConsolidator (358 lines)
│   │   ├── skills.py           # ★ SkillsLoader — SKILL.md discovery (229 lines)
│   │   ├── subagent.py         # ★ SubagentManager — background tasks (236 lines)
│   │   └── tools/              # Tool implementations
│   │       ├── base.py         # ★ Tool ABC + validation (182 lines)
│   │       ├── registry.py     # ★ ToolRegistry — register/execute (71 lines)
│   │       ├── filesystem.py   # ReadFile, WriteFile, EditFile, ListDir tools
│   │       ├── shell.py        # ExecTool — shell commands with safety guards
│   │       ├── web.py          # WebSearchTool + WebFetchTool
│   │       ├── message.py      # MessageTool — send to channel
│   │       ├── spawn.py        # SpawnTool — create subagent
│   │       ├── cron.py         # CronTool — schedule jobs
│   │       └── mcp.py          # ★ MCPToolWrapper — MCP server integration
│   ├── bus/                    # Message bus (in-process)
│   │   ├── events.py           # InboundMessage / OutboundMessage dataclasses
│   │   └── queue.py            # MessageBus — dual asyncio.Queue
│   ├── channels/               # Chat platform adapters
│   │   ├── base.py             # ★ BaseChannel ABC (140 lines)
│   │   ├── manager.py          # ★ ChannelManager — routing + lifecycle
│   │   ├── registry.py         # Auto-discovery (pkgutil + entry_points)
│   │   ├── telegram.py         # Telegram adapter (32K)
│   │   ├── feishu.py           # Feishu/Lark adapter (50K — largest file)
│   │   ├── discord.py          # Discord adapter
│   │   ├── slack.py            # Slack adapter
│   │   ├── dingtalk.py         # DingTalk adapter
│   │   ├── matrix.py           # Matrix adapter
│   │   ├── email.py            # Email adapter
│   │   ├── whatsapp.py         # WhatsApp adapter
│   │   ├── wecom.py            # WeCom adapter
│   │   ├── mochat.py           # MoChat adapter (39K)
│   │   └── qq.py               # QQ adapter
│   ├── cli/                    # CLI commands
│   │   └── commands.py         # ★★ All CLI commands (1134 lines)
│   ├── config/                 # Configuration
│   │   ├── schema.py           # ★ Pydantic config models (261 lines)
│   │   ├── loader.py           # Config load/save/migrate
│   │   └── paths.py            # Runtime path helpers
│   ├── cron/                   # Scheduled tasks
│   │   ├── service.py          # CronService — timer-based scheduler
│   │   └── types.py            # CronJob, CronSchedule, etc.
│   ├── heartbeat/              # Periodic agent wake-up
│   │   └── service.py          # HeartbeatService — LLM-driven 2-phase
│   ├── providers/              # LLM provider abstraction
│   │   ├── base.py             # ★ LLMProvider ABC + retry logic (281 lines)
│   │   ├── registry.py         # ★ ProviderSpec registry (524 lines)
│   │   ├── litellm_provider.py # ★ LiteLLMProvider — main provider (356 lines)
│   │   ├── custom_provider.py  # Direct OpenAI-compatible endpoint
│   │   ├── azure_openai_provider.py # Azure OpenAI
│   │   ├── openai_codex_provider.py # OpenAI Codex (OAuth)
│   │   └── transcription.py    # Groq Whisper transcription
│   ├── security/               # Safety utilities
│   │   └── network.py          # SSRF protection, URL validation
│   ├── session/                # Conversation persistence
│   │   └── manager.py          # ★ Session + SessionManager (JSONL)
│   ├── skills/                 # Built-in skill definitions
│   │   ├── cron/SKILL.md       # Cron scheduling instructions
│   │   ├── memory/SKILL.md     # Memory management instructions
│   │   ├── github/SKILL.md     # GitHub workflow
│   │   ├── summarize/SKILL.md  # Summarization
│   │   ├── skill-creator/      # Skill creation metaskill
│   │   ├── weather/SKILL.md    # Weather lookup
│   │   ├── tmux/               # Tmux session management
│   │   └── clawhub/SKILL.md    # ClawHub integration
│   ├── templates/              # Workspace file templates
│   │   ├── AGENTS.md           # Agent instructions template
│   │   ├── SOUL.md             # Personality template
│   │   ├── USER.md             # User profile template
│   │   ├── TOOLS.md            # Tool usage notes
│   │   ├── HEARTBEAT.md        # Heartbeat task file template
│   │   └── memory/MEMORY.md    # Initial memory template
│   └── utils/                  # Shared utilities
│       ├── helpers.py          # Token estimation, file helpers
│       └── evaluator.py        # LLM-based notification gating
├── bridge/                     # WhatsApp bridge (TypeScript/Node.js)
│   └── src/                    # Socket.IO bridge server
├── tests/                      # 45 test files
├── pyproject.toml              # Package definition
├── Dockerfile                  # Docker build
└── docker-compose.yml          # Docker compose
```

## ★ Markers: Must-Read-First Files

| Priority | File | Why |
|---|---|---|
| ★★★ | `agent/loop.py` | The entire runtime control flow lives here |
| ★★ | `cli/commands.py` | All commands + wiring of components |
| ★★ | `agent/context.py` | System prompt composition |
| ★★ | `agent/memory.py` | Memory architecture + LLM consolidation |
| ★ | `agent/tools/base.py` | Tool contract definition |
| ★ | `agent/tools/registry.py` | How tools are discovered and executed |
| ★ | `agent/tools/mcp.py` | MCP integration pattern |
| ★ | `channels/base.py` | Channel adapter contract |
| ★ | `config/schema.py` | Entire config surface area |
| ★ | `providers/base.py` | LLM provider contract + retry |
| ★ | `providers/litellm_provider.py` | Main LLM routing |
| ★ | `providers/registry.py` | Provider matching logic |
| ★ | `session/manager.py` | Session persistence model |

## Critical vs Auxiliary Files

### Critical to Runtime

All files under `agent/`, `bus/`, `cli/commands.py`, `config/`, `providers/base.py`, `providers/litellm_provider.py`, `providers/registry.py`, `session/manager.py`, `channels/base.py`, `channels/manager.py`, `channels/registry.py`, `cron/service.py`, `heartbeat/service.py`, `security/network.py`, `utils/helpers.py`.

### Auxiliary

- `templates/` — only used during `onboard` to seed workspace files
- `skills/` — prompt assets, not runtime code
- `bridge/` — external Node.js process for WhatsApp
- `tests/` — not loaded at runtime
- `utils/evaluator.py` — only used by heartbeat and cron for notification gating
- Individual channel adapters (e.g., `telegram.py`) — only loaded when enabled
