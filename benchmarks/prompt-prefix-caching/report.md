# Benchmark: Prompt Prefix Caching (Exact-Prefix Cache Hits)

## Source
- OpenAI API Docs, "Prompt caching" — https://developers.openai.com/api/docs/guides/prompt-caching

## What metric improves?
- Latency: OpenAI claims prompt caching can reduce latency up to 80%.
- Cost: OpenAI claims it can reduce input token costs up to 90% for cached tokens.
- (Indirect) Context utilization efficiency: by encouraging stable, reused prefixes (system prompt, tool schema, examples).

## Technique summary
Cache hits are only possible for exact prefix matches. Structure prompts so that stable content (system instructions, policies, tool definitions, few-shot examples) is at the beginning, and per-request variable data is at the end.

OpenAI notes:
- Caching is enabled automatically for prompts >= 1024 tokens.
- Routing uses a hash of an initial prefix (typically the first 256 tokens) and optionally a developer-provided `prompt_cache_key` to improve hit rates.

## Theoretical comparison vs our baseline
Baseline (RAG system):
- MRR@10 = 0.78
- latency p50 = 45ms
- corpus: 246 docs, Cosmos DB vector + BM25

Impact reasoning:
- Retrieval accuracy (MRR@10): not directly affected.
- End-to-end latency: likely improves when generation dominates latency and prompts have a long shared prefix (common in agent/tool setups with large tool schemas). If our generation step is a significant portion of request time, prefix caching can reduce LLM processing time for the cached prefix.
- Cost: if our app repeats large system/tool prefixes across turns, caching can materially reduce input-token spend.

Where this matters in a RAG pipeline:
- Multi-turn chat with stable system prompt + stable tool schemas + stable formatting instructions.
- Agent tool loops where each step repeats a long tool schema and policies.

## Claimed improvements (from source)
- Latency reduction up to 80%.
- Input token cost reduction up to 90%.

## How to benchmark in our environment (suggested)
- Hold retrieval constant (same retrieved passages).
- Vary prompt structure:
  1) Stable prefix first (instructions/tools/examples), variables last.
  2) Mixed variables throughout (lower cache hit rate).
- Measure:
  - End-to-end p50/p95 latency
  - Effective $/request and cached-token ratio
  - Any changes in output quality (should be neutral; confirm with offline eval)
