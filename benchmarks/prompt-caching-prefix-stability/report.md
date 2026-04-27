# Benchmark report — Prompt-caching prefix stability

## Classification
**BENCHMARKABLE** — measurable improvements in *latency* and *input token cost* when repeated requests share an exact prompt prefix.

## Source
- OpenAI API Docs — **"Prompt caching"**
- https://developers.openai.com/api/docs/guides/prompt-caching

## What metric improves?
- **Latency**: source claims "Prompt Caching can reduce latency by up to **80%**".
- **Cost**: source claims "input token costs by up to **90%**" (for cached prefix tokens).

## Technique (what to implement)
1. **Make the prompt prefix stable**: put static instructions/examples first; push variable content (timestamps, request IDs, user-specific context, dynamic tool lists, tool outputs) to the end.
2. Ensure **tools/images** included in the prompt are identical between requests (exact-prefix requirement).
3. Only prompts **>=1024 tokens** are eligible for caching (per source).
4. Optionally set **`prompt_cache_key`** to influence routing and improve cache hit rates when many requests share long prefixes.

## Theoretical comparison vs our baseline (reasoned)
Baseline: **MRR@10=0.78**, **latency p50=45ms**, hybrid search on 246 docs (Cosmos DB vector + BM25).

- **Retrieval accuracy (MRR@10)**: *unchanged* — prompt caching does not alter retrieval ranking.
- **End-to-end latency**: if your current p50 includes a meaningful model-inference component with repeated system/tool definitions, caching can reduce that portion substantially. If we assume:
  - p50 total = 45ms
  - model-inference share = 30ms (example)
  - and caching yields 50% reduction on that share (conservative vs "up to 80%")
  - then new p50 ≈ 45ms − 15ms = **30ms** (≈33% faster).
- **Token cost**: with stable prefixes, repeated calls in an agent loop (same system prompt + tool schemas) can see large input-token discounts on those cached tokens (claimed up to 90%).

## How to benchmark (recommended)
- Workload: run a fixed agent flow with identical system prompt + tool schemas for N requests; vary only user input.
- Metrics: p50/p95 latency, input tokens billed, cache hit rate (if exposed), cost/request.
- A/B: "stable prefix" vs "unstable prefix" (inject timestamp into first 256 tokens).

## Notes / limits from the source
- Cache hits require **exact prefix matches**.
- Caching applies after a prefix hash (often first ~256 tokens) and is automatic for eligible prompts.
- High request rates can "overflow" routing and reduce cache effectiveness (source mentions ~15 req/min as an approximate threshold where overflow may begin).
