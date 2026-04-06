# Intent-Conditioned Tool Output Compression Proxy (Context Gateway)

## Source
- **Show HN: Context Gateway – Compress agent context before it hits the LLM** (discussion + design details)
  https://news.ycombinator.com/item?id=47367526
- **Context Gateway (GitHub README)**
  https://github.com/Compresr-ai/Context-Gateway
- **Compresr docs: Context Gateway**
  https://compresr.ai/docs/gateway

## Problem
Tool calls in coding agents (e.g., `grep`, file reads, web/MCP fetches) can inject huge, low-signal outputs into the LLM context window. This causes:
- Context exhaustion (hitting hard context limits)
- Increased cost (more input tokens)
- Degraded quality due to “context rot” / long-context accuracy drop
- Latency spikes when the agent must stop and summarize/compact synchronously

## Solution
Insert a **transparent local proxy** between the agent and the LLM API that:
1. **Monitors token usage** over the running conversation.
2. **Compresses tool outputs on the fly**, *conditioned on the tool-call intent* (e.g., compress `grep` output differently depending on what the agent was searching for).
3. **Pre-computes background compaction** when context utilization crosses a threshold (e.g., 75–85%), so when the limit is reached the swap-in is instant.
4. Keeps a reversible path via an **`expand()`** mechanism to retrieve the original tool output if needed later (per HN description).
5. Optionally **lazy-loads tool descriptions**, so the LLM only sees tools relevant to the current step (per HN description).

Concrete implementation details (from docs/README):
- Runs as a local proxy (default **`http://localhost:8080`**) and agents route API calls through it.
- Configurable **`CONTEXT_THRESHOLD`** (default 0.75) to trigger background compression.
- Configurable compressor model (e.g., **`COMPRESSION_MODEL=espresso_v1`**) and **`TARGET_COMPRESSION_RATIO`** (e.g., 0.5).
- Produces observable logs such as:
  - `logs/history_compaction.jsonl`
  - `logs/tool_output_compression.jsonl`
  - `logs/telemetry.jsonl`

## Trade-offs
- **Information loss risk**: removing “irrelevant” parts can hide details the agent later needs; requires a reliable `expand()` fallback and good intent conditioning.
- **Security boundary mixing**: compressing untrusted external content (web/MCP) together with trusted system instructions can blur trust boundaries; requires scan-then-compress or separate compression passes for trusted vs untrusted inputs (raised in HN comments).
- **Added moving part**: proxy introduces a new failure mode (misconfiguration, proxy downtime, serialization issues).
- **Compression compute cost**: background compaction still spends tokens/compute; you trade fewer “main model” tokens for smaller-model compression calls.

## When to use / When NOT to use
**Use when:**
- Your agent regularly ingests large tool outputs (repo-wide grep, large file reads, logs, issue lists).
- You see latency spikes or cost spikes near context limits.
- You need a solution that is **agent-agnostic** (Claude Code/Cursor/OpenHands/etc.) and does not require changing agent source.

**Do NOT use when:**
- Your agent already enforces small tool outputs (strict truncation/filters) and stays well below context limits.
- You need strict determinism/auditability of *exact* tool outputs shown to the LLM (compression changes the evidence).
- You frequently process adversarial/untrusted text without a robust sanitization pipeline.

## Implementation notes
- **Languages**: Go (Context Gateway), integrates with any agent via HTTP proxy configuration.
- **Effort**:
  - 1–2 days to adopt for a single developer workflow (install + config).
  - 1–2 weeks for team deployment (Docker/K8s + monitoring + policy around trusted/untrusted inputs).
- **Libraries/infra**: runs as a local binary; can be containerized (docs show Docker).
- **Key config knobs** (from docs): `CONTEXT_THRESHOLD`, `COMPRESSION_MODEL`, `TARGET_COMPRESSION_RATIO`, `AGENT_TYPE`, `PROXY_PORT`.
