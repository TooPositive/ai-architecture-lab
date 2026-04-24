# Context Gateway MCP Sandbox (Context Mode)

## Source
- **Stop Burning Your Context Window — We Built Context Mode** (Feb 26, 2026) — Mert Köseoğlu
- https://mksg.lu/blog/context-mode

## Problem
Tool-using agents (e.g., MCP-based) often stream *large raw tool outputs* (logs, HTML, screenshots, CSVs, diffs) directly into the LLM context window. This quickly consumes context, causing degraded reasoning quality, forced truncation, and higher cost.

## Solution
Insert a **Context Gateway** between the agent runtime and tools that:
1. **Runs tools in an isolated sandbox** and keeps raw outputs *outside* the LLM conversation context.
2. **Returns only small, purpose-built stdout summaries** (or extracted slices like exact matching lines / code blocks) back to the LLM.
3. Optionally provides a **local searchable knowledge base** (e.g., SQLite FTS5 + BM25) for indexed artifacts so the LLM can request *targeted fetches* rather than ingesting everything.

Concrete implementation described in the source (Context Mode):
- An **MCP server** sits between Claude Code and tool outputs.
- Each `execute` spawns an **isolated subprocess** (process boundary); only captured `stdout` enters context. Raw files/responses remain in the sandbox.
- Provides multiple runtimes (JS/TS/Python/Shell/Ruby/Go/Rust/PHP/Perl/R).
- Knowledge base tool:
  - `index`: chunk Markdown by headings **while keeping code blocks intact**
  - store chunks in **SQLite FTS5**
  - search via **BM25 ranking** with **Porter stemming**
  - return **exact code blocks + heading hierarchy** (not summaries)
  - `fetch_and_index`: fetch URL → convert HTML→Markdown → chunk → index; raw page never enters context
- Includes a **PreToolUse hook** to auto-route tool outputs through the sandbox.

Reported impact numbers (from the source):
- Playwright snapshot: **56 KB → 299 B**
- GitHub issues (20): **59 KB → 1.1 KB**
- Access log (500 req): **45 KB → 155 B**
- Session aggregate: **315 KB → 5.4 KB (98% reduction)**
- Session time before slowdown: **~30 min → ~3 hours**

## Trade-offs
- **Engineering complexity**: you must run and secure a local/sidecar MCP server plus sandboxed runtimes.
- **Loss of raw-data transparency**: the model no longer “sees everything”; you must design stdout contracts and retrieval tools carefully.
- **Potential debugging friction**: developers may need a way to inspect sandbox artifacts out-of-band when the summarization hides important details.
- **Security surface**: executing arbitrary code/subprocesses needs strong isolation (least-privilege FS access, env passthrough controls, audit logs).

## When to use / When NOT to use
**Use when**
- Your agent makes frequent tool calls that return large payloads (browser snapshots, logs, API JSON, diffs).
- You see context window pressure as a primary failure mode (truncation, increasing hallucinations, rising prompt costs).
- You can constrain tool outputs into small, structured summaries (tables, counts, extracted lines).

**Avoid when**
- Tool outputs are already small or highly variable such that summarization routinely removes critical nuance.
- You cannot operate sandbox infrastructure securely (e.g., strict compliance environments without an approved execution sandbox).

## Implementation notes
- **Languages**: Node.js (MCP server), plus multiple runtimes for sandbox execution; SQLite for FTS5.
- **Libraries/tech**: MCP SDK, SQLite FTS5 + BM25, Markdown chunker preserving code blocks, HTML→Markdown converter.
- **Effort**:
  - Prototype gateway (stdout-only contract + artifact storage): ~1–3 days
  - Add indexing/search and safe fetch: ~3–7 days
  - Harden isolation/observability/ops: 2–4+ weeks depending on environment
