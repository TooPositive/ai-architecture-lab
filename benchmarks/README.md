# Benchmarks

Continuous benchmark results from a production RAG pipeline (LogiCore). New techniques are benchmarked against the current baseline whenever they're detected in collected articles.

## Current Baseline

| Metric | Value | Configuration |
|--------|-------|---------------|
| Retrieval latency (p50) | 45ms | Cosmos DB hybrid search |
| Retrieval latency (p99) | 180ms | |
| MRR@10 | 0.78 | 246-doc corpus |
| nDCG@10 | 0.72 | |
| Embedding model | text-embedding-3-large | 1536 dimensions |
| Search method | Hybrid (BM25 + Vector, RRF) | Cosmos DB native |
| Chunking | Recursive, 1000 tokens, 200 overlap | |

## Methodology

Each benchmark:
1. Runs against the same corpus (LogiCore default, 246 documents)
2. Uses the same evaluation queries (50 hand-labeled query-document pairs)
3. Reports: latency (p50, p95, p99), accuracy (MRR, nDCG), and cost
4. Runs 3x for statistical significance
5. Compares against the current baseline above

## Results

| Week | Technique Tested | vs Baseline | Report |
|------|-----------------|-------------|--------|
| *First benchmark coming April 2026* | | | |
