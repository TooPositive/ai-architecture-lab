# Context Virtualization via MCP Sandboxing + Event Indexing

## Source
- **Context Mode** (GitHub) — "Context Mode: The other half of the context problem" — https://github.com/mksglu/context-mode

## Problem
Agentic IDE/tooling workflows (MCP) often stream large tool outputs (DOM snapshots, logs, search results) directly into the LLM context. This quickly consumes the context window and forces compaction/summarization that can drop critical working-state (what files were edited, current tasks, last errors).

## Solution
Insert a **context virtualization layer** between the agent and its tools/outputs:

1. **Sandbox tools for high-volume outputs**
   - Route large-output tool calls through a set of "ctx_*" sandbox tools (e.g., `ctx_execute`, `ctx_execute_file`, `ctx_batch_execute`) that capture raw outputs **outside** the LLM prompt.
   - Only return a small, structured synopsis/handle to the LLM (e.g., tool name, status, output size, artifact IDs).

2. **Event-sourced session continuity store**
   - Persist tool interactions + agent actions (file edits, git ops, tasks, errors, decisions) into **SQLite** as an append-only event log.
   - Index events with **FTS5** and retrieve relevant snippets using **BM25** at run time, instead of re-injecting full history during compaction.

3. **Hook-based enforcement**
   - Use platform hooks (e.g., `PreToolUse`, `PostToolUse`, `PreCompact`, `SessionStart`) to automatically:
     - Detect tools likely to produce large output (shell, file reads, web fetch, grep/search).
     - Rewrite/reroute them to sandbox equivalents.
     - On compaction, prevent dumping raw tool outputs back into context; rely on retrieval from the indexed store.

4. **Retrieval on demand**
   - Provide MCP endpoints like `ctx_search` (BM25 over FTS5) + `ctx_fetch_and_index` to bring back only the minimal set of events/artifacts needed for the current step.

**Concrete implementation (from source):**
- Claims "Context Saving": **315 KB → 5.4 KB (98% reduction)** by keeping raw tool output out of context.
- "Session Continuity": track actions in SQLite, index with FTS5, retrieve via BM25 search.
- Hook targeting: match only high-output tools to minimize overhead while intercepting context-bloat sources.

## Trade-offs
- **Complexity & moving parts:** MCP server + hooks + local DB (SQLite/FTS5) increases deployment surface area vs a plain agent.
- **New failure modes:**
  - If the SQLite store is corrupted/unavailable, the agent may lose continuity and need fallback prompts.
  - Hook misconfiguration can cause raw outputs to leak into context or break tool routing.
- **Privacy/security considerations:** tool outputs are stored locally; must manage retention (source notes previous session data deleted unless continued).
- **Search quality limits:** BM25 over event text is fast and good for keyword recall, but may miss semantically related context unless augmented.

## When to use / When NOT to use
**Use when:**
- IDE agents (Claude Code/Cursor/Copilot) routinely handle large tool outputs (DOM snapshots, logs, repository scans).
- You see frequent forced compactions and degraded statefulness.
- You can run a local MCP server and configure hooks.

**Avoid when:**
- Workflows are short-lived or tool outputs are already small/structured.
- You cannot install hooks or local services (locked-down environments).
- You require cloud-only storage and cannot persist outputs locally.

## Implementation notes
- **Languages/stack (source repo):** Node/TypeScript MCP server; SQLite + FTS5; BM25 retrieval.
- **Integration effort:**
  - 0.5–1 days to try MCP-only mode (manual tool selection).
  - 1–2 days to enable hook-based enforcement for supported clients.
  - 2+ days to adapt routing rules + event schema to a custom agent/tooling stack.
- **Operational notes:**
  - Implement a routing matcher list for "high-output" tools.
  - Add a `ctx_doctor`-like health check (FTS5 enabled, hooks installed, DB writable).
  - Add retention controls and session IDs/correlation IDs.
