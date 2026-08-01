---
name: agent-creator
description: "Create a new OB Agents agent with a real persona — role, responsibilities, boundaries, style, goals — instead of a one-line description. Use when the user wants to create an agent, spawn a specialist or sub-agent for a task, or when another skill needs an agent created."
---

# Agent Creator

Create a well-described OB Agents **agent** in three phases: interview, generate, hand off. The result is a **triad** — `SOUL.md`, `MEMORY.md`, `USER.md` — in the agent's Vault, with the Core Directives block appended by the tool.

This skill drives the `obagents` CLI. For the full Active Layer tool schemas, see [feature-map.md](feature-map.md).

## Phase A — Interview

Ask the user for the seven answers, one at a time:

1. **Name** — must match `^[a-z0-9-_]+$` (lowercase letters, digits, hyphens, underscores). No `@` prefix.
2. **One-line description** — what the agent is, for tool listings.
3. **Role** — what the agent is for, in one or two sentences.
4. **Responsibilities** — the recurring work the agent owns.
5. **Boundaries** — what the agent does not do, including explicit non-goals.
6. **Style** — how the agent communicates and works.
7. **Goals** — what success looks like for the agent.

When a field stalls, propose a default drawn from the **archetype** that matches the role instead of skipping the question.

**Completion criterion:** all seven answers are recorded before Phase B starts.

## Phase B — Generate

Match the interview to a built-in **archetype**: `engineer`, `designer`, `copywriter`, or `orchestrator`. When the persona fits one, generate directly:

```bash
obagents create <name> --template <archetype> --description "<one-line description>"
```

When the persona fits no archetype, author a custom template directory with three files — `SOUL.md` (sections: Role, Responsibilities, Boundaries including non-goals, Style, Goals), framed `MEMORY.md`, framed `USER.md`. Use the `{{AGENT_NAME}}` and `{{AGENT_DESCRIPTION}}` placeholders so the tool substitutes the interviewed answers, and reuse the operating-principles bullets from the default SOUL where role-appropriate. Skeleton files contain no Core Directives block — the tool appends it. Then:

```bash
obagents create <name> --template /path/to/template-dir --description "<one-line description>"
```

Pass the interviewed one-liner as `--description` — the CLI only asks for it interactively when no name is given, and the persona placeholder depends on it.

**Completion criterion:** `obagents create` reports success and every placeholder was substituted in the written triad.

## Phase C — Hand Off

1. **Verify the triad** — `SOUL.md`, `MEMORY.md`, `USER.md` exist in the agent's Vault, and `SOUL.md` ends with the Core Directives block.
2. **Link** — `obagents link <name>` (pick the target tool; `-t <target>` skips the prompt).
3. **Activate** — `obagents activate <name>`, or `obagents serve <name>` to run the agent's Active Layer MCP server.

**Completion criterion:** the verified triad is shown to the user and the agent is linked and active.

## Working memory discipline

After creation, the agent maintains its memory through the Active Layer tools:

- Record durable outcomes as typed entries — `build-fixed`, `decision`, `bug-fixed`, or `milestone` — kept atomic and under 2000 characters.
- Refresh a stale `MEMORY.md` with `consolidate_agent` — the summary is written verbatim; nothing populates automatically.
- Persist a reusable capability with `learn_skill` so future sessions reload it instead of re-deriving it.
