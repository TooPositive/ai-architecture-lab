# Benchmark report: Prompt caching via prefix-hash routing + `prompt_cache_key`

## Source
- OpenAI API docs: **"Prompt caching"**
  https://developers.openai.com/api/docs/guides/prompt-caching

## Technique summary
OpenAI prompt caching enables automatic reuse of computation for **exact prompt prefix matches**, reducing both **latency** and **input token cost** for repeated (or mostly repeated) prompts. The docs describe a concrete mechanism: requests are routed based on a **hash of the initial prefix** (typically first ~256 tokens; exact length varies), and developers can optionally provide `prompt_cache_key` to improve cache hit rates when many requests share long common prefixes.

## What metric improves?
- **Latency**: docs claim up to **80% reduction** in latency on cache hits.
- **Cost**: docs claim up to **90% reduction** in input token costs on cached tokens.
- **Throughput / context utilization**: indirect improvement because large static prefixes become cheaper/faster to send repeatedly (system prompt, tool schemas, examples).

## Claimed improvements (per source)
From the OpenAI docs:
- "Prompt Caching can reduce latency by up to 80% and input token costs by up to 90%."
- Cache hits require **exact prefix matches**; static content should be at the start, variable content at the end.
- Caching is enabled automatically for prompts **>= 1024 tokens**.
- Routing uses a **prefix hash** (typically first ~256 tokens) and optionally `prompt_cache_key`.

## Theoretical comparison vs our baseline
**Our baseline:** MRR@10=0.78, latency p50=45ms, hybrid search on 246 docs using Cosmos DB vector + BM25.

Prompt caching does not change retrieval quality (MRR@10), but it can materially reduce **end-to-end request latency and cost** when your application repeatedly sends:
- long, stable system prompts
- repeated tool schemas / JSON schema
- repeated few-shot examples
- repeated retrieved context blocks (e.g., popular FAQs)

If your current p50=45ms is dominated by retrieval+network and *not* model latency, the relative win may be smaller; but for most RAG/chat apps, model time dominates. A cache-hit could plausibly cut *model-side* latency significantly, lowering p50 and p95 even if retrieval remains constant.

### Expected effect on latency (reasoned)
Let:
- `T_total = T_retrieval + T_llm`
- If `T_llm` drops by up to 80% on cache hits, then
  `T_total_hit ≈ T_retrieval + 0.2*T_llm`
Even modest cache-hit rates (e.g., repeated system prompt across many users) can reduce fleet-wide p50/p95.

### Expected effect on cost (reasoned)
If a large portion of your input tokens are stable prefix tokens, and those tokens are eligible for caching, then billed input cost decreases for those tokens. This is especially relevant for agentic systems where the system prompt + tool definitions can be thousands of tokens.

## How to benchmark in our lab
1. Fix a representative workload (e.g., 1000 agent requests) with a long stable prefix and variable suffix.
2. Run with and without `prompt_cache_key` (when available) and measure:
   - end-to-end p50/p95 latency
   - billed input tokens / $ cost
   - cache-hit rate (if the provider returns any indicator; otherwise infer via cost deltas)
3. Try adversarial cases that break prefix matching:
   - dynamic system-prompt injection
   - non-deterministic tool schema ordering
   - timestamp/user-id in prefix

## Notes / caveats (source-grounded)
- Cache hits are only possible for **exact prefix matches**.
- "Overflow" behavior: if requests for same prefix+key exceed ~15 req/min, some may route to other machines and reduce effectiveness.
- This is provider-managed and model/version dependent; it is not portable to non-OpenAI backends unless you implement KV/prefix caching yourself.
