# OB Agents Active Layer — Feature Map

The Active Layer is the live, project-scoped MCP gateway (`obagents serve` — one shared server per project, no agent argument; the served agent/project are resolved dynamically per call) that exposes the OB Agents Vault to AI coding tools. This map covers every tool it registers, with exact schemas and semantics.

> **Schemas evolve.** This map was cross-checked against `src/mcp/tools/*.ts` at authoring time. If a tool behaves differently than described, the live MCP server is the source of truth — verify before relying on a detail.

All agent names are passed **without** a leading `@` (e.g. `designer`, not `@designer`).

## Gateway trust boundary (0.5.0)

The gateway serves **one project tree** — the directory it was started in. Tool-call `project`/`projectPath` arguments must resolve to the startup project or a subdirectory of it; sibling or arbitrary paths are rejected (project spoofing guard). `link_agent` resolves its target through the gateway with an empty roster allowed (bootstrap), and `consolidate_agent` rejects agents that are not in the resolved project's roster.

## Hive tools

Hive orchestration — spawning, linking, consulting, and consolidating agents.

### `create_agent`

Initialize a new AI agent in the Vault. Use to spawn a worker or a specialist for a task.

| Param | Type | Notes |
|---|---|---|
| `name` | string | Validated: `^[a-z0-9-_]+$`, reserved names (`__proto__`, `constructor`, …) rejected, max 1000 agents (enforced here too) |
| `description` | string | One-line description |

Returns `{ success, agent, path }`.

### `link_agent`

Assign an existing agent to a specific project workspace. Injects the agent's context into the project.

| Param | Type | Notes |
|---|---|---|
| `name` | string | Existing agent |
| `targets` | string[] | One or more of the verified core targets: `claude-code`, `cursor`, `codex`, `opencode`, `antigravity`, `copilot`, `command-code`, `generic` (the 9 legacy targets — windsurf, roo, continue, aider, kilo, grok, qwen, pi, swe-agent — are unlink-only cleanup since 0.4.0) |
| `projectPath` | string? | Defaults to the gateway's startup project; must stay inside the gateway's project tree |

Returns `{ success, outcome }`. Link/unlink mutations apply target adapters first with rollback on error; only success commits graph metadata. Per-link MCP entries pin `--project` to the project dir; global gateway installs run unpinned `serve`.

### `consolidate_agent`

Archive an agent's current `MEMORY.md` and replace it with a shorter summary to save context window space.

| Param | Type | Notes |
|---|---|---|
| `name` | string | Agent whose memory is consolidated; must be linked to the resolved project (roster check) |
| `summary` | string | The new short summary (≤2500 chars) |
| `project` | string? | Resolves through the gateway; defaults to the served project |

Returns `{ success, episodeId }`. The archived memory is preserved as an episode tagged with the canonical compound project tag; covered `memory` rows are marked via `archived_by` so decay prunes only what this consolidation actually covered. The summary is written verbatim — nothing "populates automatically".

### `load_agent_context`

Retrieve another agent's full compiled state (SOUL, MEMORY, USER). Cheap, deterministic, memory-only — no files read, no web search.

| Param | Type | Notes |
|---|---|---|
| `targetAgent` | string | Existing agent; must be in the resolved project's roster (gateway mode) |
| `project` | string? | Resolves through the gateway |

Returns `{ memory, note }` where `note` (MEMORY_ONLY_NOTE) states the lookup was a deterministic read of the agent's recorded episode log — no project files read, no web search. Prefer this over `consult_agent` when you need the full rules + memory, not an answer.

### `consult_agent`

Query another agent's memory deterministically to discover past decisions. The ONLY reliably scoped way to read another agent's memory.

| Param | Type | Notes |
|---|---|---|
| `targetAgent` | string | Agent whose memory is searched; roster-checked |
| `query` | string | Search text |
| `limit` | number? | Positive int, max 100 |

Returns ranked episodes. This is a lookup, not task execution — do not escalate to a live sub-agent without user approval. Do not substitute a generic `search_history`, file reads, or web search for this: those are not scoped to the target agent and will miss its memory.

