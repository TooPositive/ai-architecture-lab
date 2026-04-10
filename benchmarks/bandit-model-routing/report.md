# Benchmark: Bandit-based Adaptive Model Routing (cascading local→cloud with learned policy)

## Source
- **Adaptive Model‑Routing Middleware for .NET Agents: Cost, Latency & Accuracy‑Aware Switching Between Local and Cloud LLMs**
  - https://arxiv.org/html/2510.07429

## What metric improves?
- **Cost** (token spend) by sending "easy" prompts to cheaper models.
- **Latency** (especially tail latency) by avoiding slow models when not needed.
- **Quality / task success rate** maintained via escalation/fallback for hard prompts.

## Claimed / described improvement (from source)
The source describes learning routing policies online using **(contextual) bandit feedback** instead of fixed thresholds, optimizing multi-objective tradeoffs (cost/latency/accuracy) and adapting as models/prices/prompts change.

## Theoretical comparison vs our baseline
Baseline (given):
- RAG: MRR@10=0.78
- Latency p50=45ms
- Hybrid search on 246 docs using Cosmos DB vector + BM25

Routing is orthogonal to retrieval MRR@10; it affects generation cost/latency and potentially answer quality.

### Expected effect in our system
- For internal doc QA where many questions are simple lookups, a router can:
  - Run cheap/local LLM for draft answer
  - Escalate to larger model when confidence is low or when evaluator signals failure
- This can reduce average $/query while keeping accuracy near the large-model level.

### Benchmark design
- Dataset: a representative set of prompts (easy/medium/hard) with ground-truth evaluation signals (human label or automatic task score).
- Compare:
  1) Always-large model
  2) Fixed rule router (e.g., based on prompt length / heuristic)
  3) Bandit router (contextual bandit)
- Metrics:
  - $/query, tokens/query
  - latency p50/p95
  - task success rate / human preference
  - escalation rate

## Notes / risks
- Requires a feedback signal (explicit user rating, implicit success proxy, or judge model).
- Bandit policies can drift if feedback is noisy or delayed.
