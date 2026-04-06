# Pattern: Background pre-compaction proxy ("Context Gateway") for instant context management

## Source
- Compresr. **"Context Gateway" documentation**
  https://compresr.ai/docs/gateway
- Compresr-ai. **"Context-Gateway" (GitHub repository README)**
  https://github.com/Compresr-ai/Context-Gateway

## Classification
**NEW_PATTERN** (the specific proxy behavior of *pre-computing summaries at a threshold* so compaction is instant at the limit, plus on-the-fly tool-output compression and compression event logging, is a concrete architecture technique beyond generic "summarize history" guidance).

## Problem
Long-running agent sessions (coding agents, tool-using assistants) hit context-window limits. Typical systems:
- Wait until the limit is reached, then perform a blocking summarization/compaction step.
- Lose time/UX ("pause") right when the user needs responsiveness.
- Waste context on large tool outputs (logs, HTML snapshots, issue lists).

## Solution
Insert a **transparent local proxy** between the agent and the LLM API that continuously monitors token usage and performs **background compaction** before it’s needed.

### Architecture (from the sources)
- Run a local proxy (default `http://localhost:8080`) and point the agent’s API base URL at it.
- The proxy:
  1. **Monitors token usage** across the conversation.
  2. At a configurable threshold (docs: **75%** of context limit), it **pre-computes a summary** in the background.
  3. When the conversation reaches the limit, it performs **instant compaction** (swap in the precomputed summary) so the user does not wait.
  4. **Compresses large tool outputs on the fly** (separate from general history compaction).
  5. Logs compression events + metrics to JSONL for observability (examples in docs):
     - `logs/history_compaction.jsonl`
     - `logs/tool_output_compression.jsonl`
     - `logs/telemetry.jsonl`

### Concrete implementation details
- Install via `curl -fsSL https://compresr.ai/api/install | sh` and run `context-gateway` to start a TUI wizard (source docs).
- Configure via `~/.config/context-gateway/.env` (source docs), including:
  - `CONTEXT_THRESHOLD=0.75`
  - `COMPRESSION_MODEL=espresso_v1`
  - `TARGET_COMPRESSION_RATIO`
  - `AGENT_TYPE` (claude_code/openclaw/opencode/custom)

## Trade-offs
- **Extra moving part**: introduces a proxy service to operate and secure (local or remote deployment).
- **Summarization quality risk**: aggressive compression can lose details; you need good summaries and potentially "pin" important facts outside compaction.
- **Cost/latency for background summarization**: you pay for extra summarization calls (but it shifts cost/latency off the critical path).
- **Potential privacy/security concerns**: proxy sees full prompts/tool outputs; requires careful handling in enterprise environments.

## When to use / When NOT to use
### When to use
- Long-running interactive sessions (coding agents, incident response agents) where pauses are unacceptable.
- Tool-heavy agents producing large outputs (logs, diffs, DOM snapshots).
- Teams that want **observability** over compression decisions (JSONL event logs).

### When NOT to use
- Short conversations where compaction almost never triggers.
- Environments where adding a proxy is not acceptable due to compliance or threat model (unless you self-host and harden it).

## Implementation notes
- Languages/libraries:
  - Context Gateway is implemented in **Go** (GitHub repo).
  - Works as a provider-agnostic HTTP proxy to OpenAI/Anthropic/etc. (docs).
- Effort:
  - 1–2 hours to trial locally with an agent that supports custom base URLs.
  - 1–2 days to productionize (deployment, TLS, auth, log shipping, policies).
- Operational best practices:
  - Treat compaction artifacts as first-class logs; ship JSONL events to your observability stack.
  - Separate "tool output compression" from "history compaction" so you can tune them independently.
