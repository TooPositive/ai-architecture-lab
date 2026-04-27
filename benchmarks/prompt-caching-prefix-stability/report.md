# Benchmark: Prompt caching via prefix-stable prompts (OpenAI Prompt Caching)

## Source
- **Docs**: "Prompt caching" (OpenAI API docs)
- **URL**: https://developers.openai.com/api/docs/guides/prompt-caching

## What metric improves?
- **Latency**: Source claims prompt caching can **reduce latency up to 80%**.
- **Cost (input tokens)**: Source claims it can **reduce input token costs up to 90%**.

This does not primarily target retrieval accuracy (MRR), but it impacts **end-to-end p50 latency** and cost for repeated calls in agentic systems where prompts share large common prefixes.

## Technique summary
Prompt Caching automatically applies when:
- prompt length is **≥ 1024 tokens**, and
- subsequent requests have an **exact prefix match** (cache hits only possible for exact prefix matches).

Optimization guidance from the source:
- Put **static instructions/examples first** and move **dynamic content to the end**.
- Tools/images must be identical between requests to benefit from caching.
- Optional `prompt_cache_key` can be used to influence routing and improve hit rates.

## Theoretical comparison vs our baseline
Baseline (given):
- Hybrid search on 246 docs (Cosmos DB vector + BM25)
- **MRR@10 = 0.78**
- **Latency p50 = 45ms**

Comparison reasoning:
- If your pipeline includes LLM calls whose prompts reuse a long stable prefix (system prompt, tool schema, guardrails, examples), Prompt Caching reduces **compute time** on the model side because requests are routed to servers that recently processed the same prompt prefix.
- The baseline 45ms appears retrieval-side; Prompt Caching affects the generation-side. In realistic systems, generation latency is typically much larger than retrieval latency, so an **up to 80% reduction in generation-side latency** can dominate end-to-end improvements.

### Expected measurable outcomes (experiment design)
Measure before/after with identical traffic distribution:
- LLM **end-to-end latency** p50/p95
- LLM **effective $/request** (input token billed cost)
- **Cache hit rate** (% requests with cached prefix)

To be comparable across systems, normalize by:
- fixed model
- fixed prompt length
- fixed request rate (docs warn about overflow above ~15 req/min per prefix+key combination)

## Claimed improvements (from source)
- Latency reduction: **up to 80%**
- Input token cost reduction: **up to 90%**

## Confidence
High. Directly cited from OpenAI API documentation; exact improvements are stated as "up to" and will depend on prompt reuse and routing dynamics.
