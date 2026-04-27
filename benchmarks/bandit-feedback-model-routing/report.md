# Benchmark report — Bandit-feedback model routing with preference-tunable trade-offs (BaRP)

## Source
- **Learning to Route LLMs from Bandit Feedback: One Policy, Many Trade-offs** (arXiv)
- https://arxiv.org/html/2510.07429

## Technique summary
Train a **router policy** that selects among multiple LLMs per request using **bandit feedback** (only observe the chosen model’s outcome), while conditioning on a **preference vector** to shift the cost–quality trade-off at inference time without retraining.

## What metric improves?
This technique targets measurable improvements in:
- **Cost**: lower average $/request by choosing smaller/cheaper models when appropriate.
- **Latency**: lower p50/p95 latency by routing easy prompts to faster models.
- **Quality**: maintain or improve task scores by escalating hard prompts to stronger models.
- **Operator control**: explicit preference knob to move along a Pareto frontier (cost vs performance).

## Claimed improvements (from the paper)
The paper reports that its method (BaRP):
- Outperforms “strong offline routers” by **≥12.46%**.
- Outperforms the **largest LLM by ≥2.45%**.
(See abstract in the arXiv HTML.)

## Theoretical comparison vs our baseline (reasoned)
Our baseline is a RAG system with **MRR@10=0.78, latency p50=45ms** on a small corpus (246 docs) using Cosmos DB vector + BM25 hybrid search.

BaRP is not a retrieval algorithm; it’s a **model selection layer** that sits above RAG. The expected effects relative to baseline:

### Retrieval accuracy (MRR@10)
- **No direct improvement** to retrieval MRR@10, because retrieval stays the same.
- **Indirect improvement** possible in end-to-end answer quality if the baseline’s generator is sometimes underpowered: harder queries can be routed to a stronger model for better synthesis grounded in retrieved docs.

### Latency p50
- **Likely improves** if many prompts can be handled by a smaller/faster model while preserving correctness.
- Trade-off: some prompts will be escalated (higher latency), so p95 may increase unless you enforce a latency-aware preference vector or hard cap.

### Cost
- **Likely improves** significantly for workloads with a long tail of “easy” prompts. The router can default to a cheap model and only pay for expensive calls when signals indicate difficulty.

### Context utilization
- Not directly; but smaller models might require stricter prompt budgets, which can force tighter retrieval/top-k selection strategies.

## How to benchmark against our baseline
1. Keep retrieval fixed (Cosmos DB vector+BM25) to isolate routing effects.
2. Add 2–4 candidate LLMs (e.g., small local, mid-tier cloud, high-end cloud).
3. Define a bandit feedback signal (e.g., task success, human rating, or automatic judge score).
4. Compare:
   - End-to-end task score (exact match / rubric score / judge score)
   - Cost per successful answer
   - Latency p50/p95
   - Escalation rate (fraction routed to expensive model)

## Notes / risks
- Bandit feedback means you don’t get labels for models you didn’t call; training stability and exploration strategy matter.
- Need a reliable reward signal; if noisy, routing may drift.

## Confidence
- **High** that this is benchmarkable: the paper is explicit about performance and cost trade-offs and reports comparative gains.
- **Medium** on transfer to our specific RAG baseline: impact depends on how often generation quality (vs retrieval) is the bottleneck.
