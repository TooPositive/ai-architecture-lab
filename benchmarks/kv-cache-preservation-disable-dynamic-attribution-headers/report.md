# Benchmark report: Preserving KV/prefix cache by disabling dynamic attribution headers in system prompt

## Source
- AgentPatterns.ai. **"Disable Attribution Headers to Preserve KV Cache in Local Inference"**
  https://www.agentpatterns.ai/context-engineering/kv-cache-invalidation-local-inference/

## Technique
Some agent runtimes inject **dynamic attribution/telemetry** into the *beginning* of the prompt (e.g., system prompt). For local inference servers that rely on **prefix matching / KV cache reuse**, tiny changes in the prefix cause cache misses and force full prompt reprocessing.

The technique is to **disable or stabilize** these dynamic injections so the prompt prefix remains identical between turns, enabling KV cache reuse.

## What metric improves?
- **Latency** (especially multi-turn): avoids reprocessing large, repeated prefixes.
- **Compute cost** (local GPU/CPU): fewer prompt tokens re-evaluated per turn.

## Theoretical comparison vs our baseline
Baseline (given):
- Retrieval: MRR@10 = 0.78
- Latency: p50 = 45ms (retrieval)
- Hybrid search: Cosmos DB vector + BM25 on 246 docs

This technique primarily targets **generation-side latency**, not retrieval latency.
If your system prompt + tool definitions are large (common for agents), KV cache preservation can reduce per-turn latency by eliminating repeated prefill computation.

A rough upper bound: if prefill dominates, improvements can be much larger than 45ms; if retrieval dominates, improvements are smaller.

## Claimed/mechanistic improvements (from source)
The source describes the failure mode (dynamic prompt prefix breaks KV caching) and prescribes removing/stabilizing attribution headers. It does not provide a single universal numeric speedup, but the mechanism is directly measurable.

## How to benchmark in our environment
1. Use a local inference backend that supports prefix/KV caching (e.g., llama.cpp server or similar).
2. Fix a large static system prompt (e.g., 10k tokens tool schema).
3. Run 100 multi-turn interactions where only the user message changes.
4. Compare p50/p95 latency with:
   - (A) dynamic prefix injection enabled (simulate by adding a random header line in the system prompt each turn)
   - (B) dynamic prefix injection disabled (exact identical prefix)
5. Track backend metrics: prefill tokens/sec, cache hit ratio, total tokens processed.

## Classification
**BENCHMARKABLE** (latency and compute can be measured; technique targets a specific caching failure mode described in the source).
