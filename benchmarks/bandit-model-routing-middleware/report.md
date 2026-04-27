# Benchmark: Bandit-Based Model Routing Middleware (Cascading/Adaptive Router)

## Source
- **Adaptive Model‑Routing Middleware for .NET Agents: Cost, Latency & Accuracy‑Aware Switching Between Local and Cloud LLMs**
- https://arxiv.org/html/2510.07429

## What metric improves?
- **Cost**: route “easy” prompts to cheaper models (often local or smaller).
- **Latency**: route “easy” prompts to faster models; add fallbacks/escalations only when needed.
- **Quality/accuracy**: preserve quality by escalating “hard” prompts to stronger models.

## Theoretical comparison vs our baseline
Baseline (given) is a RAG retrieval baseline (MRR@10=0.78, p50 latency=45ms on hybrid search). Model routing is mostly orthogonal to retrieval quality, but it affects end-to-end latency and cost of the **generation** step and any agent/tool loops.

Expected effects relative to baseline system behavior:
- **If generation dominates latency**: routing can reduce p50 and especially p95 by using faster/cheaper models on the majority of requests.
- **If retrieval dominates latency** (45ms p50 retrieval): routing may have smaller impact on p50 but can still cut token cost.

Why bandits matter (vs static routing):
- Static rules (e.g., “if prompt length < X then use small model”) are brittle when models/prices shift.
- Contextual bandits can learn routing using observed outcomes (success/failure, user rating, task score) and prompt features (task type, safety risk, tool use needed).

## Claimed improvements / key claims from source
From the paper summary in the KB entry:
- Routing policies can be learned **online** with **bandit feedback**, avoiding hard-coded thresholds.
- Supports **multi-objective** optimization (cost/latency/accuracy) and Pareto trade-offs.

(*The provided snippet does not include numeric deltas; benchmark should validate with our own workload.*)

## How we would benchmark vs baseline
Hold retrieval constant (Cosmos DB hybrid). Add router in front of LLM calls.

Measure:
- $/request (tokens + model pricing)
- latency p50/p95 end-to-end
- quality proxy (LLM-as-judge score, task success rate, or human rating)
- escalation rate (% routed to large model)

Experiment design:
- A/B: static routing vs contextual bandit routing
- Online learning over a fixed request stream

## Confidence
- **Medium**: bandit routing is established in research; the source is specific and recent.
- Numeric improvement claims not included in the snippet; requires measurement in our stack.
