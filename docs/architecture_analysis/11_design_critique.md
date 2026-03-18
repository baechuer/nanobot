# 11 — Design Critique and Trade-Off Analysis

## Strengths

### S1: Radical Simplicity
The entire agent runtime fits in ~4,500 lines of Python across 13 packages. There is no framework lock-in, no ORM, no external services required. A developer can read the entire codebase in a few hours.

**Code evidence**: `agent/loop.py` is 511 lines. The core processing pipeline (`_process_message`, `_run_agent_loop`, `_save_turn`) is ~250 lines total.

### S2: Clean Module Boundaries
Packages have clear responsibilities with minimal cross-dependency:
- `bus/` knows nothing about agents or channels
- `channels/` only interacts with the bus
- `agent/` only interacts with the bus and providers
- `providers/` is completely independent

**Code evidence**: Import graph is shallow. `bus/events.py` imports only `dataclasses` and `datetime`.

### S3: Provider Registry Design
The `ProviderSpec` + registry pattern (`providers/registry.py`) is the best-designed subsystem. Adding a new provider requires only:
1. Adding a `ProviderSpec` tuple entry
2. Adding a field to `ProvidersConfig`

No if-elif chains, no scattered provider-specific logic.

### S4: Safety-in-Depth for Shell
The `ExecTool` implements layered security: deny patterns → allow patterns → workspace restriction → path traversal detection → SSRF protection → timeout → output truncation. Each layer is independently understandable and testable.

### S5: Channel Plugin Architecture
Auto-discovery via `pkgutil` + `entry_points` means new channels can be added as installable packages without modifying core code.

### S6: HEARTBEAT.md Design
Using a user-editable Markdown file for heartbeat tasks is elegant — the agent can read *and write* the file, making it a bidirectional communication channel between the agent and its autonomous behavior.

---

## Weaknesses

### W1: Single Processing Lock — No Concurrency

```python
_processing_lock = asyncio.Lock()  # agent/loop.py:27
```

**Impact**: Only one message processed at a time across all channels. In gateway mode with multiple channels, messages from Telegram users block Discord users.

**Severity**: Medium. Acceptable for personal assistant use; unacceptable for multi-user deployment.

**Suggested fix**: Per-session locks instead of global lock. This would enable concurrent processing of independent conversations.

### W2: Session Save Is Full-File Rewrite

```python
def save(session):
    with open(path, "w") as f:  # session/manager.py:196
        for msg in session.messages:
            f.write(json.dumps(msg) + "\n")
```

**Impact**: Every message save rewrites the entire session file. After 500 messages, this is non-trivial I/O.

**Severity**: Low. File sizes are small for personal use. Would become a problem at scale.

**Suggested fix**: True append-only writes during `add_message()`, only rewrite on compaction.

### W3: No Memory Deduplication

The LLM-driven consolidation can produce redundant entries in `MEMORY.md`. There's no semantic deduplication — if the user mentions their name in 10 conversations, the memory might contain 10 references to it.

**Severity**: Low-Medium. Wastes context window space.

**Suggested fix**: Post-consolidation dedup pass, or include dedup instruction in consolidation prompt.

### W4: Hardcoded Constants

Several important limits are hardcoded rather than configurable:

| Constant | Value | Location |
|---|---|---|
| Max iterations | 25 | `agent/loop.py` |
| Max output chars | 10,000 | `agent/tools/shell.py:46` |
| Max history messages | 500 | `session/manager.py:69` |
| Message split size | 2000 | `utils/helpers.py:51` |
| Retry delays | 1, 2, 4 | `providers/base.py:77` |

**Severity**: Low. But creates surprise for users who need different limits.

### W5: No Graceful Degradation for MCP

If an MCP server connection fails during `_connect_mcp()`, tools from that server are silently unavailable. There's no retry, no reconnection on subsequent turns.

**Severity**: Medium. MCP connections are lazy and one-time. If the server comes back up, the agent must be restarted.

