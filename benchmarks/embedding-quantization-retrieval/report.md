# Benchmark: Embedding Quantization for Faster/Cheaper Retrieval

## Source
- **Binary and scalar embedding quantization for faster, cheaper vector retrieval** (Hugging Face blog; summarized in internal knowledge base)
  - Knowledge node: `Binary and scalar embedding quantization for faster, cheaper vector retrieval`

## What metric improves?
Primary improvements are typically:
- **Latency (p50/p95)** for vector similarity search (faster distance computations; better cache locality).
- **Cost / memory footprint** of the vector index (smaller embeddings → less RAM/disk; potentially fewer nodes).
Secondary potential improvements:
- **Throughput (QPS)** for retrieval.

Retrieval accuracy usually **degrades slightly** depending on quantization level (int8 vs binary).

## Technique summary
Quantize stored embedding vectors from fp16/fp32 to:
- **Scalar quantization** (e.g., int8-like representation), or
- **Binary quantization** (bit vectors; compare via Hamming/bit ops)

This reduces storage and accelerates similarity search.

## Theoretical comparison vs our baseline
Baseline environment:
- Corpus: **246 docs**
- Retrieval: **Cosmos DB vector + BM25 hybrid**
- Baseline metrics: **MRR@10=0.78**, **latency p50=45ms**

Expected effects with quantization (reasoned):
- **Latency**: likely improves if the vector scan/ANN dominates p50. Smaller vectors reduce compute per similarity and improve memory bandwidth. However, on 246 docs, you may already be dominated by network/DB overhead rather than similarity compute, so gains could be modest unless you are CPU-bound.
- **Cost**: if using a managed vector DB priced by RU/CPU/memory, quantization can reduce memory footprint and improve cache hit rates, potentially reducing RU or allowing smaller SKU. With only 246 docs, absolute savings are small, but the technique becomes meaningful as corpus grows (10^5–10^8 vectors).
- **Retrieval quality (MRR@10)**: may drop slightly; hybrid BM25 can partially compensate by preserving lexical signals.

## Claimed improvements (from source)
The source claims quantized embeddings can yield **significantly cheaper and faster retrieval** with **small quality trade-offs**, comparing full-precision vs low-bit representations (scalar/binary).

## How to benchmark in our stack
1. Export existing embeddings and build a quantized variant (int8 and/or binary).
2. Rebuild the vector index with quantized vectors (or use a library/index that supports them, e.g., FAISS/PQ style; DB support varies).
3. Run the same evaluation set:
   - Measure **MRR@10**, **recall@10**
   - Measure **latency p50/p95** end-to-end and retrieval-only
   - Measure memory usage / cost (if self-hosted)

## Confidence
- **Medium** that latency/cost improvements exist in general (well-established technique).
- **Low-to-medium** that it materially improves our specific baseline at 246 docs, because overhead may not be similarity-compute bound.
