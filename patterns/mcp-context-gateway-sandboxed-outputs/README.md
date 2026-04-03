# Pattern: MCP Context Gateway (Sandboxed Tool Outputs + FTS Retrieval)

## Source
- **Stop Burning Your Context Window — We Built Context Mode** (Feb 26, 2026), Mert Köseoğlu
  https://mksg.lu/blog/context-mode
- Reference implementation (MIT): https://github.com/mksglu/context-mode

## Classification
**NEW_PATTERN** — The explicit “output-side context virtualization” via an MCP proxy that executes tools in a sandbox and only returns tiny, computed stdout + uses local FTS (BM25) retrieval for session continuity is not yet a widely standard, codified architecture pattern (as of the article’s publication).

## Problem
Tool-using agents (via MCP) quickly exhaust the LLM context window because:
- tool definitions consume a large fixed prefix
- *raw tool outputs* (logs, HTML, snapshots, issue lists, CSVs) get pasted into the conversation

This causes:
- frequent compaction (loss of working memory, file/task state)
- degraded model performance and higher cost/latency due to long prompts

## Solution
Insert an **MCP “context gateway”** between the agent runtime (e.g., Claude Code) and high-output tools. The gateway enforces two mechanisms:

1) **Sandboxed execution for high-output tools**
- The gateway intercepts tool calls (via hooks such as PreToolUse/PostToolUse).
- It runs the requested operation in an isolated subprocess “sandbox”.
- Raw artifacts stay outside the LLM context (files, API payloads, snapshots).
- Only a compact stdout/result (often <1KB) is returned to the LLM.

2) **Local index + retrieval instead of “dumping memory back into context”**
- All high-volume state (tool events, file edits, tasks, errors, decisions) is persisted to local SQLite.
- A SQLite FTS5 index is built; retrieval uses **BM25** ranking and returns *exact snippets / code blocks* with hierarchy (not summaries).
- When the conversation compacts, the system doesn’t re-inject everything; it rehydrates only what’s relevant via search.

### Concrete implementation details (from Context Mode)
- Sandbox spawns an isolated subprocess per `execute` call; captures stdout only.
- Supports multiple runtimes (JS/TS/Python/Shell/Ruby/Go/Rust/PHP/Perl/R).
- Credential passthrough for CLIs (e.g., `gh`, `aws`, `gcloud`, `kubectl`) via inherited env/config (avoid copying secrets into chat).
- Knowledge base indexing chunks markdown by headings while keeping code blocks intact, stored in SQLite FTS5.
- URL ingestion: fetch → convert HTML to markdown → chunk → index; raw page never enters context.

## Trade-offs
- **Extra hop / operational surface**: You now run and secure an MCP server + hook logic (versioning, upgrades, compatibility with clients).
- **Debuggability**: When things go wrong, you debug sandbox execution + hook routing, not just prompts.
- **Loss of “raw visibility”**: Engineers may want to see the full raw output in chat. You’ll need a drill-down mechanism (e.g., open artifact files locally, or an opt-out mode).
- **Tool parity limitations**: Some tools require interactive sessions or large structured payloads; stdout-only may be insufficient without designing additional “summary” outputs.
- **Security boundary needs care**: Credential passthrough and subprocess isolation must be hardened (filesystem/network allowlists, per-tool permissions).

## When to use / When NOT to use

### Use when
- Your agent frequently calls tools that return large payloads (browser snapshots, logs, database dumps, issue lists, API responses).
- You run long agent sessions where compaction causes task loss and repeated work.
- You need cost and latency control from shrinking prompts.
- You can tolerate an additional local/sidecar service in the developer workflow or agent runtime.

### Don’t use when
- Your tools already return small structured outputs (few KB) and sessions are short.
- You require full raw outputs inside the LLM for downstream reasoning and cannot summarize safely.
- You can’t operate a sandboxed execution environment securely (multi-tenant, untrusted code, strict compliance constraints) without significant investment.

## Implementation notes
- **Languages**: Node/TypeScript (Context Mode uses Node; sandbox runs multiple runtimes).
- **Storage**: SQLite + FTS5 virtual tables.
- **Ranking**: BM25 via SQLite FTS5 (built-in).
- **Hooks**: Requires an agent runtime that supports MCP hooks (e.g., Claude Code plugin hooks; Gemini CLI hooks).
- **Effort**:
  - Prototype (single tool + sandbox + stdout capture): 1–2 days
  - Full workflow integration with hooks + multi-runtime + indexing: 1–3 weeks

## Notes on benchmarkability
The source provides concrete compression ratios (e.g., 315KB→5.4KB, 98% reduction) but does not provide a standardized, apples-to-apples measurement against a published RAG baseline. Treat as a **pattern** first; you can benchmark downstream improvements in session length, latency, and cost in your environment.
