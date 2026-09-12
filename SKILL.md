---
name: kirby-agentmd-generation
description: "Use to generate, optimize, and slim down AGENTS.md / CLAUDE.md files by offloading reference knowledge to the codebase-memory-mcp graph and configuring the persistent memory-cache."
category: technique
triggers: [agents-md, generate-agents-md, slim-agents, memory-bloat, codebase-memory-mcp, memory-cache]
---

# Playbook: Generate & Slim `AGENTS.md` via `codebase-memory-mcp`

> Portable runbook. Import this into any project where an always-loaded agent memory file (`AGENTS.md` / `REASONIX.md` / `CLAUDE.md`) is being generated or has grown bloated and `codebase-memory-mcp` is available. Written generically — substitute your project's specifics.

## Goal

Generate or shrink the always-loaded memory file by moving **reference knowledge** (stack facts, domain specs, constants, invariants, endpoint allowlists, deployment facts, directory trees) into on-demand stores (memory-cache), while keeping **behavioural rules** (approval gates, skill selection, commit rules, build pipeline) load-bearing in the memory file.

### Context Economics & The "Dumb Zone"
- **Always-Loaded Tax**: Every token in `AGENTS.md` / `CLAUDE.md` is re-sent on *every interaction turn*. A 4,000-token file across a 40-turn session consumes 160,000 tokens purely in instruction overhead.
- **The Dumb Zone**: As session context exceeds 100k–200k tokens, agent reasoning degrades (loss of negative constraints, hallmarked omissions, erratic tool calls).
- **Budget Target**: Keep root memory files under **150–200 lines** (~1.5k–2k tokens). Disclose all reference material behind sharp context pointers.
- **Compaction Dementia**: Never execute slimdowns across an auto-compacted session (`/compact`). Auto-compaction drops subtle invariant rules. Start a fresh session before Step 3.

## Critical: Memory-Cache Setup (`codebase-memory-mcp`)

Setting up the persistent memory-cache correctly is essential. Reference knowledge is useless if the agent forgets it across sessions or if teammates/CI cannot access it.

| Capability | Holds | Retrieval |
| --- | --- | --- |
| Code graph (auto-indexed) | Files, folders, modules, routes, functions, callers/callees | `search_graph` / `get_architecture` / `trace_path` |
| `manage_adr` | One sectioned ADR document per project, persists across sessions | `manage_adr(mode='get')` — on demand |
| Markdown File nodes | `.md` files indexed as File nodes, grep-searchable | `search_code` |

- **Machine-Local Cache**: The live graph DB lives in `~/.cache/codebase-memory-mcp/` — absent on a fresh clone.
- **Persistent Artifact (`.codebase-memory/`)**: Executing `index_repository(persistence=true)` writes `.codebase-memory/graph.db.zst` (+ `artifact.json`, `.gitattributes`) **into the repo**. **You MUST commit these** so teammates/CI bootstrap from the artifact instead of re-indexing.
- **ADR Mutation Warning**: `manage_adr(mode='update')` **mutates `graph.db.zst`**. You must seed the ADR *before* committing the `.codebase-memory/` artifact, or expect a follow-up commit.

## Step 0 — Verify Memory-Cache is Configured

1. **Find the registration** — check, in order: project `.kimi-code/mcp.json`, project `.mcp.json`, global `~/.reasonix/config.toml` `[[plugins]]`. Validate JSON: `python3 -m json.tool <file>`.
2. **Confirm the binary** — `<command> --version`; then a **stdio handshake probe**:
   ```bash
   printf '%s\n' \
     '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2024-11-05","capabilities":{},"clientInfo":{"name":"probe","version":"0"}}}' \
     '{"jsonrpc":"2.0","id":2,"method":"tools/list","params":{}}' \
     | <command> 2>/dev/null | head -c 400
   ```
