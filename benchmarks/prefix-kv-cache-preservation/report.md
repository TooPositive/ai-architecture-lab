# Benchmark report: Prefix-Stable System Prompt (KV-Cache Preservation)

## Technique
Keep the system-prompt prefix bit-identical across requests so the inference server can reuse KV/prefix cache instead of re-prefilling a large static prompt every turn.

## Source
- "PSA: Using Claude Code without Anthropic: How to fix the 60-second local KV cache invalidation issue." (Reddit r/LocalLLaMA)
  - https://www.reddit.com/r/LocalLLaMA/comments/1s7tn5s/psa_using_claude_code_without_anthropic_how_to/

## What metric improves?
- **Latency** (especially per-turn/p50 in tool-heavy agent loops)
- **Cost/throughput** on local inference (less repeated prefill compute)
- **Context utilization** indirectly (less need to shorten system prompt just to keep latency tolerable)

## Theoretical comparison vs our baseline
Baseline (given): RAG MRR@10=0.78, latency p50=45ms, hybrid search on 246 docs using Cosmos DB vector + BM25.

This technique does not change retrieval quality (MRR@10), but can reduce end-to-end agent latency when:
- your stack includes a long static system prompt (10K–20K+ tokens), and
- your inference backend supports prefix/KV caching, and
- the client previously injected dynamic text into the system prompt each call.

Reasoning:
- With prefix caching working, the server can reuse the prefill for the stable prefix and only compute attention for the incremental new tokens (tool outputs + user input).
- With prefix caching broken, each call redoes prefill for the full system prompt, dominating latency even when the new turn is small.

## Claimed improvements from the source
The source claims that Claude Code’s dynamic prompt injection "instantly breaks prefix matching, flushes your entire KV cache, and forces your hardware to re-process a 20K+ token system prompt from scratch for every minor tool call" and that this can be fixed via configuration (`~/.claude/settings.json`).

No numeric latency improvement is provided in the post, so this is **benchmarkable** but requires local measurement.

## How we would benchmark it
- Instrument requests to log:
  - hash(system_prefix)
  - prompt token counts (prefill vs decode)
  - server-side cache hit/miss (if available)
  - latency breakdown (prefill time vs decode time)
- Run a tool-heavy workflow (e.g., 30 tool calls) under two conditions:
  1) dynamic system prefix (cache misses)
  2) frozen system prefix (cache hits)
- Compare p50/p95 latency and tokens/sec.
