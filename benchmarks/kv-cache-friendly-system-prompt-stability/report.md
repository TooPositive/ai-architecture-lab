# Benchmark: KV-cache-friendly system prompt stability (avoid prefix invalidation)

## Source
- **"Claude Code with local backends: KV cache invalidation caused by dynamic system-prompt injection (60-second issue) and settings.json fix"** (knowledge base node: `search-knowledge-7ab40724-38b2-49ad-ba9b-70f52013dd14`)

## What metric improves?
- **Latency** (especially p50/p95) for multi-turn tool-using sessions on backends that support **prefix matching / KV cache reuse** (e.g., llama.cpp server).
- **Cost / compute** (reprocessing fewer input tokens per turn).

## Technique (what to do)
Keep the **system prompt prefix stable across turns** so the inference backend can reuse the KV cache. Specifically, avoid injecting frequently-changing content into the earliest part of the prompt, such as:
- telemetry headers
- dynamic git status
- ever-changing tool definition blocks

If the client must include dynamic metadata, place it later in the prompt or in a separate, smaller message segment that does not break prefix matching. The source notes that configuring Claude Code via `~/.claude/settings.json` can reduce/disable these dynamic injections.

## Theoretical comparison vs our baseline
Baseline (given):
- MRR@10 = 0.78
- Latency p50 = 45ms
- Hybrid search over 246 docs (Cosmos DB vector + BM25)

This technique does **not** change retrieval accuracy (MRR). It targets end-to-end agent turn latency when the model backend supports KV reuse.

Reasoning:
- In KV-cached inference, each next token generation depends on cached attention keys/values from the prompt prefix.
- If the prefix changes each turn, the backend cannot reuse cached states and must recompute the full prefix, turning an incremental step into a near full forward pass over the entire prompt (potentially 20k+ tokens).
- Therefore, stabilizing the prefix should reduce per-turn compute roughly proportional to the number of repeated prefix tokens, improving latency and throughput.

## Claimed improvements (from the source)
- The source reports Claude Code injecting dynamic telemetry and git-status updates into the system prompt on every request, which **breaks prefix matching and flushes the KV cache**, leading to extremely slow behavior on local backends (described as a “60-second issue”).
- It also notes high per-turn overhead due to large tool-definition payloads (~62,600 characters of tool definitions per turn) contributing to repeated prompt processing.

## How to benchmark against our baseline
Suggested A/B test (same model + hardware):
- A: dynamic system prompt (changes each turn)
- B: stable system prompt prefix (dynamic content moved out or disabled)

Measure:
- end-to-end per-turn latency p50/p95 across N=50 turns
- tokens/sec and total prompt tokens processed per turn
- cache hit rate (if backend exposes it)

Expected outcome:
- Large reduction in per-turn latency for tool-heavy sessions with stable prefixes; negligible effect on MRR@10.
