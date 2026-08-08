# OB Agents Active Layer — Feature Map

The Active Layer is the live, project-scoped MCP gateway (`obagents serve` — one shared server per project, no agent argument; the served agent/project are resolved dynamically per call) that exposes the OB Agents Vault to AI coding tools. This map covers every tool it registers, with exact schemas and semantics.

> **Schemas evolve.** This map was cross-checked against `src/mcp/tools/*.ts` at authoring time. If a tool behaves differently than described, the live MCP server is the source of truth — verify before relying on a detail.

All agent names are passed **without** a leading `@` (e.g. `designer`, not `@designer`).

## Hive tools

Hive orchestration — spawning, linking, consulting, and consolidating agents.

### `create_agent`

Initialize a new AI agent in the Vault. Use to spawn a worker or a specialist for a task.

| Param | Type | Notes |
|---|---|---|
| `name` | string | Validated: `^[a-z0-9-_]+$`, max 1000 agents |
| `description` | string | One-line description |

Returns `{ success, agent, path }`.

### `link_agent`

Assign an existing agent to a specific project workspace. Injects the agent's context into the project.

| Param | Type | Notes |
|---|---|---|
| `name` | string | Existing agent |
| `targets` | string[] | One or more of the verified core targets: `claude-code`, `cursor`, `codex`, `opencode`, `antigravity`, `copilot`, `generic` (the 10 legacy targets — windsurf, roo, continue, aider, kilo, grok, qwen, pi, swe-agent, command-code — are unlink-only cleanup since 0.4.0) |
| `projectPath` | string? | Defaults to the current working directory |

Returns `{ success, outcome }`. Link/unlink mutations apply target adapters first with rollback on error; only success commits graph metadata.

### `consolidate_agent`

Archive an agent's current `MEMORY.md` and replace it with a shorter summary to save context window space.

| Param | Type | Notes |
|---|---|---|
| `name` | string | Agent whose memory is consolidated |
| `summary` | string | The new short summary |

Returns `{ success, episodeId }`. The archived memory is preserved as an episode; the summary is written verbatim — nothing "populates automatically".

### `load_agent_context`

Retrieve another agent's full compiled state (SOUL, MEMORY, USER). Cheap, deterministic, memory-only — no files read, no web search.

| Param | Type | Notes |
|---|---|---|
| `targetAgent` | string | Existing agent; errors if it does not exist |

Returns `{ memory, note }` where `note` (MEMORY_ONLY_NOTE) states the lookup was a deterministic read of the agent's recorded episode log — no project files read, no web search. Prefer this over `consult_agent` when you need the full rules + memory, not an answer.

### `consult_agent`

Query another agent's memory deterministically to discover past decisions. The ONLY reliably scoped way to read another agent's memory.

| Param | Type | Notes |
|---|---|---|
| `targetAgent` | string | Agent whose memory is searched |
| `query` | string | Search text |
| `limit` | number? | Result cap |

Returns ranked episodes. This is a lookup, not task execution — do not escalate to a live sub-agent without user approval. Do not substitute a generic `search_history`, file reads, or web search for this: those are not scoped to the target agent and will miss its memory.

## Memory tools

An agent's own deep memory — state, episodes, and skills.

### `read_state`

Read the agent's full compiled state (SOUL, MEMORY, and USER). Takes no arguments.

### `update_state`

Record a structured memory entry (typed milestone) into the agent's deep-memory store. Requires `type` and `summary` (as of v0.2.1). `MEMORY.md` remains the agent's readable prose view.

| Param | Type | Notes |
|---|---|---|
| `type` | enum | `build-fixed` \| `decision` \| `bug-fixed` \| `milestone` |
| `summary` | string | Required, 1–2000 characters; keep entries atomic — split into multiple calls |
| `supersedes` | number? | Positive integer episode id; must exist and belong to this agent |
| `project` | string? | Defaults to the served project, else the global scope |

Returns `{ success, entryId, type, supersedes, project, needsConsolidation, rowsSinceConsolidation, threshold, nearDuplicates }` — with `duplicate: true` when the exact entry already exists (idempotent by content).

### `search_history`

Search the agent's long-term episodic memory via FTS5. Returns ranked episodes.

| Param | Type | Notes |
|---|---|---|
| `query` | string | Search text |
| `limit` | number? | Default 10 |
| `global` | boolean? | When set, searches across projects instead of the served project |

Results include `superseded_by` — treat superseded entries as historical, not current.

### `learn_skill`

Save a skill as `skills/<name>/SKILL.md` inside the agent's vault and record an episode.

| Param | Type | Notes |
|---|---|---|
| `name` | string | Must match `^[a-z0-9-_]+$` (lowercase, digits, hyphens, underscores) |
| `protocol` | string | Full SKILL.md content |

Returns `{ success, path }` — or `{ success, unchanged: true, path }` when the file is byte-identical. The vault writes the protocol verbatim; future sessions reload the skill instead of re-deriving it.

## Semantics that apply to all tools

- **Project scoping** — memory entries and searches are scoped to the served project unless `project`/`global` is given.
- **Consolidation trigger** — at 20 rows since the last consolidation (`threshold`), `needsConsolidation` becomes true and a `consolidate_agent` call is due.
- **Duplicates** — `update_state` dedupes by exact content; `learn_skill` by exact file content.
- **MEMORY_ONLY_NOTE** — deterministic lookups return a note clarifying their source: the agent's recorded episode log (decisions, tool-call records, skills, consolidation summaries), scoped to the agent's vault — no project files were read and no web search was performed. It describes episode sources, not a separate "memory-only" store. Prefer `consult_agent` over generic `search_history` when you need another agent's memory.
