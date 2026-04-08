# Benchmark: Bandit-learned model routing middleware (cost/latency/quality trade-offs)

## Source
- **"Adaptive Model‑Routing Middleware for .NET Agents: Cost, Latency & Accuracy‑Aware Switching Between Local and Cloud LLMs"**
  https://arxiv.org/html/2510.07429

## What metric improves?
- **Cost** (token spend)
- **Latency** (especially tail latency via local-first routing + escalation)
- **Quality** (maintain quality by routing “hard” prompts to stronger models)

## Technique
Use an adaptive router that selects among multiple LLMs per request using **(contextual) bandit feedback** rather than fixed thresholds.

Inputs/features can include:
- prompt length / estimated difficulty
- task type (chat, coding, RAG answer synthesis)
- tool-use requirement
- risk/safety classification
- latency SLO and budget

Feedback signals:
- user rating
- downstream task success/failure
- automated eval score
- “fallback triggered” events

## Theoretical comparison vs our baseline
Baseline (given):
- MRR@10 = 0.78
- latency p50 = 45ms
- hybrid search on 246 docs (Cosmos DB vector + BM25)

Routing impacts primarily **generation** cost/latency and “answer quality under budget”, not retrieval MRR directly.

Reasoning:
- Our baseline likely uses a single fixed model for answer synthesis.
- A router can send easy questions to a cheaper/faster model (or local) and only escalate to a premium model when uncertainty/difficulty is high.
- With bandit learning, the routing policy adapts as traffic and model pricing/quality drift, reducing brittle hand-tuning.

## Claimed improvements
The source argues that bandit-based and contextual-bandit routing can learn per-request decisions from partial feedback and optimize multi-objective trade-offs (cost/latency/accuracy) more robustly than static routing policies.

## How to benchmark against our baseline
A/B/C setup:
- A: single “strong” model
- B: static rules (length threshold + fallback)
- C: bandit/contextual-bandit router

Metrics:
- cost per successful answer
- latency p50/p95
- answer quality proxy (LLM-as-judge or task success)
- escalation rate

Expected outcome:
- Lower average cost and latency at similar quality, with improved robustness to drift.
