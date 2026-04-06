# Benchmark report: Prompt prefix caching by prompt structuring (exact-prefix cache hits)

## Source
- OpenAI Docs. **"Prompt caching"**
  https://developers.openai.com/api/docs/guides/prompt-caching

## Technique
Structure prompts to maximize **exact prefix matches**, so the provider’s prompt cache can be used:
- Put **static** content first: system prompt, instructions, examples, tool definitions (must be identical).
- Put **variable** content last: user input, retrieved passages, per-request metadata.

OpenAI notes:
- Cache hits require **exact prefix matches**.
- Prompt caching can reduce **latency up to 80%** and **input token costs up to 90%** (for cached tokens), and is automatically applied for prompts ≥ 1024 tokens.

## What metric improves?
- **Latency**: lower p50/p95 for repeated calls sharing a long prefix.
- **Cost**: reduced billed input tokens (cached portion discounted).
- **Context utilization**: indirectly improves effective throughput because repeated static prefixes become cheaper/faster to process.

## Theoretical comparison vs our baseline
Baseline (given):
- Retrieval: MRR@10 = 0.78
- Latency: p50 = 45ms
- Hybrid search: Cosmos DB vector + BM25 on 246 docs

Prompt prefix caching does **not** change retrieval MRR directly, but it can change end-to-end response time and cost:
- If your agent repeatedly sends a large system prompt + stable tool schema, the cache can apply to the **first N tokens** (provider-dependent; OpenAI describes hashing and routing based on an initial prompt prefix).
- For RAG apps, keep the RAG results in the **tail** so the prefix remains cacheable.

Expected impact (qualitative):
- If your system+tools prefix is large (e.g., 2k–20k tokens) and stable across turns, end-to-end latency can drop significantly on multi-turn sessions, while retrieval latency (45ms) remains constant.
- The effect is strongest when the generation step dominates latency or when system prompts are huge (coding agents, tool-heavy agents).

## Claimed improvements (from source)
- Up to **80% lower latency** (cached prefix)
- Up to **90% lower input token cost** (cached prefix)

## How to benchmark in our environment
1. Define a stable system prompt + tool schema (constant) sized ≥ 1024 tokens.
2. Run 200 requests with unique user queries + unique retrieved contexts appended at the end.
3. Measure end-to-end p50/p95 latency and cost per request with and without prompt structuring (i.e., deliberately randomize or mutate the prefix to eliminate cache hits).
4. Keep retrieval path constant to isolate generation-side improvements.

## Classification
**BENCHMARKABLE** (clear measurable changes in latency and token cost; source provides concrete improvement claims).
