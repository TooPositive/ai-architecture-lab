# MCP Context Compression Gateway (Tool Output Interposition)

## Source
- **Stop Burning Your Context Window — We Built Context Mode** — Mert Köseoğlu
- https://mksg.lu/blog/context-mode

## Problem
In MCP-based agent systems, tool outputs are often injected verbatim into the LLM context. Large outputs (logs, Playwright snapshots, issue lists) quickly consume the context window, increasing cost and reducing model performance due to loss of relevant instructions/history.

## Solution
Insert an **MCP server / gateway** between the agent runtime and tools that **compresses/summarizes/tool-output-diffs** before they reach the LLM.

Architecture (as described):
- “Context Mode is an MCP server that sits between Claude Code and these outputs.”
- Claim: **315 KB → 5.4 KB (98% reduction)** for tool outputs in a session.

Concrete implementation details:
1. Agent calls tools through MCP as usual.
2. Gateway intercepts tool responses and transforms them into compact artifacts:
   - extract key fields
   - summarize sections
   - keep pointers/handles to the full raw payload (stored out-of-band)
   - optionally return diffs vs previous snapshots
3. Only the compact representation is injected into the LLM prompt.
4. On demand, agent can request the raw payload (or a focused slice) via a follow-up tool call.

## Trade-offs
- **Pros**
  - Dramatically improves context utilization; prevents tool noise from crowding instructions.
  - Lowers token cost and can reduce latency by shrinking prompt size.
  - Creates a natural place for redaction (PII) and output normalization.
- **Cons**
  - Risk of losing detail needed for correct reasoning if compression is too aggressive.
  - Requires storage for raw tool outputs (blob store/local cache) and secure access control.
  - Additional component to run/monitor; failure mode can block tool use.

## When to use / When NOT to use
**Use when:**
- You use MCP tools that return large payloads (browser automation, logs, codebase scans).
- Sessions are long-running and you observe context blow-ups.
- You want standardized, safe tool output formatting/redaction.

**Do NOT use when:**
- Tool outputs are small and structured already.
- You cannot tolerate any information loss and cannot implement raw-on-demand fallback.

## Implementation notes
- Best implemented as an **MCP proxy server** with:
  - a compression policy per tool
  - a raw-payload store (S3/Azure Blob/local disk)
  - identifiers in the returned summary that reference stored raw payloads
- Languages: Node.js/TypeScript, Python, Go all work for MCP servers.
- Effort: ~2–7 days depending on number of tools and whether you add diffing + storage + auth.
