# ADR-001: Hybrid Search Over Pure Vector

## Status
Accepted

## Date
2025-08-15

## Context
LogiCore's RAG pipeline initially used pure vector search (cosine similarity on text-embedding-3-large). During evaluation, we found that queries containing exact terms — API names like `SemanticKernel.AddAzureOpenAIChatCompletion`, error codes like `HTTP 429`, config keys like `maxTokens` — returned semantically related but wrong documents. The vector embedding captured the meaning but lost the exact string match.

We needed a retrieval approach that handles both semantic similarity and exact keyword matching.

## Decision
Use hybrid search combining BM25 full-text search with vector similarity, merged using Reciprocal Rank Fusion (RRF). Cosmos DB supports this natively via `ORDER BY RANK RRF(VectorDistance, FullTextScore)`, so both searches run server-side in a single query.

## Alternatives Considered
1. **Pure vector with metadata filtering** — Add exact-match metadata fields (API names, error codes) and filter before vector search. Rejected: requires knowing which terms need exact matching in advance, and the metadata schema becomes a maintenance burden.
2. **Pure vector with higher dimensions** — Use 3072-dim embeddings for better precision. Rejected: higher storage cost, and the fundamental issue is keyword vs. semantic, not precision within semantic.
3. **Two-stage retrieval** — Vector search first, then re-rank with BM25. Rejected: adds latency and complexity of a separate re-ranking step. Hybrid RRF does this in one pass.

## Consequences
### Positive
- MRR@10 improved from 0.65 (pure vector) to 0.78 (hybrid)
- Exact keyword queries now return correct results
- Single Cosmos DB query (no client-side merging needed)

### Negative
- ~2x latency increase (45ms p50 vs 25ms for pure vector)
- Requires both vector index and full-text index on the container
- Debugging search results is harder (need to understand both ranking systems)
