# Benchmark Report: Bandit-Feedback Model Routing (BaRP)

## Source
- **Learning to Route LLMs from Bandit Feedback: One Policy, Many Trade-offs** (Wang et al., arXiv:2510.07429, 2025)
- https://arxiv.org/abs/2510.07429

## What metric improves?
Primarily **cost/quality efficiency** and **accuracy under budget constraints**, via better per-request routing decisions:
- Higher task performance than strong offline routers under partial-feedback realism
- Better than always using the largest LLM (reported +2.45% improvement)
- Better than strong offline routers (reported +12.46% improvement)

(Exact metrics depend on their tasks; paper frames this as routing performance, not retrieval MRR.)

## Technique summary
Most production routers are trained offline assuming labels/outcomes for *all* candidate models per prompt. In reality, you only observe the outcome from the model you picked. BaRP trains under that **bandit/partial-feedback** assumption and supports a **preference vector** so operators can adjust the quality↔cost trade-off at inference time without retraining.

## Theoretical comparison vs our baseline (reasoning)
**Our baseline**: single fixed model + RAG with MRR@10=0.78, latency p50=45ms, hybrid search over 246 docs (Cosmos DB vector + BM25).

BaRP is orthogonal to retrieval. Expected improvements would be on **LLM invocation cost and tail latency**, while trying to preserve answer quality:
- Route easy questions (or “high-retrieval-confidence” queries) to a cheaper/smaller model.
- Escalate to a stronger model when:
  - retrieval confidence is low (e.g., low similarity margin, low BM25 overlap)
  - the question requires synthesis or tool use
  - safety/accuracy risk is high

If we add routing on top of our RAG baseline, we can plausibly improve:
- **Cost**: fewer calls to expensive models on easy queries
- **Latency**: lower p50 and possibly lower p95 by handling more queries with faster models
- **Quality**: potentially maintained or slightly improved if escalation catches hard cases

Key point from the paper: the router can be trained under the same partial-feedback constraints encountered in deployment, reducing mismatch versus offline-supervised routing.

## Claimed improvements (from source)
From arXiv abstract:
- “outperforms strong offline routers by at least **12.46%**”
- “outperforms … the largest LLM by at least **2.45%**”
- supports “preference-tunable inference … without retraining”

## How we would benchmark it in our lab
Even though our baseline is retrieval-heavy, we can benchmark routing with end-to-end task metrics:
- **Answer quality**: task success rate or LLM-judge win rate; for RAG QA also measure MRR@10 separately to ensure retrieval unchanged
- **Latency**: p50/p95 end-to-end
- **Cost**: tokens * price across model pool
- **Escalation rate**: % requests routed to strong model

Suggested A/B:
- A: always-strong model
- B: static heuristic router (length/keywords)
- C: learned router (BaRP-like)

Confidence: **Medium** (paper reports improvements; mapping to our specific RAG baseline requires implementation and a model pool).