**Suggested fix**: Add reconnection logic in `MCPToolWrapper.execute()` or periodic health checks.

### W6: Memory Consolidation Uses LLM Tokens

Every consolidation requires a full LLM call to compress messages into MEMORY.md + HISTORY.md. This is:
- An additional cost per ~50 messages
- Additional latency when the threshold is hit
- Dependent on LLM quality for memory accuracy

**Severity**: Medium. The trade-off is thoughtful (LLM-quality compression) but expensive.

### W7: No Structured Error Reporting

Tool errors are returned as string prefixed with "Error:". The agent loop doesn't distinguish between tool validation errors, execution errors, and infrastructure errors.

```python
if result.startswith("Error"):  # registry.py:55
    return result + _HINT
```

**Severity**: Low. Works in practice because the LLM handles error strings well.

---

## Trade-Off Analysis

### T1: Single Process vs Multi-Process

**Chosen**: Single process, single event loop
**Alternative**: Worker pool with task queue (Celery, etc.)

| | Single Process | Worker Pool |
|---|---|---|
| Complexity | Very low | High |
| Scalability | 1 concurrent request | N concurrent requests |
| Memory sharing | Free (same process) | Requires serialization |
| Deployment | `python -m nanobot` | Multiple processes + broker |
| Failure domain | All-or-nothing | Per-worker isolation |

**Assessment**: Right choice for a personal assistant. The global lock will become a bottleneck only in multi-user scenarios.

### T2: File-Based vs Database

**Chosen**: JSONL sessions, Markdown memory, JSON cron
**Alternative**: SQLite, PostgreSQL, Redis

| | File-Based | Database |
|---|---|---|
| Setup | Zero | Requires DB |
| Inspection | Text editor | SQL client |
| Performance | O(n) per save | O(1) per operation |
| Durability | fsync-dependent | ACID |
| Concurrency | File lock (OS) | Transaction isolation |

**Assessment**: Excellent choice for simplicity and inspectability. The Markdown memory files are particularly good — users can directly read and edit them.

### T3: LLM Consolidation vs Embedding Search

**Chosen**: LLM compresses messages → MEMORY.md
**Alternative**: Embed messages → vector DB → retrieval

| | LLM Consolidation | Vector Search |
|---|---|---|
| Quality | High (semantic compression) | Variable (depends on embedding + retrieval) |
| Cost | One LLM call per consolidation | Embedding per message + search per query |
| Latency | Spike at consolidation | Consistent |
| Human-readable | Yes (Markdown) | No (vectors) |
| Dependencies | None extra | Vector DB, embedding model |

**Assessment**: MEMORY.md is a unique strength — it's the only AI assistant framework where you can open a text file and read exactly what the agent remembers.

### T4: Skills as Prompts vs Skills as Code

**Chosen**: SKILL.md files injected into system prompt
**Alternative**: Executable skill modules with custom logic

| | Prompt Skills | Code Skills |
|---|---|---|
| Authoring | Write Markdown | Write Python |
| Capabilities | Guide tool use | Arbitrary logic |
| Sandboxing | Naturally sandboxed | Needs isolation |
| Composability | Limited (prompt concatenation) | Full (function composition) |

**Assessment**: Prompt skills are remarkably effective given their simplicity. The trade-off is that complex multi-step procedures can't be reliably encoded in natural language alone.

---

## Refactoring Priorities

### Priority 1: Per-Session Locking (W1)
Replace `_processing_lock` with per-session locks to enable concurrent message processing.

### Priority 2: Extract Constants to Config (W4)
Move hardcoded limits to `config.json` schema with sensible defaults.

### Priority 3: MCP Reconnection (W5)
Add connection health checking and lazy reconnection in `MCPToolWrapper.execute()`.

### Priority 4: Append-Only Session Writes (W2)
Only append new messages to JSONL files; rewrite only on compaction/consolidation.

### Priority 5: Structured Error Types (W7)
Replace string-based error detection with a proper error enum/dataclass.
