# Benchmark: Bandit-feedback + Preference-Tunable Model Routing (BaRP)

## Source
- **Learning to Route LLMs from Bandit Feedback: One Policy, Many Trade-offs**
  - URL: https://arxiv.org/html/2510.07429

## What metric improves?
- **Cost-effectiveness** (accuracy at a given cost, or cost at a given accuracy)
- **Quality**: claims to outperform (a) strong offline routers and (b) the largest single LLM in their candidate set.
- **Generalization** to unseen tasks (out-of-distribution).

## Claimed improvements (from paper)
- Outperforms strong offline routers by **at least 12.46%** and the largest LLM by **at least 2.45%**; robust generalization to unseen tasks.
- Provides **preference-tunable inference** (dial performance vs cost at test time without retraining).

## Theoretical comparison vs our baseline (reasoned)
Our current RAG baseline metrics are retrieval-centric (MRR@10=0.78, latency p50=45ms, hybrid search on 246 docs using Cosmos DB vector + BM25). BaRP is not a retrieval technique; it is a **model selection layer** that sits *after* retrieval and before generation (or tool use).

Expected impacts if added to our stack:
- **Latency**: could improve *average* latency by routing easy queries to cheaper/faster models, but may increase tail latency due to exploration and escalation policies.
- **Cost**: likely reduced by sending a large fraction of prompts to smaller models, reserving expensive models for harder prompts.
- **Answer quality**: can be maintained or improved by escalating hard prompts. The paper claims to beat even the largest model in their pool, suggesting effective specialization-by-routing.

Key reasoning difference vs a static cascade:
- Static thresholds (token length, heuristic complexity) tend to drift as workloads/pricing/models change.
- BaRP trains under **bandit feedback** (only observe outcome of chosen model), matching real deployment conditions; it can keep learning online.

## How to benchmark in our environment
To make this benchmarkable alongside our baseline, we would measure:
1. **End-to-end answer quality** on our evaluation set (e.g., exact match / LLM-judge / task success).
2. **Cost per query** (tokens * price across models).
3. **Latency p50/p95** end-to-end (retrieve + generate).
4. **Regret / learning curve** over time if training online.

Compare configurations:
- Single best model (no routing)
- Simple cascade (cheap-first with escalation on low confidence)
- BaRP-style contextual bandit router

## Classification
- **BENCHMARKABLE** (paper provides measurable claims and a clear method comparison).
