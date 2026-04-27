# Benchmark: Prompt Prefix Caching (Provider KV/Prefix Cache)

## Source
- **OpenAI API Prompt Caching (discounts for recently seen input tokens)** (KB node `search-knowledge-13272d5b-841e-435a-84b8-570169d228b8`; summarized from OpenAI documentation/announcement).
- URL: *not provided in KB record*.

## What metric improves?
- **Cost** (discounted input tokens when the request shares a recently-seen prefix).
- Potential **latency** improvement depending on provider implementation (not explicitly claimed in the KB summary; treat as uncertain).
- **Context utilization stability** indirectly improves if you keep a stable prefix and move dynamic content later, but primary lever is cost.

## Theoretical comparison vs our baseline
Baseline (given):
- MRR@10 = 0.78
- latency p50 = 45ms
- hybrid search on 246 docs (Cosmos DB vector + BM25)

Prompt prefix caching does **not** change retrieval quality (MRR@10) because it’s upstream of retrieval; it reduces repeated prompt compute/billing for shared prefixes (system prompt, tool instructions).

Expected impact:
- **Cost reduction** proportional to prefix reuse rate across requests in a short time window.
  - If your agent makes N tool calls per user turn and repeats the same long system/tool instruction prefix each time, caching can discount a large portion of input tokens.
- **Latency**: if provider uses cached KV states, there may be lower compute per request; however this is not confirmed by the provided source summary, so treat as **unknown** until measured.

## Claimed improvements (from source)
- The source describes **automatic discounts** for input tokens that were recently seen by the model, optimizing for repeated/shared prompt prefixes.
- Optimization guidance from the source summary: keep the **beginning** of prompts stable; move dynamic fields (timestamps, per-request tool outputs) later to avoid cache misses.

## How we would benchmark vs baseline
- Add metrics:
  - input tokens billed (and discounted) per request
  - effective $/request
  - cache hit rate (% tokens cached)
- Keep retrieval pipeline constant (Cosmos DB vector+BM25).
- Run a workload with repeated tool calls and long, stable system prompts to measure cost deltas.

## Confidence
- **Medium** on cost benefit (source explicitly frames it as cost discounts).
- **Low** on latency benefit (not explicitly claimed in the source summary).
