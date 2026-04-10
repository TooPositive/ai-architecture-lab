# Benchmark: OpenAI Prompt Caching (repeated-prefix input discounts)

## Source
- OpenAI API docs: **Prompt caching** (documentation page)
  - https://developers.openai.com/api/docs/guides/prompt-caching

## What metric improves?
- **Cost (input token $)** for repeated requests that share a long prefix (system prompt, repeated instructions, shared context blocks).
- Potentially **latency** (provider-dependent), but the source emphasizes billing discounts.

## Claimed / described improvement (from source)
The documentation describes that prompt caching "reduce[s] latency and cost" by applying caching to repetitive prompt prefixes. (Exact discount rates and cache window are provider-controlled; see docs for current terms.)

## Theoretical comparison vs our baseline
Baseline (given):
- RAG: MRR@10=0.78
- Latency p50=45ms
- Hybrid search on 246 docs using Cosmos DB vector + BM25

Prompt caching does **not** directly change retrieval quality (MRR@10). It targets LLM-side input reuse.

### Expected effect in our system
- If our agent has a large stable system prompt (tool schemas, policies, routing rules) reused across turns, then:
  - **Input cost per turn decreases** proportionally to the fraction of tokens that qualify as cached.
  - **End-to-end latency may decrease** if the provider skips some compute for cached prefix (implementation-specific; not guaranteed).

### How to benchmark
1. Fix a representative multi-turn workload (e.g., 20-turn agent session).
2. Ensure the first N tokens (system + tool definitions) are identical across turns.
3. Compare billed input tokens / $ across:
   - caching-eligible runs (stable prefix)
   - non-eligible runs (mutated prefix each turn)
4. Report:
   - $/session and $/turn
   - p50/p95 latency
   - cache hit rate (if exposed)

## Notes / risks
- Caching eligibility depends on provider and request structure.
- If tool definitions or system prompts change each turn, caching benefits drop sharply.
