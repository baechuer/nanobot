# Nanobot Architecture Analysis

A comprehensive, code-grounded architecture audit of [nanobot](https://github.com/HKUDS/nanobot) — a lightweight personal AI assistant framework.

> **Historical scope**: Documents `00`-`12` were written against roughly `v0.1.4.post5`.
> The repository has moved on substantially. Start with `13_current_runtime_refresh.md`
> for the codebase shape in the current checkout (`v0.1.5` in this repo).

## What This Contains

| Document | Purpose |
|---|---|
| [00_overview.md](00_overview.md) | High-level architecture, decomposition, and Mermaid diagrams |
| [01_repo_map.md](01_repo_map.md) | Directory structure, file responsibilities, must-read markers |
| [02_boot_sequence.md](02_boot_sequence.md) | Boot paths for `onboard`, `agent`, and `gateway` commands |
| [03_agent_loop.md](03_agent_loop.md) | Core agent loop logic, pseudocode, state mutations |
| [04_memory_system.md](04_memory_system.md) | Two-layer memory model, consolidation policy, persistence |
| [05_skills_and_tools.md](05_skills_and_tools.md) | Tool registry, MCP integration, skill loading |
| [06_context_and_reasoning.md](06_context_and_reasoning.md) | Context construction, prompt assembly, reasoning capabilities |
| [07_channels_and_gateway.md](07_channels_and_gateway.md) | Channel adapters, message bus, outbound routing |
| [08_proactive_runtime.md](08_proactive_runtime.md) | Cron, heartbeat, subagents, background execution |
| [09_config_security_runtime_constraints.md](09_config_security_runtime_constraints.md) | Config schema, security controls, workspace isolation |
| [10_component_abstraction.md](10_component_abstraction.md) | Research-level component decomposition |
| [11_design_critique.md](11_design_critique.md) | Strengths, weaknesses, trade-offs, refactor priorities |
| [12_callgraph_hotspots.md](12_callgraph_hotspots.md) | Top functions/classes to read first |
| [13_current_runtime_refresh.md](13_current_runtime_refresh.md) | Current-runtime architecture guide and mismatch audit for `v0.1.5` |

## Recommended Reading Order

### For current onboarding
1. `13_current_runtime_refresh.md` — What the current repo looks like now
2. `00_overview.md` — Historical overview from the older runtime snapshot
3. `01_repo_map.md` — Historical file map; still useful for orientation
4. `03_agent_loop.md` — Historical deep dive into the loop concepts

### For historical context
1. `00_overview.md` — Architecture mental model
2. `01_repo_map.md` — What's where
3. `03_agent_loop.md` — The central runtime
4. `02_boot_sequence.md` — How it starts

### For research
1. `10_component_abstraction.md` — Agent component model
2. `06_context_and_reasoning.md` — Cognitive architecture
3. `04_memory_system.md` — Persistence model
4. `11_design_critique.md` — Trade-off analysis

### For refactoring
1. `12_callgraph_hotspots.md` — Where to start reading
2. `11_design_critique.md` — What needs work
3. `09_config_security_runtime_constraints.md` — Safety boundaries

## Methodology

Every claim is grounded in specific code locations (file paths, line numbers, class/function names). Where behavior is inferred rather than directly observed, it is explicitly marked as **[Inferred]**.

Analysis was performed by recursive reading of every source file in the repository, tracing runtime execution paths, and cross-referencing test coverage.
