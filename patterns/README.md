# Architecture Patterns

Implemented patterns for enterprise AI systems. Each pattern includes:
- Problem statement and solution description
- Mermaid architecture diagram
- Trade-offs (every pattern has downsides)
- When to use / when NOT to use
- Implementation notes with code references
- Benchmark data where available

## Catalog

| Pattern | Category | Summary |
|---------|----------|---------|
| [Hybrid RAG Retrieval](hybrid-rag-retrieval/) | Retrieval | BM25 + vector search with Reciprocal Rank Fusion |
| [Autonomous Goal Loop](autonomous-goal-loop/) | Orchestration | Stateless goal iteration for crash-safe agent execution |
| [Evolution Engine](evolution-engine/) | Self-Improvement | Execution trace reflection with auto-apply proposals |
| [Evaluator Gate](evaluator-gate-for-self-modification/) | Safety | Independent LLM review before self-modification merges |
| [Budget-Governed Execution](budget-governed-agent-execution/) | Cost Control | Hierarchical budget limits for autonomous agents |

## How Patterns Are Added

1. New technique detected in collected articles (9,300+ sources)
2. Pattern Lab agent classifies: new pattern vs. known vs. benchmarkable
3. If new: implementation goal spawns, writes code + docs + diagram
4. If benchmarkable: benchmark goal runs against LogiCore baseline
5. Results published here with auto-generated ADR
