# Benchmark report: Task-complexity model hierarchy routing

## Source
- **model-hierarchy-skill** (GitHub) — "Cost-optimize AI agent operations by routing tasks to appropriate models based on complexity" — https://github.com/zscole/model-hierarchy-skill

## Technique summary
Classify each agent step by complexity (ROUTINE / MODERATE / COMPLEX) and route to an LLM tier accordingly (cheap / mid / premium). Default sub-agents to cheap models unless a task demands higher reasoning.

## What metric improves?
- **Cost** (primary) — token spend shifts from premium to cheaper tiers.
- Potentially **latency** (secondary) — smaller/cheaper models can be faster, though this depends on provider/runtime.
- **Quality** should be held constant by escalating only the hard steps.

## Claimed improvements (from source)
- Source claims **~10× cost reduction with equivalent quality on the tasks that matter** via a typical mix: **80% routine, 15% moderate, 5% complex**.
- Example cost math in source (assuming 100K tokens/day):
  - Pure premium: ~$225/month
  - Pure mid: ~$45/month
  - Hierarchy (80/15/5): ~$19/month

## Theoretical comparison vs our baseline
Our baseline is a RAG system with:
- MRR@10=0.78
- latency p50=45ms
- hybrid search on 246 docs using Cosmos DB vector + BM25

This technique does **not** change retrieval (MRR@10) directly; it changes the **generation/orchestration cost** of agentic steps around retrieval.

### Expected impact (reasoned)
- **Retrieval accuracy:** no change (same retriever + prompt).
- **Latency p50:** could improve for ROUTINE steps if served by faster models; may increase slightly if misclassification causes escalation retries.
- **Cost per user query / per run:** should decrease roughly in proportion to the share of tokens moved off premium models.

### What we can benchmark
For an agentic RAG workflow (retrieve → cite → answer → optional follow-up tool calls):
1. Instrument per-step token counts and chosen model tier.
2. Compare total cost/run vs always using a single premium model.
3. Add a quality guard (human eval or automated rubric) to ensure no degradation on final answers.

## Notes for experiment design
- Implement a classifier (rules or lightweight model) to label steps ROUTINE/MODERATE/COMPLEX.
- Add an escalation policy: if a step fails (tool error, low confidence), retry on next tier.
- Track: cost/run, latency/run, retries, and answer quality.