## Memory tools

An agent's own deep memory — state, episodes, and skills.

### `read_state`

Read the agent's full compiled state (SOUL, MEMORY, and USER).

| Param | Type | Notes |
|---|---|---|
| `targetAgent` | string? | Defaults to the active runtime agent |
| `project` | string? | Resolves through the gateway |

### `update_state`

Record a structured memory entry (typed milestone) into the agent's deep-memory store. Requires `type` and `summary` (as of v0.2.1). `MEMORY.md` remains the agent's readable prose view.

| Param | Type | Notes |
|---|---|---|
| `type` | enum | `build-fixed` \| `decision` \| `bug-fixed` \| `milestone` |
| `summary` | string | Required, 1–2000 characters; keep entries atomic — split into multiple calls. Injection-scan rejected: instruction-injection, credential-shaped, invisible Unicode |
| `supersedes` | number? | Positive integer episode id; must exist and belong to this agent |
| `project` | string? | Defaults to the served project, else the global scope |

Returns `{ success, entryId, type, supersedes, project, needsConsolidation, rowsSinceConsolidation, threshold, nearDuplicates, memoryAppended, memorySkipped }` — with `duplicate: true` when the exact entry already exists (idempotent by content). When the prose mirror hits its character cap, `memoryFull: true` + consolidation guidance is returned: consolidate first, then re-record.

### `search_history`

Search the agent's long-term episodic memory via FTS5. Returns ranked episodes.

| Param | Type | Notes |
|---|---|---|
| `query` | string | Search text |
| `limit` | number? | Default 10, positive int, max 100 |
| `global` | boolean? | When set, searches across projects instead of the served project |

Results include `superseded_by` — treat superseded entries as historical, not current.

### `learn_skill`

Save a skill as `skills/<name>/SKILL.md` inside the agent's vault and record an episode.

| Param | Type | Notes |
|---|---|---|
| `name` | string | Must match `^[a-z0-9-_]+$` (lowercase, digits, hyphens, underscores) |
| `protocol` | string | Full SKILL.md content; capped at 16K chars; injection-scanned; symlinked skill paths refused |

Returns `{ success, path }` — or `{ success, unchanged: true, path }` when the file is byte-identical. The vault writes the protocol verbatim; future sessions reload the skill instead of re-deriving it.

## Semantics that apply to all tools

- **Project scoping** — memory entries and searches are scoped to the served project unless `project`/`global` is given. Project paths containing `%`/`_` are matched literally (LIKE wildcards escaped).
- **Consolidation trigger** — at 20 rows since the last consolidation (`threshold`), `needsConsolidation` becomes true and a `consolidate_agent` call is due. Consolidation rows mark their inputs via `archived_by`; decay prunes only rows a consolidation actually covered.
- **Duplicates** — `update_state` dedupes by exact content; `learn_skill` by exact file content.
- **MEMORY_ONLY_NOTE** — deterministic lookups return a note clarifying their source: the agent's recorded episode log (decisions, tool-call records, skills, consolidation summaries), scoped to the agent's vault — no project files were read and no web search was performed. It describes episode sources, not a separate "memory-only" store. Prefer `consult_agent` over generic `search_history` when you need another agent's memory.

## Target wiring (0.5.0)

| Target | Uses MCP? | Config file written | Wiring mechanism |
|--------|-----------|---------------------|------------------|
| cursor | ✅ | `.cursor/mcp.json` | mcpServers |
| copilot | ✅ | `~/.vscode/mcp.json` | servers |
| claude-code | ✅ | `~/.claude.json` (global) | mcpServers + JSONC-preserving `~/.claude/settings.json` contextPaths edit |
| opencode | ✅ | `opencode.json` | opencode |
| codex | ✅ | `codex mcp add obagents` (exec) | CLI, fails loudly on error |
| antigravity | ✅ | `~/.gemini/config/mcp_config.json` | mcpServers |
| command-code | ✅ | `~/.commandcode/mcp.json` (global, user scope) | mcpServers with `transport: "stdio"` |
| generic | ❌ | `AGENT.md` | instructions only |
