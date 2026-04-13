# Context Mode Output Sandbox (MCP Context Compression)

## Source
- **Stop Burning Your Context Window — We Built Context Mode** — Mert Köseoğlu (Feb 26, 2026)
- https://mksg.lu/blog/context-mode

## Problem
Agent/tool workflows (especially MCP-based) often stream large raw artifacts (HTML pages, logs, Playwright snapshots, GitHub issue lists, CSVs) directly into the LLM context. This rapidly consumes the context window, increases latency/cost, and degrades reasoning quality over long sessions.

## Solution
Insert an **MCP “context gateway” server** between the agent runtime and external tools. Route high-volume tool outputs into a **sandbox/executor subprocess** and only return **small, deterministic summaries or computed results** back into the LLM context.

Concrete architecture (as described in the source):
1. **PreToolUse hook** intercepts tool calls.
2. **Sandbox executor** runs code in an isolated subprocess; the *raw data* (files, API responses, snapshots) stays outside the prompt.
3. Only **stdout** (small) is returned to the LLM.
4. A **knowledge base tool** builds a local index for fetched content:
   - Convert HTML → Markdown
   - Chunk Markdown **by headings** while **keeping code blocks intact**
   - Store in **SQLite FTS5**
   - Query with **BM25** + stemming
   - Return exact matching sections / code blocks (not generated summaries)

Reported impact (from the source article):
- Session output volume reduced **315 KB → 5.4 KB** (~**98% reduction**)
- Session time before slowdown **~30 min → ~3 hours**

## Trade-offs
- **Engineering complexity**: you must operate an MCP gateway with hooks and an execution sandbox (process isolation, credential passthrough).
- **Debuggability**: hiding raw tool output from the LLM means you need good logging/trace tooling outside the prompt to diagnose failures.
- **Loss of “full-fidelity” context**: if the gateway returns overly aggressive summaries, the model may miss edge-case details; mitigated by providing targeted “fetch exact section” APIs.
- **Security surface**: credential passthrough to subprocesses must be carefully scoped; sandboxing reduces but doesn’t eliminate risk.

## When to use / When NOT to use
**Use when:**
- Long-running coding or ops sessions where tools emit large artifacts (browser snapshots, logs, CSV analytics, issue lists).
- You have many tools enabled (large tool schemas + large outputs) and hit context limits.
- You can tolerate an extra gateway hop to reduce LLM token usage.

**Avoid when:**
- Your tools already return compact, structured outputs (small JSON).
- You need the model to directly reason over the *entire* raw artifact (e.g., full legal text) and cannot rely on targeted retrieval.

## Implementation notes
- **Interface**: MCP server with interception hooks (e.g., a “PreToolUse” hook) to auto-route tool outputs.
- **Sandbox execution**: isolated subprocess per call; capture stdout only.
- **Index**: SQLite **FTS5** with **BM25** ranking; chunk by markdown headings; preserve code blocks.
- **Languages/runtimes** (from source): JS/TS/Python/Shell/Ruby/Go/Rust/PHP/Perl/R; optional Bun for faster JS/TS.
- **Effort**: ~2–5 engineering days for a minimal gateway + sandbox + a single “index/search” tool; more if you add robust security, tracing, and multi-tool routing.
