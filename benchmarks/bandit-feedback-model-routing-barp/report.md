# Benchmark Report — Bandit-feedback Model Routing (BaRP)

## Technique
**BaRP: Bandit-feedback Routing with Preferences** — learn an LLM router under *partial feedback* (only observe outcome from the chosen model), and support **preference-tunable** routing at inference time (dial cost vs quality without retraining).

## Source
- **Learning to Route LLMs from Bandit Feedback: One Policy, Many Trade-offs** (arXiv:2510.07429, Oct 8 2025)
- https://arxiv.org/abs/2510.07429

## What metric improves?
- **Cost–quality efficiency** (better quality at same cost, or lower cost for same quality)
- **Routing robustness in production** (trained with deployment-realistic partial feedback rather than full-information labels)
- **Operator control**: tune preference vector at inference time to trade off accuracy vs cost/latency

## Claimed improvements (from source)
From the abstract:
- “Outperforms strong offline routers by **at least 12.46%** and the largest LLM by **at least 2.45%**” in comprehensive experiments.

## Theoretical comparison vs our baseline
**Our baseline**: single fixed policy hybrid retrieval pipeline (MRR@10=0.78, latency p50=45ms, hybrid search on 246 docs using Cosmos DB vector + BM25).

**Why this is comparable**: While our baseline numbers are for retrieval, our *end-to-end assistant* cost/latency often depends on **which LLM** we call after retrieval. A router changes downstream generation cost/latency/quality without changing retrieval.

### Expected effects
- **Latency**: routing some requests to smaller/faster models can reduce p50/p95 generation latency, especially for low-complexity queries.
- **Cost**: fewer tokens processed by expensive models; BaRP explicitly targets adaptive selection under varying prices and preferences.
- **Quality**: for hard queries, route to stronger models; BaRP aims to avoid overpaying while maintaining outcome quality.

### Why BaRP may outperform heuristic routers
- Production reality: you rarely can label *all candidate models* on every prompt (expensive). BaRP learns under the same **partial-feedback** constraint as deployment.
- Preference-tunable inference: instead of training separate routers for different budgets, BaRP supports a **preference vector** that can be changed at runtime.

## How to benchmark in our lab
Suggested evaluation (extend our baseline harness):
1. Keep retrieval fixed (Cosmos DB vector + BM25).
2. Define candidate LLM set: e.g., small local (routing/planning), mid-tier API, top-tier API.
3. Define reward proxy: task success (LLM-as-judge), user feedback, or pass/fail unit tests for tool tasks.
4. Compare routers:
   - Heuristic thresholds (token length, intent)
   - Offline supervised router (if feasible)
   - **BaRP-style bandit-trained router**
5. Metrics:
   - Outcome quality win-rate
   - $ cost / request
   - Latency p50/p95
   - Regret / reward over time (online learning)

## Confidence
High that the *paper exists* and claims improvements (source is arXiv). Medium on transferability to our system until we implement a comparable reward signal and candidate model set.
