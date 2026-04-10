# Benchmark: Embedding Quantization for Vector Retrieval (Scalar/Binary)

## Source
- **"Binary and scalar embedding quantization for faster, cheaper vector retrieval"** (Hugging Face blog; summarized in internal KB node `search-knowledge-9be082ac-d8ff-4f29-aab4-af8b8351d75a`).

## What metric improves?
- **Latency / throughput** of vector similarity search (faster comparisons; smaller memory bandwidth).
- **Cost** (lower storage footprint; potentially cheaper infra).
- Potentially **cache efficiency** (more vectors fit in RAM/CPU cache).

## Technique summary
Quantize stored embedding vectors from fp16/fp32 into:
- **Scalar quantization** (e.g., int8-like values)
- **Binary quantization** (bit-packed vectors; similarity via bit operations / Hamming distance)

Retrieval uses approximate distance on quantized vectors (optionally followed by re-ranking on full precision for top-K).

## Theoretical comparison vs our baseline
**Baseline**: MRR@10=0.78, latency p50=45ms, hybrid search on 246 docs using Cosmos DB vector + BM25.

### Expected impact
- On a small corpus (246 docs), absolute p50 improvements may be modest, because the dominant cost can be network + query overhead rather than brute-force vector math.
- Still benchmarkable: quantization can reduce CPU time and memory pressure, which matters when:
  - scaling to larger corpora
  - consolidating infra
  - running on edge / constrained environments

### Hypothesis (benchmark plan)
- Keep hybrid BM25+vector unchanged, but replace vector index with quantized embeddings.
- Measure:
  1. **p50/p95 latency** for retrieval
  2. **MRR@10 / nDCG@10** (expect small degradation for binary; smaller for scalar)
  3. **Storage size** of embeddings + index

### Why it should help
- Smaller representations reduce the number of bytes read per candidate and speed up similarity computations (especially binary bitwise operations), which should reduce the vector component of latency and increase throughput.

## Claimed improvements (from source)
The source frames quantization as enabling **"significantly cheaper and faster retrieval"** with "small quality trade-offs" by compressing embeddings into scalar or binary representations (KB node `search-knowledge-9be082ac-d8ff-4f29-aab4-af8b8351d75a`).

## Confidence
- **High** that quantization reduces storage and vector compute cost (widely used idea).
- **Medium** that it improves *our* p50 latency at 246 docs, because overheads may dominate at this scale.
