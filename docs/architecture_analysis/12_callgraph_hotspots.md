# 12 — Critical Call Chains and Code-Reading Hotspots

## Top Call Chains

### Chain 1: User Message → Response (The Main Path)

```
cli/commands.py:gateway()
  → agent/loop.py:run() → consume_inbound()
    → agent/loop.py:_dispatch()
      → agent/loop.py:_process_message(msg)
        → session/manager.py:get_or_create(key) 
        → agent/memory.py:MemoryConsolidator.consolidate()     [if threshold hit]
        → agent/context.py:ContextBuilder.build_messages()
          → SkillsLoader.get_always_skills()
          → SkillsLoader.load_skills_for_context()
          → MemoryStore.read_memory()
        → agent/loop.py:_run_agent_loop()
          → providers/base.py:chat_with_retry()
            → providers/litellm_provider.py:chat()
              → litellm.acompletion()              [external LLM call]
          → agent/tools/registry.py:execute()
            → agent/tools/{tool}.py:execute()       [tool execution]
          → [loop back to chat_with_retry if tool_calls]
        → agent/loop.py:_save_turn()
          → session/manager.py:save()               [JSONL write]
      → bus/queue.py:publish_outbound()
    → channels/manager.py:_dispatch_outbound()
      → channels/{platform}.py:send()               [platform delivery]
```

**Lines to read**: `loop.py:134-242` (\_process_message), `loop.py:260-365` (\_run_agent_loop)

### Chain 2: Memory Consolidation

```
agent/loop.py:_maybe_consolidate()
  → utils/helpers.py:estimate_prompt_tokens() [token count check]
  → agent/memory.py:MemoryConsolidator.consolidate()
    → agent/memory.py:_format_messages()       [build conversation text]
    → agent/memory.py:MemoryStore.read_memory() [current MEMORY.md]
    → providers/base.py:chat_with_retry()       [LLM consolidation call]
    → agent/memory.py:MemoryStore.write_memory() [update MEMORY.md]
    → agent/memory.py:MemoryStore.append_to_history() [append HISTORY.md]
    → session/manager.py:save()                 [advance pointer]
```

**Lines to read**: `memory.py:193-300` (consolidate)

### Chain 3: Tool Execution (Generic)

```
agent/loop.py:_run_agent_loop()
  → provider response with tool_calls
  → for each tool_call:
    → agent/tools/registry.py:execute(name, params)
      → tool.cast_params(params)     [type coercion]
      → tool.validate_params(params) [JSON Schema validation]
      → tool.execute(**params)       [actual execution]
    → loop.py:_publish_tool_progress()  [progress hint to channel]
```

### Chain 4: Heartbeat Wake-Up

```
heartbeat/service.py:_run_loop()
  → heartbeat/service.py:_tick()
    → _read_heartbeat_file()                     [read HEARTBEAT.md]
    → _decide(content)                            [Phase 1: LLM decision]
      → providers/base.py:chat_with_retry()       [lightweight LLM call]
      → parse tool_call → (action, tasks)
    → if action == "run":
      → on_execute(tasks)                          [Phase 2: full agent loop]
        → agent/loop.py:process_direct(tasks)
      → utils/evaluator.py:evaluate_response()    [Phase 3: notification gate]
        → providers/base.py:chat_with_retry()      [another LLM call]
      → on_notify(response)                        [deliver to channel]
```

### Chain 5: Cron Job Execution

```
cron/service.py:_on_timer()
  → _load_store()                       [check for external modifications]
  → _execute_job(job)
    → on_job(job)                       [callback to agent]
      → agent/loop.py:process_direct(job.payload.message)
        → [full agent loop — same as Chain 1]
  → _save_store()                       [persist job state]
  → _arm_timer()                        [schedule next wake]
```

### Chain 6: MCP Tool Call

```
agent/tools/mcp.py:MCPToolWrapper.execute(**kwargs)
  → asyncio.wait_for(session.call_tool(original_name, arguments=kwargs), timeout)
    → MCP SDK → transport (stdio/SSE/HTTP) → external MCP server
    → parse result.content → text
```

### Chain 7: Provider Model Resolution

```
cli/commands.py:_make_provider(config)
  → config/schema.py:Config.get_provider_name(model)
    → providers/registry.py:find_by_model(model)      [standard provider match]
    → providers/registry.py:find_gateway(provider_name, api_key, api_base)  [gateway match]
  → config/schema.py:Config.get_provider(model)       [get credentials]
  → Switch provider_name → instantiate appropriate provider
  → providers/litellm_provider.py:LiteLLMProvider.__init__()
    → _setup_env()                                     [set env vars]
    → find_gateway()                                    [detect gateway]
```

## Function Hotspot Map

### Tier 1: Must-Read Functions (Core Control Flow)

