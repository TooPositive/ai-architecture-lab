# Benchmark report — OpenAI API Prompt Caching (reused prefix discounts)

## Source
- **OpenAI API Prompt Caching (discounts for recently seen input tokens)** (summarized in internal knowledge base)
  - (No public URL captured in the knowledge node; internal reference id: `search-knowledge-13272d5b-841e-435a-84b8-570169d228b8`)

## What metric improves?
- **Cost** (input token cost) for workloads with repeated prompt prefixes.
- Potential secondary benefit: **latency** if the provider’s caching also avoids recomputation (source emphasizes billing discount; compute savings may or may not be exposed as latency).

## Claimed/expected improvement (from source)
- The API applies **automatic discounts** to input tokens that were **recently seen** by the model when the request contains the same prefix again.

## Theoretical comparison vs our baseline
Baseline (given):
- Retrieval: MRR@10 = **0.78**
- Latency: p50 = **45 ms**
- Corpus: **246 docs**, hybrid search Cosmos DB vector + BM25

Prompt caching does **not** change retrieval quality directly (MRR@10 should remain ~0.78 given same retriever/reranker and prompts).

Where it can help:
1. **Agent/tooling overhead**: If your agent uses a large stable system prompt + tool schemas + policy text on every call, and many calls share that prefix, prompt caching can reduce **per-call input cost**.
2. **Context utilization**: Not directly; but cost reduction makes it more feasible to keep a stable long prefix.

Expected effect on latency vs baseline:
- If your bottleneck is Cosmos DB retrieval (45 ms p50), prompt caching may not move end-to-end p50 much.
- If your bottleneck is LLM input processing for long prefixes, caching could reduce end-to-end p50/p95 (provider-dependent).

## How to benchmark in our environment
To make this benchmarkable against the baseline, measure:
- **$/request** and **$/1k queries** for a workload with high shared-prefix reuse (e.g., 10–20 tool calls per user task, same system prompt).
- **Latency p50/p95** for the LLM portion (and end-to-end).
- **Cache hit rate** proxy: proportion of requests that reuse an identical prefix in the cache window.

Minimal test design:
- Fix retriever settings (same Cosmos DB hybrid query).
- Run N=500 identical-structure requests differing only in user question, with identical system/tool prefix.
- Compare cost/latency with and without prompt-prefix reuse (e.g., randomized prefix to force misses).

## Confidence
- **Medium**: The feature exists per source summary, but the knowledge node does not include the original OpenAI doc URL or numeric discount details, limiting rigor.