3. **Confirm the index binds THIS repo** — `list_projects` → `root_path` must equal `realpath` of the workspace.
4. **Confirm freshness** — `index_status` → status `ready`/`indexed`; check `check_index_coverage` `metadata.generation_matches: true`.
5. **Functional probe** — `search_graph` for a known symbol returns correct file:line.
6. **Session-liveness caveat** — MCP servers registered mid-session only join **new sessions**. Defer indexing/ADR to next session via an open task in `docs/TASKS.md`.

## Step 1 — Investigate (measure before moving)

```bash
wc -c AGENTS.md
grep -n "^## " AGENTS.md
awk '/^## 1\./,/^## 2\./' AGENTS.md | wc -c
```

1. Classify every section: **BEHAVIORAL** (keep) · **REFERENCE** (offload) · **MIXED** (split facts out, keep rules).
2. **Migration Manifest**: Compile an inventory table before touching files.

## Step 2 — Decide

1. **Storage**: `docs/ mirror + ADR` (recommended — canonical in git, ADR is the runtime graph copy) · `ADR-only` (max shrink, but not versioned) · `docs/-only` (no ADR).
2. **Scope**: `reference data only` · `reference data + compress rules` (bigger cut, more edit risk).
3. **Workspace Isolation**: Recommend executing in an isolated Git worktree (`git worktree add .worktrees/slimdown -b chore/agents-slimdown`).

## Step 3 — Execute Memory-Cache Generation

1. **Create canonical git-versioned files** — e.g. `docs/reference/{project-stack}.md`. Move content **verbatim**.
2. **Rewrite the memory file** —
   - Add a "Knowledge Map" table using **sharp context pointers** (e.g., "Consult `docs/reference/api.md` ONLY when modifying REST endpoints").
   - Ban eager transclusion (never use `@import` in `CLAUDE.md`).
   - Keep and compress behavioural rules.
3. **Generate the Persistent Memory Cache (CRITICAL)** —
   ```
   index_repository(repo_path=<workspace realpath>, mode="full", persistence=true)
   ```
   Record `nodes`/`edges`/`skipped_count`. Inspect `parse_partial` files.
4. **Seed the ADR** — `manage_adr(mode='update')` with the same content (the runtime graph copy). Do this **before** committing — the write mutates `.codebase-memory/graph.db.zst`.
5. **Verify** —
   - **Section-level verbatim diff**.
   - Constants preserved verbatim: `grep -F "embedding dims **768**" docs/reference/*.md`.
   - Graph probes: `search_graph`, `trace_path`, `check_index_coverage`.
   - ADR round-trip: `manage_adr(mode='get')`.
   - Every path named in the new Knowledge Map resolves.
   - `git status --short` shows only intended files.

## Step 4 — Commit (conventional)

Order matters (ADR writes mutate the artifact):

```bash
git add AGENTS.md docs/reference/ <tracker files> .kimi-code/mcp.json   # slim-down commit
git commit -m "docs(agents): generate AGENTS.md and offload reference knowledge"
# ... next session: index + seed ADR, then:
git add docs/TASKS.md .codebase-memory/
git commit -m "chore(repo): configure memory-cache into graph and seed ADR"
```

Update the project task tracker in the same commits.

## Pitfalls

- **The `@import` Transclusion Trap** — Never use `@import docs/reference/...` in `CLAUDE.md`. Claude Code eagerly injects imported files into context.
- **Compaction Loss** — If an agent session auto-compacts mid-task, verify offloaded pointers are still active.
- **Subagents Lack MCP Access** — Subagents do not automatically inherit parent MCP sessions. Pass exact file paths instead.
- **Never offload behavioural protocols** — (approval gates, commit rules, skill-selection) must stay in context.
- **`docs/` is canonical; the ADR is a mirror** — `.codebase-memory/graph.db.zst` artifact is for bootstrapping.
- **The ADR write mutates the graph artifact** — seed before committing `.codebase-memory/`.
