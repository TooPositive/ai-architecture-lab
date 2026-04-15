# Context Gateway for Tool Outputs (MCP Output Sandbox)

## Source
- **Article:** Stop Burning Your Context Window — We Built Context Mode
- **URL:** https://mksg.lu/blog/context-mode

## Problem
Tool-using agents (e.g., MCP-based IDE agents) often stream **large, raw tool outputs** (HTML pages, logs, Playwright snapshots, issue lists, CSVs) directly into the LLM context window. This quickly consumes tokens, increases cost, and degrades session performance (e.g., hitting context limits / slowing down after prolonged sessions).

## Solution
Insert a **Context Gateway** between the agent and tools that:
1) Executes data-heavy tool calls inside an **isolated sandbox process**.
2) Keeps raw tool outputs *out of the conversation context*.
3) Returns only **small, targeted artifacts** to the LLM (e.g., computed summaries, extracted rows, exact code blocks, ranked search hits).

### Architecture (concrete)
- Agent calls tools through a gateway (implemented as an MCP server).
- Gateway provides two core capabilities:
  - **Sandboxed execution** for data processing:
    - Each `execute` spawns an isolated subprocess (process boundary).
    - Captures `stdout`; only `stdout` is returned into LLM context.
    - Supports multiple runtimes (article lists: JS/TS, Python, Shell, Ruby, Go, Rust, PHP, Perl, R).
    - Supports credential passthrough for authenticated CLIs (article lists: `gh`, `aws`, `gcloud`, `kubectl`, `docker`) by inheriting env/config, without copying raw outputs into the prompt.
  - **Local knowledge base indexing + retrieval** for fetched docs:
    - `fetch_and_index(url)` fetches a page, converts HTML→Markdown, chunks by headings while keeping code blocks intact, and indexes into **SQLite FTS5**.
    - Retrieval uses **BM25** ranking + Porter stemming; returns exact snippets/code blocks rather than LLM-generated summaries.

### Operational hook
- Add a *pre-tool-use hook* that automatically routes tool outputs through the gateway so agent authors don’t have to manually change every tool call.

## Trade-offs
- **Pros**
  - Huge reduction in tokens sent to the model (article claims 315 KB → 5.4 KB, ~98% reduction), improving session longevity and cost.
  - Reduces risk of leaking sensitive raw data into prompts/logs if the gateway enforces filtering and only returns derived outputs.
  - Enables deterministic transformations (parsing, aggregation) outside the LLM, improving reliability for tasks like CSV analytics and log triage.
- **Cons**
  - More moving parts: an MCP server + sandbox runtime management + indexing store (SQLite FTS5).
  - Debugging becomes two-layered (agent prompt vs. gateway execution).
  - If the gateway returns overly-compressed results, the LLM may lose necessary detail; requires careful design of “return contracts” (what to return, size limits, and how to request more).

## When to use / When NOT to use
- **Use when**
  - Agents regularly call tools that return large payloads (browser snapshots, logs, API responses, repo scans).
  - You have long-running interactive sessions (coding agents) where context pressure accumulates.
  - You want a clean separation between raw data handling and LLM reasoning.
- **Do NOT use when**
  - Tool outputs are already small and structured (e.g., a few JSON fields) and you benefit from direct grounding in raw outputs.
  - You cannot safely run sandboxed subprocesses (strict runtime restrictions) and don’t have an alternative isolation mechanism (containers/VMs).

## Implementation notes
- **Implementation surface:** MCP server (Node.js via `npx` is shown in the article install instructions).
- **Data plane:** subprocess sandbox + stdout capture; optionally implement per-tool output budgets (bytes/tokens).
- **Indexing:** SQLite FTS5 + chunking by headings; preserve code blocks intact.
- **Approx effort:**
  - Prototype: 2–5 days (single runtime + stdout filtering + 1–2 tools).
  - Production: 2–4 weeks (multi-runtime sandboxing, auth passthrough hardening, telemetry, policies, redaction, tests).