| Function | File | Lines | Why |
|---|---|---|---|
| `AgentLoop._process_message()` | `loop.py` | ~134-242 | **The processing pipeline** — every message goes through here |
| `AgentLoop._run_agent_loop()` | `loop.py` | ~260-365 | **The LLM interaction loop** — tool calls happen here |
| `ContextBuilder.build_messages()` | `context.py` | ~45-120 | **Prompt assembly** — controls what the LLM sees |
| `MemoryConsolidator.consolidate()` | `memory.py` | ~193-300 | **Memory compression** — how context is managed |
| `ToolRegistry.execute()` | `tools/registry.py` | ~38-59 | **Tool dispatch** — validation + execution |

### Tier 2: Important Functions (Key Subsystems)

| Function | File | Lines | Why |
|---|---|---|---|
| `gateway()` | `commands.py` | ~457-642 | Wires all components together |
| `LLMProvider.chat_with_retry()` | `base.py` | ~226-275 | Retry/fallback logic for LLM calls |
| `LiteLLMProvider._resolve_model()` | `litellm_provider.py` | ~91-108 | Model name → LiteLLM routing |
| `Session.get_history()` | `manager.py` | ~69-93 | History windowing + tool-call alignment |
| `HeartbeatService._tick()` | `service.py` | ~143-175 | Two-phase heartbeat logic |
| `connect_mcp_servers()` | `mcp.py` | ~74-184 | MCP connection + tool registration |

### Tier 3: Supporting Functions (Good to Know)

| Function | File | Lines | Why |
|---|---|---|---|
| `_make_provider()` | `commands.py` | ~364-420 | Provider factory with gateway detection |
| `ExecTool._guard_command()` | `shell.py` | ~144-176 | Security guard logic |
| `validate_url_target()` | `network.py` | ~30-62 | SSRF protection |
| `SkillsLoader.build_skills_summary()` | `skills.py` | ~101-140 | Skills → XML for system prompt |
| `SubagentManager.spawn()` | `subagent.py` | ~70-120 | Background agent creation |
| `CronService._on_timer()` | `service.py` | ~227-243 | Cron tick handler |

## Class Hierarchy Map

```
LLMProvider (ABC)                    # providers/base.py
  ├── LiteLLMProvider               # providers/litellm_provider.py  ← 90% of traffic
  ├── CustomProvider                # providers/custom_provider.py
  ├── AzureOpenAIProvider           # providers/azure_openai_provider.py
  └── OpenAICodexProvider           # providers/openai_codex_provider.py

Tool (ABC)                           # agent/tools/base.py
  ├── ReadFileTool                  # agent/tools/filesystem.py
  ├── WriteFileTool                 # agent/tools/filesystem.py
  ├── EditFileTool                  # agent/tools/filesystem.py
  ├── ListDirTool                   # agent/tools/filesystem.py
  ├── ExecTool                      # agent/tools/shell.py
  ├── WebSearchTool                 # agent/tools/web.py
  ├── WebFetchTool                  # agent/tools/web.py
  ├── MessageTool                   # agent/tools/message.py
  ├── SpawnTool                     # agent/tools/spawn.py
  ├── CronTool (6 sub-tools)       # agent/tools/cron.py
  └── MCPToolWrapper                # agent/tools/mcp.py

BaseChannel (ABC)                    # channels/base.py
  ├── TelegramChannel              # channels/telegram.py
  ├── DiscordChannel               # channels/discord.py
  ├── SlackChannel                 # channels/slack.py
  ├── WhatsAppChannel              # channels/whatsapp.py
  ├── FeishuChannel                # channels/feishu.py
  ├── DingTalkChannel              # channels/dingtalk.py
  ├── MatrixChannel                # channels/matrix.py
  ├── EmailChannel                 # channels/email.py
  ├── MoChatChannel                # channels/mochat.py
  ├── WeComChannel                 # channels/wecom.py
  └── QQChannel                    # channels/qq.py
```

## Where to Start (By Goal)

| If you want to... | Start reading... |
|---|---|
| Understand the agent | `loop.py:_process_message()` then `_run_agent_loop()` |
| Add a new tool | `tools/base.py` (Tool ABC), copy `tools/shell.py` as template |
| Add a new channel | `channels/base.py` (BaseChannel ABC), copy `channels/telegram.py` |
| Add a new LLM provider | `providers/registry.py` (ProviderSpec), `providers/base.py` |
| Modify the system prompt | `context.py:build_messages()` |
| Change memory behavior | `memory.py:MemoryConsolidator.consolidate()` |
| Debug tool execution | `tools/registry.py:execute()` |
| Understand config | `config/schema.py` top-to-bottom |
| Add MCP support | `tools/mcp.py:connect_mcp_servers()` |
| Change boot sequence | `commands.py:gateway()` or `commands.py:agent()` |
