# Skills (`abdullah-mhassan/skills`)

**Modular, reusable skills for AI coding assistants and autonomous agents.**

Skills are folders of instructions, scripts, and resources that AI agents load dynamically to improve performance on specialized engineering and agentic tasks.

---

## Installation

```bash
npx skills add abdullah-mhassan/skills
```

---

## Skills Catalog

| Skill | Category | Description | Location |
| :--- | :--- | :--- | :--- |
| `agent-creator` | `obagents` | Design, structure, and author new OB Agents with rich personas, roles, boundaries, and goals. | [`skills/obagents/agent-creator`](skills/obagents/agent-creator/SKILL.md) |

---

## Usage

Invoke installed skills directly in your agent session:

```text
Use $agent-creator to build a specialized QA engineer agent with strict boundary rules.
```

---

## Updating Skills

Update installed skills using the Skills CLI:

```bash
npx skills update
```

---

## Repository Structure

```
.
└── skills/                       # Skills organized by category (<category>/<skill-name>/)
    └── obagents/                 # OB Agents skill category
        └── agent-creator/        # Agent creator skill definition
            ├── SKILL.md          # Primary skill prompt & frontmatter
            └── feature-map.md    # Capability mapping reference
```
