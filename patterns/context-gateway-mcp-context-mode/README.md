# Context Gateway for MCP Tool Outputs ("Context Mode")

## Source
- **Article:** *Stop Burning Your Context Window — We Built Context Mode*
- **URL:** https://mksg.lu/blog/context-mode

## Problem
Tool-using agents (e.g., via MCP) often stream large raw tool outputs (HTML snapshots, logs, issue lists) directly into the LLM context. Over multi-step sessions this:
- exhausts the context window,
- increases latency/cost due to repeated large inputs, and
- causes the agent to lose earlier instructions/history when truncation occurs.

The source reports examples like a Playwright snapshot (~56 KB) and GitHub issue dumps (~59 KB), where after ~30 minutes a large fraction of a 200K window is consumed.

## Solution
Insert a **Context Gateway** (implemented as an MCP server) between the agent runtime and external tools. The gateway intercepts tool outputs and replaces them with compact, model-friendly representations before they are appended to the conversation context.

Concrete architecture (as described):
1. Agent calls a tool through MCP as usual.
2. The **Context Gateway** receives the tool output.
3. The gateway applies a *compression/abstraction transform* (e.g., summarize, extract structured fields, strip boilerplate).
4. The agent receives back a **compact artifact** (often a short summary + references/handles), which is what gets added to the LLM context.

The source claims a reduction from **315 KB → 5.4 KB (≈98% reduction)** for captured tool outputs.

## Trade-offs
- **Loss of raw detail:** Aggressive compression can remove information the agent may later need. This pushes you toward (a) structured extraction and (b) keeping raw payloads addressable via handles.
- **Additional component to operate:** You introduce an MCP server that must be deployed, secured, and monitored.
- **Risk of compression errors:** If the gateway uses an LLM summarizer/extractor, it can hallucinate or omit key facts; if rule-based, it can be brittle across formats.

## When to use / When NOT to use
### Use when
- Your agent makes many tool calls that return large payloads (browser automation, log/trace retrieval, ticket dumps).
- Sessions are long-running and context truncation is a recurring failure mode.
- Latency/cost spikes correlate with large tool outputs.

### Avoid when
- Tool outputs are already small/structured (simple JSON with a few fields).
- You require provable, lossless grounding directly in the prompt (e.g., compliance workflows where raw evidence must be shown verbatim to the model).

## Implementation notes
- **Form factor:** MCP server sitting between Claude Code (or other MCP-capable clients) and tools.
- **Techniques:**
  - deterministic scrubbing (strip boilerplate, remove repeated headers),
  - structured extraction (IDs, titles, timestamps, key-value fields),
  - optional LLM summarization step for narrative text.
- **Artifacts:** Consider returning (a) compact summary + (b) stable content hash/URI to retrieve the full raw output out-of-band.
- **Effort (rough):**
  - 1–2 days for a minimal rule-based gateway for 1–2 tools
  - 1–2 weeks for a robust multi-tool gateway with observability and raw-artifact storage
