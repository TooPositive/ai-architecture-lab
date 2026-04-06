# Pattern: MCP output sandbox for context compression ("Context Mode")

## Source
- Mert Köseoğlu. **"Stop Burning Your Context Window — We Built Context Mode"** (Feb 26, 2026)
  https://mksg.lu/blog/context-mode

## Problem
Tool-using agents (e.g., MCP/Claude Code) often inject **raw tool output** (HTML pages, Playwright snapshots, logs, issue lists, CSVs) into the LLM context window. This causes:
- Rapid context exhaustion (large, repeated payloads).
- Higher cost + latency (more tokens processed each turn).
- Degraded performance over long sessions due to bloated context.

## Solution
Insert a **Context Gateway (MCP server)** between the agent and tools, so raw outputs never enter the model context. Instead:
1. **PreToolUse hook** intercepts tool calls and routes them through a sandbox.
2. Each call runs inside an **isolated subprocess** (process boundary) that can access files/CLIs, but only returns a **small, curated stdout** back to the agent.
3. For knowledge/doc retrieval, the gateway indexes fetched content in a local store (SQLite FTS5 + BM25) and returns **exact relevant snippets** rather than full documents.

Concrete implementation details from the source:
- Execution sandbox: *"Each execute call spawns an isolated subprocess"*; captures stdout; only stdout enters conversation.
- Available runtimes include JS/TS/Python/Shell/Go/Rust/etc; Bun auto-detected for faster JS/TS execution.
- Credential passthrough for authenticated CLIs (gh/aws/gcloud/kubectl/docker) by inheriting env + config paths.
- Knowledge base tool: chunk markdown by headings while keeping code blocks intact; store in **SQLite FTS5**; search with **BM25**; apply **Porter stemming** at index time.
- fetch_and_index: fetch URL → convert HTML→markdown → chunk → index; raw page never enters context.

## Trade-offs
- **Operational complexity**: you now run and version an MCP server + sandbox runtimes.
- **Debuggability**: when the LLM can’t see raw output, you must provide a way to inspect full artifacts out-of-band (logs/artifact storage).
- **Risk of over-compression**: returning only stdout/snippets can omit details; requires careful tool design (e.g., add "show more" or artifact links).
- **Security boundary**: subprocess isolation helps, but you must still harden execution (resource limits, allowlists, filesystem/network policy).

## When to use / When NOT to use
**Use when:**
- Agents call tools that return large payloads (browser snapshots, API responses, logs).
- Long interactive sessions where context bloat is the dominant failure mode.
- You can tolerate a gateway component and want predictable context usage.

**Do NOT use when:**
- Your tools already return compact structured outputs (<1–2KB typical).
- You need the LLM to reason over full raw artifacts inline (e.g., full legal contract review) and cannot add an out-of-band artifact viewer.
- Your environment forbids code execution sandboxes.

## Implementation notes
- Interface: MCP server with a **PreToolUse hook** (as described in the source).
- Storage: SQLite FTS5 + BM25; add optional vector store later if semantic retrieval needed.
- Sandbox: spawn per-call subprocess; enforce timeouts/memory limits; return structured JSON + short human summary.
- Effort: ~2–7 days for a minimal gateway (hook + subprocess + truncation rules); longer if adding indexing + artifact viewing UI.

## Classification
**NEW_PATTERN** (this is more specific than generic “summarize tool output”: it’s an MCP-side architecture with a sandbox + FTS index so raw outputs never enter context, with published measured reductions).
