# Benchmark report: Bandit-feedback LLM routing (one policy, many cost/latency/quality trade-offs)

## Source
- Wang Wei, Tiankai Yang, Hongjie Chen, Yue Zhao, Franck Dernoncourt, Ryan A. Rossi, Hoda Eldardiry. **"Learning to Route LLMs from Bandit Feedback: One Policy, Many Trade-offs"** (arXiv:2510.07429)
  https://arxiv.org/abs/2510.07429
  (PDF) https://arxiv.org/pdf/2510.07429

## Technique
Instead of static rules ("use small model unless confidence low"), learn a **routing policy** that chooses among multiple LLMs per request using **bandit feedback**:
- Observe context/features (prompt length, task type, risk, tool-use requirement, etc.).
- Choose a model (or cascade) to satisfy multi-objective constraints.
- Update policy online from partial feedback signals (task success, user rating, downstream score).
- Support multiple operating points (cost vs latency vs quality) from a single learned policy.

## What metric improves?
- **Cost**: lower average $/request by sending easy prompts to cheaper models.
- **Latency**: lower p50/p95 by avoiding unnecessary use of slow models.
- **Quality / success rate**: maintain or improve outcome metrics by escalating hard prompts.

## Theoretical comparison vs our baseline
Baseline (given):
- RAG retrieval: MRR@10 = 0.78
- Retrieval latency: p50 = 45ms
- Hybrid search: Cosmos DB vector + BM25 on 246 docs

Routing does not change retrieval MRR directly, but can improve end-to-end system objectives:
- If a large fraction of queries can be answered with a smaller model without harming answer quality, average cost and latency improve.
- For difficult queries (e.g., low-recall retrieval results, long contexts, or complex reasoning), the router can select a stronger model or trigger a cascade (cheap first, expensive fallback).

## Claimed improvements
The paper proposes learning-based routing from bandit feedback; this report does not quote specific numeric gains because the fetched PDF content was truncated in-tool (binary PDF). Use the arXiv paper to extract dataset/task-specific deltas before final KPI commitments.

## How to benchmark in our environment
1. Define a model pool: {small_local, mid_cloud, large_cloud}.
2. Define success metric(s): answer acceptance rate, tool-task completion, human/LLM judge score.
3. Implement a contextual bandit (e.g., LinUCB/Thompson sampling) over prompt features + retrieval features (MRR proxy, top-k score gap, context length).
4. Run A/B vs static routing baseline for 1–2k requests:
   - Compare total cost, p50/p95 latency, and quality metric at matched constraints.

## Classification
**BENCHMARKABLE** (multi-objective routing is measurable; compare cost/latency/quality vs static routing).
