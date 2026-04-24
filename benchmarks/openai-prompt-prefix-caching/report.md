# Benchmark report — OpenAI Prompt Caching (exact-prefix caching)

## Source
- **Prompt caching | OpenAI API docs** — https://developers.openai.com/api/docs/guides/prompt-caching

## Technique summary
OpenAI automatically caches the **prefix** of prompts (exact prefix match) to reduce **latency** and **input token cost** for repeated prompts. It is enabled automatically for prompts **≥ 1024 tokens** and relies on routing requests based on a hash of the initial prefix. Developers can improve hit rates by keeping the prefix stable and moving variable content to the end of the prompt. The API also supports a `prompt_cache_key` parameter to influence routing and improve cache hit rates.

## What metric improves?
- **Latency (p50/p95)**: claimed reduction **up to 80%** (source).
- **Cost (input token cost)**: claimed reduction **up to 90%** (source).
Secondary: may improve **throughput** by reducing compute per request.

## Theoretical comparison vs our baseline
**Our baseline:** MRR@10=0.78, latency p50=45ms, hybrid search on 246 docs using Cosmos DB vector + BM25.

Prompt caching does not change retrieval ranking, so **MRR@10 is unchanged**. It targets the LLM stage costs/latency.

### Expected latency impact
If your workload has a large, stable prefix (system prompt + tool schema + fixed instructions) and repeated calls (agent loops), you can expect a material reduction in end-to-end latency. If we assume the LLM stage dominates total latency in the path, a cache hit could reduce LLM compute time substantially. OpenAI claims **up to 80% latency reduction** for cached prompts; the realized gain depends on:
- share of time spent in the LLM vs retrieval
- prefix length and stability (exact-match requirement)
- request rate and cache overflow behavior (~15 req/min per prefix+key combination before overflow can reduce effectiveness)

### Expected cost impact
OpenAI claims **up to 90% input token cost reduction** for cached input tokens (prefix hits). If your agent sends the same long prefix every turn, input cost can drop dramatically. The realized savings depends on:
- fraction of input tokens in the stable prefix
- how often the prefix repeats within the cache retention window

## Benchmark design we can run
- Workload: 1) single-turn RAG QA 2) multi-turn tool-using agent (N=10 steps)
- Vary: prefix stability (stable vs includes timestamps / request IDs / dynamic tool lists)
- Measure: end-to-end latency p50/p95 and $/request
- Control: same retrieval pipeline to isolate LLM-side gains

## Source-claimed implementation details / constraints
From the source doc:
- Cache hits require **exact prefix matches** (“Cache hits are only possible for exact prefix matches within a prompt.”).
- Enabled automatically for prompts **1024 tokens or longer**.
- Routing uses a hash of the initial prefix (typically first ~256 tokens; varies by model).
- Optional `prompt_cache_key` can be combined with prefix hash to influence routing and improve hit rates.

## Confidence
High. This is directly documented by OpenAI API docs, including explicit claims (“up to 80%” latency and “up to 90%” input cost).
