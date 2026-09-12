# Agent Memory Generation & Optimization (Kirby AgentMD Generation)

[![Kirby Skills Collection](https://img.shields.io/badge/Kirby_Skills-Collection-blue?style=flat-square&logo=github)](https://github.com/markkirby125/kirby-skills-collection)

You rely on system prompts and memory files (`AGENTS.md`, `CLAUDE.md`, `REASONIX.md`) to instruct your AI agents. It used to be perfectly reasonable to just dump all your project references, tech stack facts, and API endpoints directly into these files.

**However, as context windows grow, this practice triggers a severe "Always-Loaded Tax."** Every single token in a memory file is re-sent on every interaction turn. When an agent's context balloons past 100k tokens, it enters the "Dumb Zone"—losing track of negative constraints, skipping steps, and making erratic tool calls due to context bloat.

By ignoring memory optimization, your AI agents become exponentially more expensive, significantly slower, and far less reliable at following core behavioral protocols.

**The Solution:** The `kirby-agentmd-generation` skill forces your agent to aggressively slim down memory files. It establishes a deterministic pipeline to move static reference knowledge into on-demand persistent stores (like the `codebase-memory-mcp` memory cache and ADRs) while keeping essential behavioral rules load-bearing in the root context. Crucially, it dictates exactly how to initialize and commit the `.codebase-memory/` cache so this knowledge persists across sessions and teams.

## Installation & Usage
This is a standard AI agent skill (compatible with Antigravity, Cursor, Windsurf).
1. Copy the `SKILL.md` file into your agent's skills directory.
2. Trigger the skill by asking your agent to "generate agents.md", "slim down memory bloat", or "configure the memory cache".

> ### 🪄 The Magic Prompt
> Copy and paste this directly to your AI (Cursor, Windsurf, Claude Code):
> 
> ```markdown
> @agent Please install the kirby-agentmd-generation skill into this workspace.
> 1. Read the `SKILL.md` file from this repository: https://github.com/markkirby125/kirby-agentmd-generation
> 2. Identify the correct rules system for our current environment (e.g., `.cursor/rules/` for Cursor, `.windsurfrules` for Windsurf, `.clinerules` for Cline, or `~/.gemini/config/skills/` for Antigravity).
> 3. Save the contents appropriately.
> 4. Confirm when the installation is complete.
> ```
