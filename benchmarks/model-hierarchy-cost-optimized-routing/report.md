# Benchmark report: Model Hierarchy (cost-optimized routing)

## Source
- **model-hierarchy-skill (OpenClaw skill): Cost-optimize AI agent operations by routing tasks to appropriate models based on complexity** (SKILL.md, Feb 2026 pricing table)
  https://github.com/zscole/model-hierarchy-skill/blob/main/SKILL.md

## Classification
**BENCHMARKABLE** — routing policies can be measured on cost, latency, and task success rate.

## What metric improves?
- **Cost**: reduce token spend by using cheaper models for routine work
- **Latency**: smaller/faster models often reduce response time for tool-heavy steps
- **Throughput**: more tasks per budget

## Technique summary
The source proposes an explicit *model tiering* policy for agent systems:
- classify tasks into **ROUTINE / MODERATE / COMPLEX**
- route each class to a model tier (Tier 1 cheap, Tier 2 mid, Tier 3 premium)
- apply overrides:
  - vision-capable routing for image tasks
  - escalation when a cheaper model fails

It also gives a concrete cost model and an example blended strategy (**80/15/5 split**) with an estimated monthly cost reduction versus “pure premium”.

## Theoretical comparison vs our baseline

### Our baseline
- RAG retrieval: MRR@10=0.78
- latency p50=45ms (retrieval layer)
- hybrid search on 246 docs using Cosmos DB vector + BM25

### Where model routing applies
This technique affects the **generation/orchestration** side (agent steps, tool use, synthesis), not directly the retrieval MRR@10.

However, end-to-end system performance can improve because:
- many steps in a RAG/agent workflow are “janitorial” (log parsing, file reads, formatting) and don’t require frontier reasoning
- routing those steps to cheap models lowers cost and may lower latency

### Expected measurable effects (reasoned)
- **Cost**: significant reduction if your workflow has many low-complexity steps.
- **Latency**: potential improvements if tool steps are gated by model response time (but note: tool/network latency may dominate).
- **Quality**: risk of regressions on tasks misclassified as routine; mitigation via escalation-on-failure.

## Claimed improvements (from source)
The SKILL.md includes a “Cost Impact” table and an example claim (illustrative):
- “Hierarchy (80/15/5) ~$19” monthly vs “Pure Opus ~$225” (for 100K tokens/day assumption).
These are not peer-reviewed results; treat as a benchmark hypothesis.

## How to benchmark
Design an experiment on your agent workload (not just retrieval):
1) Define a representative set of end-to-end tasks and success criteria.
2) Run with a single fixed model (control).
3) Run with routing policy + escalation.
Measure:
- task success rate
- total tokens by tier
- end-to-end latency p50/p95
- total cost (using provider pricing)

## Notes / risks
- Requires a reliable task classifier (rules, small router model, or learned policy).
- Adds orchestration complexity (telemetry, per-step model selection, fallbacks).
