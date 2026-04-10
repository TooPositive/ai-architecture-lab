# MCP Context Compression Proxy (Tool-Output Compaction Layer)

## Source
- **"Stop Burning Your Context Window — We Built Context Mode"** by Mert Köseoğlu (Unwind AI) 
  - https://mksg.lu/blog/context-mode

## Problem
In MCP-style agent stacks (e.g., Claude Code + MCP servers), tool results often stream back **raw, bulky artifacts** (DOM snapshots, logs, issue lists, API payloads). These outputs are appended into the LLM context verbatim, rapidly consuming the context window, increasing cost, and degrading reasoning due to context bloat.

The source quantifies this failure mode (e.g., 315KB tool output → 5.4KB after processing; 98% reduction).

## Solution
Insert a **Context Compression Proxy** between the agent runtime and MCP tools to compact tool outputs before they reach the model:

1. **Interpose at the MCP boundary**
   - Agent calls tools as usual.
   - Proxy forwards the request to the real MCP server(s).

2. **Compress tool responses**
   - Apply schema-aware compaction rules per tool type (Playwright snapshot, GitHub issues, logs).
   - Emit:
     - a short summary
     - key fields
     - stable identifiers / links / hashes for later retrieval

3. **Add recall hooks**
   - Store the raw tool output in external storage (disk/blob/db) keyed by `artifact_id`.
   - Provide a `recall(artifact_id, selector)` tool so the agent can fetch *only* the needed slices later.

4. **Budget-aware behavior**
   - Configure max-bytes or max-tokens per tool result.
   - If output exceeds budget, force summarization + storage.

### Concrete implementation
- Run an MCP server that wraps other MCP servers:
  - `proxy_tool_call(name, args) -> raw_result`
  - `compress(name, raw_result) -> compact_result`
  - `store(raw_result) -> artifact_id`
  - `recall(artifact_id, query) -> slice`
- Compaction strategies:
  - Logs: dedupe lines, keep anomalies, keep last N lines, compute counts.
  - DOM/snapshots: keep text nodes, key selectors, error banners, actionable elements.
  - Lists (issues/tickets): keep titles, states, top labels, links; drop bodies unless requested.

## Trade-offs
- **Pros**
  - Dramatically improves **context utilization** and reduces cost.
  - Improves agent stability: less distraction from irrelevant raw payload noise.
  - Enables long-running sessions without hitting context limits.
- **Cons**
  - Risk of losing details needed for edge cases (mitigated via recall tool).
  - Added system complexity: storage, artifact lifecycle, access control.
  - Requires tool-specific compaction logic to avoid over-summarizing.

## When to use / When NOT to use
- **Use when**
  - Tools return large payloads (browser automation, repos/issues, observability logs).
  - You run long agent sessions and see cost/context blow-ups.
  - You can tolerate a small extra hop for big reductions in tokens.
- **Do NOT use when**
  - Tool output is already compact and structured.
  - You cannot store raw artifacts for compliance reasons (or cannot secure them).

## Implementation notes
- **Languages**: any; MCP servers are commonly built in TypeScript/Node or Python.
- **Storage**: filesystem, S3/Azure Blob, or a DB keyed by `artifact_id`.
- **Effort**: 1–3 days for a useful proxy with 3–5 tool-specific compressors + recall.
