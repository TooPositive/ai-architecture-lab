# Context Gateway: Background Compaction Proxy

## Source
- **compresr — Context Gateway** (documentation)
- https://compresr.ai/docs/gateway

## Problem
In tool-heavy agent workflows, conversation history and large tool outputs rapidly consume the LLM context window. Naive approaches either:
- truncate history (quality loss)
- block the user while generating a summary (latency spike)
- repeatedly send huge tool outputs (cost increase)

## Solution
Insert a **transparent local proxy** between the agent and the LLM API that:
- monitors token usage continuously
- **pre-computes summaries in the background** before the limit is reached (e.g., at 75% of the context limit)
- performs **instant compaction** when the limit is reached (no user-visible wait)
- compresses large tool outputs on the fly
- logs compaction events for audit/perf analysis

### Architecture (concrete)
1. Agent (Claude Code / Codex / OpenHands / custom) sends chat/tool messages to **LLM API base URL**.
2. Instead of calling the provider directly, calls go to **Context Gateway** (HTTP proxy).
3. Gateway tracks rolling token count and triggers background compression at `CONTEXT_THRESHOLD` (default 0.75).
4. Compression uses a configured model (e.g., `COMPRESSION_MODEL=espresso_v1`) to produce a condensed history.
5. Once at the hard limit, gateway swaps in the condensed representation immediately and forwards to provider.
6. Gateway writes structured logs: history compaction, tool output compression, telemetry.

## Trade-offs
- **Extra moving part**: must run/operate a proxy (locally or as a shared service).
- **Summarization risk**: compression can drop details, especially if prompts are not designed to preserve invariants (IDs, diffs, stack traces).
- **Security**: proxy sees all prompts/tool outputs; requires careful key handling and access controls in team deployments.
- **Model cost**: background compression calls add cost, though offset by reduced context usage.

## When to use / When NOT to use
**Use when**:
- agent sessions are long-running (coding agents, ticket triage, SOC analysis)
- tools produce large outputs (browser snapshots, logs, issue lists)
- you need predictable latency (avoid blocking summaries)

**Do NOT use when**:
- conversations are short and fit comfortably in context
- the workload is highly detail-sensitive and hard to summarize safely (unless you add structured retention rules)

## Implementation notes
- **Provided implementation**: binary + Docker, supports OpenAI/Anthropic/Ollama/Bedrock per docs.
- **Integration**: point agent base URL to `http://localhost:8080` (default).
- **Config**: `~/.config/context-gateway/.env` (threshold, compression model, target ratio, logging).
- **Effort**: hours to a day for local setup; days for shared/team deployment with auth, TLS, observability integration.
