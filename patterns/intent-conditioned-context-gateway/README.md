# Intent-conditioned Context Gateway (proxy-level context compaction)

## Source
- **Context Gateway Documentation (Compresr)** 
  https://compresr.ai/docs/gateway
- **Show HN: Context Gateway – Compress agent context before it hits the LLM** 
  https://news.ycombinator.com/item?id=47367526
- **GitHub repository: Compresr-ai/Context-Gateway** 
  https://github.com/Compresr-ai/Context-Gateway

## Problem
Agentic systems (coding agents, tool-using assistants) frequently blow up their context window with:
- Large tool outputs (grep, search results, logs, file reads)
- Growing conversation histories

This creates three concrete failures:
1. **Latency spikes** when the system needs to “compact” history synchronously at the limit.
2. **Higher token cost** from repeatedly sending bulky tool outputs and long histories.
3. **Quality degradation (“context rot”)** as the model attends over more irrelevant tokens.

## Solution
Insert a **transparent local proxy** between the agent and the LLM API that performs **background, intent-conditioned compression** of context before it enters the prompt.

### Architecture (concrete)
1. **Proxy intercepts** all chat/completions requests from the agent (OpenAI/Anthropic/Ollama/Bedrock, etc.).
2. The proxy **tracks token usage** across the conversation.
3. At a configurable threshold (e.g., **75%** of the model context limit), the proxy **pre-computes a compaction** of the conversation history *in the background* (“pre-summarize before you must”).
4. When the context limit is reached, the proxy can **swap in the pre-computed compacted history instantly** (no blocking compaction call at the critical moment).
5. For tool calls, the proxy performs **tool-output compression on the fly**, conditioned on the **intent of the tool call** (as described in the Show HN post):
   - If the agent ran `grep` looking for *error handling patterns*, the compressor preserves the matching lines/snippets and strips noise.
6. The proxy supports an **expand()** mechanism (described in Show HN) so the agent can retrieve the original tool output if needed after aggressive compression.
7. The proxy emits **structured logs** for each compaction/compression event (e.g., `logs/history_compaction.jsonl`, `logs/tool_output_compression.jsonl`).

### Implementation sketch
- Run the proxy locally (default: `http://localhost:8080`) and configure the agent to use it as the LLM “base URL”.
- Set environment variables / config:
  - `CONTEXT_THRESHOLD=0.75`
  - `COMPRESSION_MODEL=espresso_v1` (as in docs)
  - `TARGET_COMPRESSION_RATIO=0.5`
- Deploy as a shared service (Docker/Kubernetes) for team-wide usage (docs show Docker example).

## Trade-offs
- **Compression can remove information** needed later; you need an escape hatch (the pattern’s `expand()` / original-output retrieval).
- **More moving parts**: a proxy becomes a critical path dependency (availability, logging hygiene, security).
- **Security boundary risk** (raised in HN comments): compressing untrusted external content can interleave prompt-injection text with trusted instructions; consider scan/segregate before compress.
- **Additional compute**: background compaction and tool-output compression consume extra model calls/compute (but intended to reduce net cost/latency by reducing prompt tokens).

## When to use / When NOT to use

### Use when
- Tool outputs are routinely huge (codebase greps, logs, traces, multi-file reads).
- You see user-visible stalls when hitting context limits (synchronous compaction).
- You need **token cost control** and better context hygiene without rewriting your agent.

### Avoid when
- Your agent already uses strict retrieval (RAG) and avoids injecting raw tool outputs.
- You must preserve **exact tool outputs** (e.g., legal/medical audit trails) and can’t tolerate lossy compression.
- You cannot safely introduce a proxy into a regulated environment without a full threat model (prompt injection / data leakage risk).

## Implementation notes
- **Languages**: Context Gateway is implemented in **Go** (repo).
- **Integration effort**: low-to-medium (hours to a day) if your agent supports configurable base URL; higher if your agent hardcodes provider endpoints.
- **Ops**: add monitoring around `telemetry.jsonl` and compaction logs; treat gateway as critical infra if deployed centrally.

