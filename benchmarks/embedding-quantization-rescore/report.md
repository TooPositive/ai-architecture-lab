# Benchmark: Embedding Quantization + Float Rescoring (Binary / Int8)

## Source
- *Binary and Scalar Embedding Quantization for Significantly Faster & Cheaper Retrieval* — Hugging Face Blog (Aamir Shakir, Tom Aarsen, SeanLee) — 2024-03-22
  https://huggingface.co/blog/embedding-quantization

## Technique
Store document embeddings in **quantized form** (binary bit-packed or scalar int8). For retrieval, use fast distance on quantized vectors to get a candidate set, then **rescore** the candidates with a higher-fidelity similarity (e.g., float32 query embedding vs binary doc embeddings).

The source explicitly claims:
- **Binary quantization** gives ~**32× reduction** in memory/storage (1-bit vs float32) and can improve retrieval speed **up to 32×**.
- With rescoring, you can preserve **~96%** of retrieval performance vs full precision; without rescoring ~**92.5%**.

## What metric improves?
- **Latency / throughput**: faster similarity computation (bit operations / Hamming) and smaller index memory footprint.
- **Cost**: less RAM/disk for the vector index → cheaper infrastructure.
- **(Potentially) context utilization**: indirect, via faster retrieval enabling higher `top_k`/reranking, but not directly claimed.

## Theoretical comparison vs our baseline
Baseline: **MRR@10=0.78**, **latency p50=45ms**, hybrid search on **246 docs** using **Cosmos DB vector + BM25**.

Reasoning:
- On a small corpus (246 docs), quantization is unlikely to produce meaningful absolute latency gains because brute-force scanning is already cheap; most of our 45ms likely comes from network/DB overhead and reranking.
- On larger corpora, the source argues quantization’s main impact is **index size + compute per similarity**, which scales with corpus size. Quantization is therefore **highly benchmarkable** when we scale beyond hundreds of docs, or when we move to a local ANN index where distance computations dominate.
- If we keep hybrid BM25+vector, the biggest expected benefit is: **vector side becomes cheaper/faster**, allowing either: (a) higher vector candidate pool for the same cost/latency, or (b) lower infra cost for the same quality.

## Claimed improvements (from source)
- Memory/storage: **32× reduction** for binary quantization.
- Speed: **up to 32×** faster retrieval speed.
- Quality retention: **~96%** of full-precision retrieval performance with rescoring; **~92.5%** without rescoring.

## How we would benchmark it
- **Retrieval quality**: compare MRR@10 and Recall@k on our eval set with (1) full precision embeddings vs (2) quantized doc embeddings + float rescoring.
- **Latency**: measure p50/p95 retrieval latency for the vector stage at increasing corpus sizes (e.g., 246 → 25k → 250k docs).
- **Cost**: measure index size on disk + RAM usage.

## Confidence
High that the technique is benchmarkable and the claims are as stated in the source article; medium that it will materially improve our current baseline at 246 docs (likely not).
