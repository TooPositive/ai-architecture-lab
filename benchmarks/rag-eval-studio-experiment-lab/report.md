# Benchmark report — RAG-Eval Studio (configuration benchmarking loop)

## Classification
**BENCHMARKABLE** — provides explicit KPIs and an experiment harness to compare RAG configuration choices (chunk size/strategy, retrieval settings) with measurable metrics.

## Source
- GitHub — **"RAG Eval Studio"**
- https://github.com/anergaratea/rag-eval-studio

## What metric improves?
Not a single technique that directly improves a metric; rather, it **enables systematic measurement** and *selection* of better configurations. Metrics in the source repo include:
- **`retrieval_hit_rate`**: whether retrieved context contains expected evidence
- **`avg_context_relevance`**: mean similarity score of retrieved chunks
- **`answer_overlap`**: token overlap between predicted and expected answer

These can be used as proxies to improve:
- retrieval accuracy (via hit rate / relevance)
- answer quality (via overlap)

## Technique (what to implement)
Create an **offline evaluation harness** that:
1. Ingests a document set (the repo supports Markdown; extend to your corpus).
2. Generates multiple retrieval indexes across a grid of configs (chunk sizes, overlap, top-k, retriever type).
3. Runs a fixed evaluation set (question + expected answer/evidence).
4. Produces a ranked table of configurations by KPI.

## Theoretical comparison vs our baseline (reasoned)
Baseline: **MRR@10=0.78**, **latency p50=45ms**, hybrid search (Cosmos DB vector + BM25) on 246 docs.

- RAG-Eval Studio does not report MRR@10 directly; it uses hit-rate/overlap-style KPIs. However, the same principle applies: by sweeping chunking/top-k/etc and selecting the best configuration, you can likely increase MRR@10 (or maintain it while reducing latency/cost).
- Expected impact magnitude cannot be claimed without running the harness on our corpus; the repo is explicitly designed to make that comparison easy and transparent.

## How to benchmark against our baseline
1. Add our evaluation metric: compute **MRR@10** on the same question set by logging the rank of the first relevant document/chunk.
2. Add latency measurement for retrieval + synthesis path; record p50/p95.
3. Run the configuration grid:
   - chunk size (e.g., 256/512/1024 tokens)
   - overlap (0/10%/20%)
   - top-k (5/10/20)
   - hybrid weights (BM25 vs vector blend)
4. Select Pareto-optimal points: maximize MRR@10 while meeting latency p50 <=45ms.

## Claimed/mechanistic justification from the source
- "RAG systems are difficult to improve if you do not measure them properly" (repo motivation).
- The repo provides an "Experiment Lab" to "Compare multiple configurations in one run" with transparent KPIs.
