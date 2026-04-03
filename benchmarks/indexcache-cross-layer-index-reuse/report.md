# Benchmark report — IndexCache (cross-layer sparse-attention index reuse)

## Technique
IndexCache accelerates *DeepSeek Sparse Attention (DSA)* by caching and reusing the lightning-indexer’s **top‑k token indices across layers**, instead of recomputing an O(L²) indexer at every layer. Layers are partitioned into **Full (F)** layers (compute indices) and **Shared (S)** layers (reuse nearest F layer’s indices).

## What metric improves?
- **Latency / throughput (prefill + decode) for long-context inference**
- **Serving cost** (fewer indexer computations)

## Claimed improvements (from source)
From the paper’s abstract and repository README:
- Remove **~75% of indexer computations** with negligible quality loss.
- On a **30B DSA model**: up to **1.82× prefill speedup** and **1.48× decode speedup** vs standard DSA at long context.

## Theoretical comparison vs our baseline (reasoning)
Baseline (given): **RAG** on 246 docs with **hybrid Cosmos DB vector + BM25**, MRR@10=0.78, latency p50=45ms.

IndexCache does **not** change retrieval quality (MRR@10) directly; it targets the **LLM inference stage**, especially when you:
- run **long-context agentic workflows** (large tool outputs, long threads), and/or
- use models employing **sparse attention with a per-layer indexer** (e.g., DSA-family).

If your end-to-end RAG latency is dominated by LLM prefill (common when contexts are large), then IndexCache’s **~1.82× prefill speedup** implies a proportional reduction in that portion of latency. Example: if 70% of your p50 is LLM prefill, a 1.82× prefill speedup would reduce overall p50 by ~ (0.70/1.82 + 0.30) ≈ 0.684 → **~31.6% lower p50**. If retrieval dominates (45ms with small prompts), gains are minimal.

## When it’s benchmarkable in our stack
Benchmark end-to-end latency & cost under:
- long prompts (50k–200k tokens)
- DSA-capable model backend (SGLang / vLLM with DSA model)

Measure:
- p50/p95 latency split (retrieval vs prefill vs decode)
- tokens/sec
- GPU utilization and cost per 1k output tokens
- quality regression on internal eval set

## Source
- **Paper:** *IndexCache: Accelerating Sparse Attention via Cross-Layer Index Reuse* (arXiv:2603.12201) — https://arxiv.org/abs/2603.12201
- **Implementation:** THUDM/IndexCache — https://github.com/THUDM/IndexCache

## Confidence
High — the speedup numbers and method are stated directly in the paper abstract and the official implementation README.