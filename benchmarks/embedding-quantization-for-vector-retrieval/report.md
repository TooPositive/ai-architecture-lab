# Benchmark: Embedding quantization (binary/scalar) for faster, cheaper vector retrieval

## Source
- **Binary and scalar embedding quantization for faster, cheaper vector retrieval** (Hugging Face blog; internal KB summary)
  (KB node: `search-knowledge-9be082ac-d8ff-4f29-aab4-af8b8351d75a`)

## What metric improves?
- **Latency**: faster similarity computations (e.g., bit operations / Hamming distance for binary).
- **Cost**: reduced memory footprint and storage costs; higher cache hit rates; lower infra requirements.
- **Throughput**: more vectors can be searched per second with the same hardware.

Potential downside metric:
- **Retrieval accuracy** (MRR/Recall): may drop depending on quantization aggressiveness.

## Technique
Instead of storing embeddings as fp16/fp32 floats, store compressed forms:
- **Scalar quantization**: represent values with fewer bits (e.g., int8-like).
- **Binary quantization**: represent embeddings as bit vectors; compute similarity with Hamming distance / XOR-popcount.

## Theoretical comparison vs our baseline
Baseline (your environment):
- Retrieval: **MRR@10=0.78**
- Latency: **p50=45ms**
- Hybrid search over **246 docs** using Cosmos DB vector + BM25

Reasoning vs baseline:
- With only 246 docs, quantization likely won’t matter much; the baseline is already small-corpus.
- If you scale to larger corpora (10^5–10^7 vectors), quantization can substantially reduce memory bandwidth pressure and ANN index size, improving p50/p95 latency and cost.

Expected effect directions:
- **Latency p50/p95**: should improve as distance computation and memory access become cheaper.
- **MRR@10**: may decrease slightly; you can mitigate with 2-stage retrieval:
  1) search over quantized vectors to get a candidate set
  2) re-score candidates using full-precision embeddings (or cross-encoder)

## Claimed improvements (from source)
- The source claims **significant** storage/cost reduction and speed improvements, with **small** quality trade-offs depending on method.
  (The internal KB summary does not provide explicit numeric deltas.)

## How to benchmark it in our lab
Even with 246 docs, we can benchmark correctness and latency; for meaningful results, also benchmark on a scaled synthetic corpus.

A/B setup:
- A: fp32/fp16 embeddings
- B: scalar-quantized embeddings
- C: binary embeddings
- Optional: B/C + re-score stage

Metrics:
- Retrieval quality: MRR@10, Recall@K against labeled queries.
- Runtime: p50/p95 query latency, CPU%, memory footprint, index build time.

Confidence: **Medium** (well-known family of methods, but source summary lacks hard numbers).
