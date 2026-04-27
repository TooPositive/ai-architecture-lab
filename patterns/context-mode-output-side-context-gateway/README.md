# Context Mode (Output-side Context Gateway)

## Source
- **Article**: "Stop Burning Your Context Window — We Built Context Mode" (Mert Köseoğlu, Feb 26, 2026)
- **URL**: https://mksg.lu/blog/context-mode
- **Code**: https://github.com/mksglu/context-mode

## Problem
Tool-using agent sessions (e.g., Claude Code via MCP) quickly exhaust the LLM context window because every tool call injects:
1) large tool definitions (input-side), and
2) large raw tool outputs (output-side: logs, snapshots, API responses, CSVs).

This causes early context saturation, degraded reasoning, and shorter productive sessions.

## Solution
Insert an **MCP “context gateway” server** between the agent runtime and tool outputs that:
- Executes tool interactions in an **isolated sandbox subprocess**
- Prevents raw outputs from entering the LLM context
- Returns only **small, task-specific stdout artifacts** (e.g., extracted lines, computed summaries, exact code blocks)
- Adds an optional **local knowledge base** that indexes fetched content and returns exact snippets on demand

### Architecture / Flow
1. **Agent (Claude Code)** invokes tools as usual.
2. A **PreToolUse hook** routes tool calls through Context Mode.
3. Context Mode runs the operation inside a **sandbox subprocess** (separate process boundary), capturing stdout only.
4. The raw data remains outside the prompt (stored on disk / in process), while the model receives a tiny response.
5. For documentation/research tasks, Context Mode can **fetch → HTML→Markdown → chunk by headings (preserve code blocks) → index into SQLite FTS5**.
6. Retrieval is via **FTS5 BM25 ranking**; responses return exact code blocks and headings (not LLM summaries).

Concrete implementation details from the source:
- Sandbox supports multiple runtimes (JS/TS, Python, Shell, Ruby, Go, Rust, PHP, Perl, R).
- Authenticated CLIs can run via credential passthrough (e.g., `gh`, `aws`, `gcloud`, `kubectl`, `docker`).
- Knowledge base indexing uses **SQLite FTS5** with **BM25** ranking and **Porter stemming**; chunking is by Markdown headings while keeping code blocks intact.
- Reported effect: **315 KB → 5.4 KB** of tool output sent to context (**98% reduction**).

## Trade-offs
- **Extra moving part**: you now operate an MCP server plus hook integration; failures here can block tool use entirely.
- **Debuggability**: because raw outputs don’t enter the prompt, you need separate logging/inspection tooling to view full tool results when troubleshooting.
- **Risk of lossy reductions**: if the sandbox script extracts the wrong subset, the LLM may miss critical evidence; requires careful “extraction script” design per tool type.
- **Latency overhead**: subprocess spawn + extraction adds overhead, though the source argues net sessions run longer and avoid context-related slowdowns.
- **Security surface**: credential passthrough to subprocesses is powerful; sandbox isolation must be correct to avoid leaking secrets into stdout.

## When to use / When NOT to use
### Use when
- Your agent relies on tools that routinely return large blobs (browser snapshots, logs, issue lists, diffs, CSVs).
- You see context windows saturate mid-session and quality drops over time.
- You want repeatable, deterministic extraction (exact code blocks, exact log lines) instead of LLM summarization.

### Don’t use when
- Tool outputs are already small (<1–2 KB) and context isn’t a bottleneck.
- You need the model to directly reason over the full raw artifact inside the prompt (e.g., nuanced language in long legal text), and you don’t have a safe extraction strategy.
- You can’t tolerate introducing a middleware component into agent execution.

## Implementation notes
- **Languages**: Node.js (as distributed via `npx` in the article), plus multi-runtime sandbox execution.
- **Storage**: SQLite FTS5 for local indexing.
- **Effort**:
  - 0.5–1 day to pilot with one tool family (e.g., logs or GitHub issues).
  - 1–2 weeks to productionize (observability, fallbacks to raw output, policy for what may be suppressed).

## Confidence
High — directly grounded in the author’s article and GitHub repository.
