# Context Sandbox + FTS5 Tool-Output Indexing (MCP)

## Source
- Stanley Ulili, "Context Mode: Reducing AI Context Bloat with an MCP Server" (Mar 23, 2026) — https://betterstack.com/community/guides/ai/context-mode-mcp/
- GitHub repo (implementation): mksglu/context-mode — https://github.com/mksglu/context-mode

## Problem
Agentic coding/RCA workflows frequently call tools (filesystem reads, web fetches, browser snapshots, logs). The raw tool outputs are injected into the LLM context window and then re-sent on every subsequent request. This causes:
- Rapid context-window consumption (token bloat)
- Frequent compactions that erase critical state (files edited, decisions, errors)
- Higher latency/cost because large prompts are repeatedly processed

## Solution
Insert an MCP "context gateway" server between the agent and tools that:

1) Intercepts tool calls (via platform hooks like PreToolUse/PostToolUse) and runs them in a sandboxed subprocess.
2) Captures raw output and applies size-based handling:
   - If output is small (e.g., < ~5KB), return a short summary.
   - If output is large, chunk + index locally in a per-project SQLite database using FTS5 (BM25 retrieval).
3) Returns to the agent only:
   - A confirmation that the artifact was indexed, plus a lightweight summary
   - Or query-scoped snippets via dedicated MCP tools (e.g., ctx_search, ctx_batch_execute, ctx_fetch_and_index in Context Mode).
4) Persists session events (file edits/reads/creates, git ops, tasks, errors, decisions) into the same local store.
5) On agent compaction events, generates a priority-tiered session snapshot (typically <2KB) and injects it into the fresh context so the agent can resume coherently.

Concrete implementation details (from sources):
- Storage: local SQLite + FTS5
- Retrieval: BM25 over indexed tool outputs + event log
- Hooking: Claude Code plugin hooks (SessionStart/PreToolUse/PostToolUse/etc.) or other clients via MCP configuration

## Trade-offs
- Local persistence / security: raw tool output is stored locally; requires governance for secrets/PII in logs and snapshots.
- Extra moving part: MCP server + hooks increase operational complexity (versioning, installation, debugging).
- Query friction: agent must learn to ask targeted queries instead of relying on full output in-context.
- Indexing cost: chunking/indexing adds CPU/IO overhead (but aims to reduce LLM token cost/latency overall).

## When to use / When NOT to use
Use when:
- Long-running agent sessions regularly hit context limits
- Tool outputs are large (logs, web snapshots, multi-file reads, issue lists)
- You need auditability/continuity across compactions

Avoid when:
- Workflows are short and tool outputs are small
- You cannot store tool outputs locally (compliance constraints) or cannot scrub secrets
- Your platform cannot support hooks/MCP reliably

## Implementation notes
- Works best where you can intercept tool calls (Claude Code plugin hooks; other clients via MCP hook configs).
- Minimal viable build:
  - An MCP server that wraps a small set of high-volume tools (read_file, run_shell, web_fetch)
  - SQLite FTS5 indexing + search(query)->snippets and fetch(docId, ranges) endpoints
  - Size threshold logic (small => summarize, large => index)
- Languages/libraries:
  - Node/TypeScript or Python for MCP server
  - SQLite (FTS5) bindings
- Approximate effort:
  - MVP: 2–5 days (1 engineer)
  - Production hardening (redaction, encryption-at-rest, lifecycle controls, telemetry): 2–4 weeks
