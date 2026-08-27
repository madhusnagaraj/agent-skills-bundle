# Agent-Building Skills Bundle

Four consultative skills for Claude Code that guide the core decisions of building an agent:

| Skill | What it guides | Fires when you say |
|---|---|---|
| `agent-architect` | Which architecture: single call, workflow, orchestrator, or autonomous agent | "build an agent", "workflow vs agent", "agent architecture" |
| `tool-designer` | Designing fewer, smarter tools with descriptions agents can act on | "create a tool", "MCP tool", "tool description" |
| `context-engineer` | What goes in the context window, when, and what gets discarded | "manage context", "compaction", "agent memory" |
| `agent-evaluator` | Defining success, designing eval cases, building the measurement loop | "evaluate agent", "agent evals", "measure performance" |

Each skill is a consultative process, not a reference dump. It asks questions before recommending, defaults to the simplest thing that works, and pushes back on over-engineering. Detailed material lives in each skill's `references/` folder and is loaded only when needed.

## Install (personal, all projects)

```bash
git clone https://github.com/madhusnagaraj/agent-skills-bundle.git
cp -R agent-skills-bundle/skills/* ~/.claude/skills/
```

## Install (one project, shared with your team)

```bash
git clone https://github.com/madhusnagaraj/agent-skills-bundle.git
mkdir -p your-project/.claude/skills
cp -R agent-skills-bundle/skills/* your-project/.claude/skills/
```

No git? Use Code > Download ZIP on this page, unzip, and copy the `skills` folder's contents to the same destinations.

Start a new Claude Code session and run `/skills` to confirm all four are listed. Then say "I want to build an agent that ..." and `agent-architect` should pick it up.

## License

MIT. Adapt freely; the folder name is the skill's command name, so rename the folder if you rename the skill.
