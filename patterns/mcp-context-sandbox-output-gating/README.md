# MCP Context Sandbox Output Gating

## Source
- **Stop Burning Your Context Window — We Built Context Mode** (Feb 26, 2026) — Mert Köseoğlu
- https://mksg.lu/blog/context-mode

## Classification
**NEW_PATTERN**

## Problem
Agent tool calls (e.g., MCP tools in Claude Code) often return **large raw outputs** (HTML pages, Playwright snapshots, logs, API payloads). These outputs are naively pasted into the LLM conversation context, quickly exhausting the context window and degrading performance over long sessions.

The source gives concrete examples (e.g., Playwright snapshot ~56KB; GitHub issues list ~59KB; access log ~45KB) and reports that with many tools active, a large portion of the context can be consumed before meaningful interaction.

## Solution
Insert a **gateway MCP server** between the agent and the tool ecosystem that: 
1) executes data-heavy operations inside an isolated sandbox,
2) keeps raw tool outputs *out of the LLM context*, and
3) returns only **small, task-specific stdout summaries/extracts**.

### Architecture / implementation details
- Add an MCP server (“Context Mode”) that provides tools such as:
  - `execute` / `batch_execute`: run scripts in an isolated subprocess; only **stdout** is returned to the agent.
  - `index` + `search`: build a local searchable knowledge base from markdown/URLs; raw content is stored locally; only **matching snippets/code blocks** are returned.
- Sandbox execution model:
  - Each tool call spawns an isolated subprocess boundary.
  - Multiple runtimes supported (as described in the source): JS/TS, Python, Shell, Ruby, Go, Rust, PHP, Perl, R.
  - Credential passthrough for CLIs (e.g., `gh`, `aws`, `gcloud`, `kubectl`, `docker`) so commands can run without dumping credentials into context.
- Knowledge base model (as described in the source):
  - Chunk markdown by headings while preserving code blocks.
  - Store in SQLite FTS5 with BM25 ranking + stemming.
  - `fetch_and_index`: fetch a URL, convert HTML → markdown, chunk, index; the raw page never enters the LLM context.
- Automation hook:
  - A “PreToolUse” hook auto-routes tool outputs through the sandbox so the user doesn’t change workflow.

## Trade-offs
- **Added moving part**: introduces a gateway service to run and maintain; failure modes now include gateway downtime/misconfiguration.
- **Observability/security**: sandbox must be hardened (filesystem/network restrictions, resource limits, auditing) since it executes code and accesses credentials via passthrough.
- **Information loss risk**: aggressive stdout-only summarization can omit details the model might have needed; requires careful tooling to support “drill-down” queries (e.g., fetch a specific log line range, extract one DOM node, etc.).
- **Latency overhead**: extra hop + subprocess execution overhead vs directly returning tool output (often offset by reduced LLM token processing).

## When to use / When NOT to use
### Use when
- Agents routinely ingest large artifacts (web snapshots, logs, CSVs, test outputs, issue lists).
- Long-running coding/research sessions hit context limits or degrade in quality over time.
- You can express tool results as **structured extracts** (counts, diffs, top-N, exact code blocks) instead of dumping full payloads.

### Don’t use when
- Your agent workflow already returns small, structured tool results (JSON summaries) and doesn’t bloat context.
- Tool outputs require full-fidelity inclusion in prompt (rare, but e.g., needing the entire legal contract text inline for reasoning).
- You can’t safely operate a sandboxed executor in your environment (strict compliance constraints) and can’t use a managed alternative.

## Implementation notes
- **Effort**: ~2–5 days for a minimal gateway (sandbox runner + output policies); 1–3 weeks for robust security hardening, observability, and IDE/agent integration hooks.
- **Languages**: gateway can be built in Node/TypeScript or Go (the referenced project uses an MCP server approach; sandboxes can be subprocess-based initially).
- **Key components**:
  - Sandbox runner (subprocess execution; stdout/stderr capture; time/memory limits).
  - Output gating policy (max-bytes/tokens; summarizers; redaction).
  - Local KB indexer (SQLite FTS5, chunker, optional URL fetcher).
  - Agent integration hook (pre/post tool call interception).

