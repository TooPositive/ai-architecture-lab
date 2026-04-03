# Context Mode Output Sandbox (MCP)

## Source
- **Stop Burning Your Context Window — We Built Context Mode** — Mert Köseoğlu (Feb 26, 2026)  
  https://mksg.lu/blog/context-mode

## Problem
Agent tool calls (especially via MCP) often return **large raw outputs** (HTML pages, logs, screenshots, issue lists, CSVs) that get pasted directly into the LLM context window. This causes:
- Rapid context exhaustion (e.g., long sessions degrade quickly)
- Higher cost/latency due to larger prompts
- Lower answer quality from irrelevant verbosity

## Solution
Insert a **context gateway MCP server** between the agent and tools that:
1. Executes heavy data processing in a **sandboxed subprocess**.
2. Keeps raw outputs **out of the LLM context**.
3. Emits only **small, task-specific stdout** (or exact relevant snippets like code blocks) back to the LLM.

### Architecture (concrete)
- Run an MCP server (“Context Mode”) that provides:
  - `execute` / `batch_execute`: run scripts in isolated subprocesses; only stdout is returned to the agent.
  - `index`: chunk markdown by headings (preserve code blocks) and store in **SQLite FTS5**.
  - `search`: query the FTS5 index; return *exact* matching code blocks + heading hierarchy (BM25-ranked).
  - `fetch_and_index`: fetch URL → convert HTML→markdown → chunk → index.
- Add a **PreToolUse hook** that automatically routes tool outputs through the sandbox so the agent workflow doesn’t change.
- Support multiple runtimes (the source lists JS/TS/Python/Shell/Ruby/Go/Rust/PHP/Perl/R).

### Example implementation sketch
- If a tool returns a 50KB blob (e.g., Playwright snapshot), instead of inserting it into context:
  - write it to a temp file inside sandbox
  - run a script to extract exactly what’s needed (e.g., the failing selector, the HTTP status, a table of errors)
  - return <1KB derived output

## Trade-offs
- **Operational complexity**: you now run an MCP server + sandbox runtime environment.
- **Debuggability**: raw tool outputs are not in the prompt transcript; you must rely on sandbox logs/artifacts.
- **Security surface**: executing code/subprocesses requires hardening (filesystem/network restrictions, secrets handling).
- **Potential loss of nuance**: if extraction scripts are wrong, the LLM may miss critical details that were present in the raw output.

## When to use / When NOT to use
### Use when
- You have long-running agent sessions (IDE agents, ops copilots) where tool output bloat is a primary failure mode.
- Your tools routinely return large artifacts (browser snapshots, logs, CSVs, issue lists, API JSON).
- You can define deterministic extraction steps (parsing logs, summarizing tabular data, grepping code).

### Don’t use when
- Tool outputs are already small (<1–2KB) and you benefit from full transparency in the LLM transcript.
- You cannot trust a sandboxed transformation layer (e.g., compliance requires raw evidence in the conversation log).
- Your environment cannot safely run subprocess sandboxes.

## Implementation notes
- **Typical stack**: Node.js MCP server + subprocess execution.
- **Storage**: SQLite FTS5 for local searchable knowledge base.
- **Effort**:
  - MVP (single agent, a few tools): ~1–2 days
  - Production hardening (RBAC, isolation, audit logs, artifact retention): ~1–3 weeks
