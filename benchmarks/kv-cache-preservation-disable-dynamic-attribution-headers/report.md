# Benchmark report: Preserve KV cache by eliminating dynamic system-prompt headers (prefix stability)

## Source
- AgentPatterns.ai. **"Disable Attribution Headers to Preserve KV Cache in Local Inference"**
  https://www.agentpatterns.ai/context-engineering/kv-cache-invalidation-local-inference/

## Technique summary
When using local inference servers (e.g., llama.cpp/LM Studio style servers), performance often depends on **prefix matching** to reuse the model **KV cache** across turns. If the client mutates the beginning of the prompt each request (e.g., changing attribution/telemetry headers), the server cannot reuse the prefix → KV cache is invalidated → the server must re-process the full prompt every turn.

The source describes disabling or removing these dynamic headers so the prompt prefix remains stable and KV caching works.

## What metric improves?
- **Latency (p50/p95)**: fewer tokens re-processed per turn due to KV cache reuse.
- **Cost (compute)** for self-hosted/local: reduced CPU/GPU work per request.
- **Context utilization** indirectly: stable prompt structure discourages repeated re-sending of huge tool definitions that can’t be cached effectively when prefixes change.

## Theoretical comparison vs our baseline
Baseline (given):
- Retrieval: MRR@10 = 0.78
- Latency: p50 = 45ms
- Corpus: 246 docs, Cosmos DB vector + BM25 hybrid

This technique does **not** improve retrieval quality directly. It targets **agent inference latency** in tool-heavy loops, especially with local backends.

Reasoning vs baseline latency:
- If your current system’s 45ms p50 is dominated by **retrieval** and network inference to managed APIs, KV cache preservation may have limited impact.
- If you run a local model where per-turn latency is dominated by repeatedly re-processing a large system prompt/tool schema, stabilizing the prefix can reduce effective prompt-processing per turn from "full prompt" to "only new tokens".
- Expected effect is largest in agent CLIs that resend large tool definitions and mutate headers.

## Claimed improvements (from the source)
The article’s core claim is qualitative/mechanistic: dynamic headers break prefix matching and therefore disable KV caching; removing them restores KV cache reuse and prevents repeated full-prompt reprocessing. (The article frames this as a practical fix for slow local inference.)

## How to benchmark against our baseline
Even though your baseline includes retrieval metrics, you can benchmark this technique as a **system optimization**:
1. Fix the RAG pipeline and queries (same retrieval settings).
2. Run a tool-using chat loop with a stable long system prompt and tool schema.
3. Compare two runs on a local backend:
   - A: dynamic attribution/telemetry header injected each turn (prefix changes)
   - B: header disabled; prefix stable
4. Measure:
   - end-to-end latency per turn (p50/p95)
   - tokens processed per turn (if server exposes)
   - CPU/GPU utilization

## Confidence
**Medium**: The causal mechanism (prefix changes → cache miss) is well-known for KV caching; the specific recommendation is grounded in the cited article, but the article is not a peer-reviewed benchmark and does not provide universal numeric deltas.
