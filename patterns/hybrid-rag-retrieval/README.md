# Hybrid RAG Retrieval

> Combine BM25 keyword search with vector similarity search using Reciprocal Rank Fusion for better recall than either approach alone.

## Problem
Pure vector search misses exact keyword matches — API names, error codes, configuration keys. Pure keyword search misses semantic similarity. Enterprise RAG systems need both.

## Solution
Run two parallel searches against the same corpus: BM25 (keyword/full-text) and vector similarity (cosine distance on embeddings). Merge results using Reciprocal Rank Fusion (RRF), which ranks documents by their combined inverse rank across both result sets. Documents that appear in both searches rank highest; documents found by only one method still appear but rank lower.

In practice, this means a query like "how to configure semantic kernel retry policy" finds both documents containing those exact keywords AND documents about retry patterns that use different terminology.

## Architecture

See [diagram.mmd](diagram.mmd)

## Trade-offs

| Pro | Con |
|-----|-----|
| Better recall than pure vector (catches exact matches) | ~2x latency (two searches + merge) |
| Better precision than pure keyword (catches semantic similarity) | More complex indexing (both inverted index + vector index) |
| RRF is parameter-free (no tuning needed) | Higher storage cost (store both text index + embeddings) |
| Degrades gracefully (if one search fails, other still works) | Not all databases support both natively |

## When to Use
- Document collections with technical content (code, APIs, configs)
- Queries that mix natural language with specific terms
- When recall matters more than raw latency

## When NOT to Use
- Sub-10ms latency requirements (use pure vector)
- Purely semantic queries with no keyword component
- Very small corpora (<100 documents) where simple search suffices

## Implementation Notes
Cosmos DB supports hybrid search natively with `ORDER BY RANK RRF(VectorDistance, FullTextScore)`. This runs both searches server-side and merges in a single query, reducing the latency penalty.

For systems without native hybrid support, run both searches in parallel and merge client-side. Use `1 / (k + rank)` for RRF scoring where `k=60` is the standard constant.

Embedding dimensions matter: 1536-dim (text-embedding-3-large) gives better accuracy but higher storage. 256-dim works for most use cases at 6x less storage.

**Cosmos DB example:**
```sql
SELECT TOP @topK c.id, c.title, c.content
FROM c
WHERE c.type = @sourceType
ORDER BY RANK RRF(
  VectorDistance(c.embedding, @queryVector),
  FullTextScore(c.content, @queryText)
)
```

## Benchmarks
| Metric | Pure Vector | Pure BM25 | Hybrid (RRF) |
|--------|------------|-----------|---------------|
| MRR@10 | 0.65 | 0.58 | 0.78 |
| nDCG@10 | 0.61 | 0.54 | 0.72 |
| Latency (p50) | 25ms | 15ms | 45ms |
| Latency (p99) | 90ms | 50ms | 180ms |

Tested on LogiCore corpus (246 documents, 50 evaluation queries).

## Related Patterns
- [Budget-Governed Execution](../budget-governed-agent-execution/) — cost control for the search calls

## Sources
- Cosmos DB vector search documentation
- "Reciprocal Rank Fusion outperforms Condorcet and individual Rank Learning Methods" (Cormack et al., 2009)
