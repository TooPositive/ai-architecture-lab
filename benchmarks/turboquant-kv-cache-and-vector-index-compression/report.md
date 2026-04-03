# Benchmark report: TurboQuant for KV-cache + vector index compression

## Source
- **TurboQuant: Redefining AI efficiency with extreme compression** (Google Research, Mar 24, 2026)
  https://research.google/blog/turboquant-redefining-ai-efficiency-with-extreme-compression/
- **TurboQuant (ICLR 2026)** arXiv:2504.19874 (PDF)
  https://arxiv.org/pdf/2504.19874

## Classification
**BENCHMARKABLE** — This is a quantization/compression technique with measurable impacts (memory footprint, throughput/latency, and potentially retrieval quality / attention accuracy).

## What metric improves?
Primarily:
- **Cost**: memory footprint of KV cache and vector storage
- **Latency / throughput**: faster similarity lookups from reduced memory bandwidth and improved cache residency
Secondary:
- **Context utilization** (indirect): smaller KV cache enables longer contexts within fixed VRAM

## Technique summary
TurboQuant is described as a theoretically-grounded vector quantization approach that aims to compress high-dimensional vectors for:
- **KV cache compression** in transformers
- **Vector search engines** (embedding indexes)

The blog claims TurboQuant achieves **high reduction in model size with “zero accuracy loss”** by combining:
- **PolarQuant**: random rotation + coordinate transform (polar coordinates) to avoid per-block normalization constants that cause memory overhead
- **QJL (Quantized Johnson–Lindenstrauss)**: uses a **1-bit sign** representation plus a special estimator to remove bias (“eliminating hidden errors”)

## Theoretical comparison vs our baseline

### Our baseline (for reference)
- Retrieval: **MRR@10 = 0.78**
- Latency: **p50 = 45ms**
- Corpus: **246 docs**
- Hybrid retrieval: **Cosmos DB vector + BM25**

### Where TurboQuant fits
TurboQuant targets the *vector representation and storage layer* (embedding vectors in a vector index; and KV cache vectors inside a model), not the BM25 layer.

In a Cosmos DB vector + BM25 hybrid setup, the closest analogue would be:
- compressing the embedding vectors used by the vector search component (either in the DB if supported, or in an application-side ANN index)

### Expected measurable effects (reasoned)
1) **Latency** (p50) could improve if vector similarity computation becomes memory-bandwidth bound and the index fits better in memory/cache.
   - With only **246 docs**, you likely won’t see large improvements vs 45ms unless your current implementation is dominated by vector compute/storage overhead rather than network/DB latency.
   - At larger corpora, reduced memory footprint and bandwidth should matter more.

2) **Retrieval accuracy (MRR@10)**
   - If quantization induces distance distortion, MRR can degrade.
   - The Google Research blog explicitly positions TurboQuant as achieving compression with **“zero accuracy loss”** (claim; must be validated against your embedding model + similarity metric + ANN method).

3) **Cost**
   - Storing full float32 embeddings is expensive at scale. Even float16 halves it; 8-bit/4-bit quantization improves further.
   - TurboQuant is explicitly designed to reduce *quantization overhead* (extra bits for per-block constants) that partially negates compression. If that overhead is a meaningful fraction of your storage, TurboQuant’s advantage becomes measurable.

## How to benchmark against baseline (suggested experiment design)
- Build two vector indexes over the same 246 docs and queries:
  1) baseline embeddings stored as float16/float32
  2) embeddings stored/encoded using TurboQuant
- Keep BM25 component identical.
- Measure:
  - MRR@10 on your eval query set
  - p50/p95 end-to-end latency
  - RAM/VRAM usage for the vector index

## Claimed improvements (from sources)
- Google Research blog: TurboQuant “achieves a high reduction in model size with **zero accuracy loss**” and is “ideal for supporting both KV-cache compression and vector search.”
- The accompanying paper (ICLR 2026) is the authoritative source for the quantitative results; review the paper’s reported compression ratios and accuracy curves for the specific tasks.

## Notes / risks
- Implementation complexity is non-trivial versus standard int8 product quantization.
- Whether you can apply this inside Cosmos DB depends on whether the service supports custom vector quantization; otherwise you’d benchmark in an app-side index (FAISS/ScaNN/HNSWlib) or a DB that supports quantization controls.
