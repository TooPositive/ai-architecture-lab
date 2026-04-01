# Context-Mode Output Sandbox (MCP)

## Source
- **Stop Burning Your Context Window — We Built Context Mode** (Mert Köseoğlu, Feb 26 2026)
- https://mksg.lu/blog/context-mode

## Problem
Agentic workflows (e.g., Claude Code via MCP) often **dump large, raw tool outputs** (logs, API responses, DOM snapshots, CSVs) directly into the LLM context window. This rapidly exhausts context, increases latency/cost, and degrades session quality over time.

## Solution
Insert an **MCP “output sandbox” gateway** between the agent and its tools so that:
1. Tool execution happens in an **isolated subprocess** (process boundary).
2. The subprocess can access local files/CLIs via credential passthrough, but **raw artifacts never enter the model context**.
3. Only a **small, task-specific stdout extract** (e.g., computed stats, filtered lines, exact code blocks) is returned to the LLM.

Concrete implementation (as described in the source):
- Provide an MCP server with a `batch_execute` / `execute` style tool.
- For each tool call, spawn an isolated subprocess in a sandbox; capture **stdout only** and return it.
- Add a pre-tool hook (e.g., `PreToolUse`) to **auto-route** tool calls through the sandbox without changing user behavior.
- Include a “knowledge base” tool that indexes Markdown into **SQLite FTS5** and returns exact matching blocks (not summaries), so the agent can retrieve without pasting raw docs.

## Trade-offs
- **Loss of raw fidelity in-context**: if you later need the full log/snapshot inside the LLM, you must explicitly fetch a larger excerpt (or re-run).
- **Sandbox complexity**: managing process isolation, allowed runtimes, and security boundaries adds operational burden.
- **Debuggability shift**: failures may require inspecting sandbox-side artifacts/logs rather than the LLM transcript.
- **Tooling constraints**: some tools may require interactive output or binary artifacts that don’t map cleanly to stdout.

## When to use / When NOT to use
**Use when:**
- Your agent relies on tools that emit large outputs (browser snapshots, issue lists, logs, test runs, CSV analytics).
- Long-running sessions degrade as context fills (multi-hour developer agent sessions).
- You want deterministic extracts (exact code blocks, filtered lines) rather than model summaries.

**Do NOT use when:**
- Your workflow requires the model to “see” large raw artifacts holistically (e.g., full document review inside the LLM).
- You can’t safely sandbox (strict OS constraints) or can’t allow subprocess execution.
- You have very small tool outputs already (no meaningful savings).

## Implementation notes
- **Languages/runtimes** (per source): JS/TS, Python, Shell, Ruby, Go, Rust, PHP, Perl, R.
- **Storage/search**: SQLite FTS5 with BM25 ranking; heading-based chunking for Markdown; keep code blocks intact.
- **Approx effort**:
  - Prototype MCP server + subprocess runner: ~1–3 days.
  - Hooking into an agent (pre-tool routing), plus safe allowlist + logging: ~3–7 days.
  - Hardening (security, quotas, OS portability): 1–3+ weeks.
