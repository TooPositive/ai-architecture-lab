# Context-Mode MCP Output Compression Gateway

## Source
- **"Stop Burning Your Context Window — We Built Context Mode" — Mert Köseoğlu**
  https://mksg.lu/blog/context-mode

## Problem
In tool-using agent setups (notably MCP-based), tool calls frequently return large payloads (HTML snapshots, issue lists, logs). Many clients naively paste these raw outputs into the LLM context on every turn, quickly consuming the context window and increasing latency/cost, which degrades reasoning quality over time.

## Solution
Insert a **Context Gateway** between the agent client and tools that **reduces tool outputs before they reach the LLM**, while keeping the full-fidelity data available out-of-band.

### Architecture (as described)
- Deploy an MCP server (“Context Mode”) that proxies tool results.
- For each tool response:
  1. Store raw payload in a blob/object store (or local disk) with a content-addressed ID.
  2. Produce a compact representation for the LLM context, such as:
     - structured summary (key fields)
     - extracted snippets
     - derived stats
     - references/handles to retrieve more later
  3. Return only the compact representation + handles to the agent client.

### Concrete implementation details
- Response format should include:
  - `summary`: short natural language + structured JSON
  - `handles`: URIs/IDs for raw payload sections
  - `budget`: token estimate of what was included
- Add policies per tool type:
  - Playwright snapshot → DOM skeleton + top-N relevant nodes
  - GitHub issues list → titles + labels + ids, omit body
  - Logs → error lines + statistical aggregation
- Allow “drill-down” tool: `fetch_raw(handle, range/filter)` to retrieve a precise slice when needed.

## Trade-offs
- **Pros**
  - Large context savings (source claims 315 KB → 5.4 KB; 98% reduction).
  - Lower latency and token cost; more stable long-running sessions.
  - Better tool UX: avoids drowning the model in irrelevant raw data.
- **Cons**
  - Summarization/compression can remove details needed later (requires drill-down path).
  - Adds infra/state (storage, handles, lifecycle/retention).
  - Requires tool-specific reduction strategies to avoid brittle summaries.

## When to use / When NOT to use
### Use when
- Your agent operates for many turns and uses tools that return large payloads.
- You pay per-token and see costs balloon due to tool output echoing.
- You need deterministic control of what enters context (“context budget”).

### Avoid when
- Short interactions with small tool payloads.
- Highly sensitive workflows where storing raw payloads introduces unacceptable risk (unless you can encrypt and manage retention).

## Implementation notes
- **Languages**: any (Node/Python/Go).
- **Libraries**:
  - MCP server SDK
  - storage: S3/Azure Blob/local FS
  - summarization: lightweight extractors first; LLM summarization only when needed
- **Effort**: ~3–10 engineer-days depending on number of tools and drill-down UX.
