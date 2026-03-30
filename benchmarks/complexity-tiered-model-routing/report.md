# Benchmark: Complexity-Tiered Model Routing (Model Hierarchy)

## Source
- **zscole/model-hierarchy-skill: OpenClaw skill for cost-optimized model routing based on task complexity** (GitHub)
- https://github.com/zscole/model-hierarchy-skill

## Classification
**BENCHMARKABLE**

## What metric improves?
- **Cost ($/task, $/1K tokens)**: route routine tasks to cheap models; reserve premium models for complex tasks.
- Secondary: **latency** may improve for routine tasks if cheaper models are faster (not claimed explicitly in the source).

## Technique summary
A router (policy + classifier) labels each agent action as one of three tiers and selects a model accordingly:
- ROUTINE (~80%): cheapest model for file ops, lookups, formatting
- MODERATE (~15%): mid-tier for summaries, drafts, code gen
- COMPLEX (~5%): premium for debugging, architecture, novel problems

The source provides an illustrative cost table and claims “~10x cost reduction with equivalent quality on the tasks that matter” for the tiered mix.

## Theoretical comparison vs our baseline
**Our baseline** (given): hybrid search on 246 docs using Cosmos DB vector + BM25; MRR@10=0.78, latency p50=45ms.

Model routing is largely orthogonal to retrieval quality (MRR) unless model choice affects query rewriting, chunk selection, or reranking. The primary expected change is **end-to-end system cost** for agentic workflows that include many “routine” steps.

### Reasoning (cost)
If a large fraction of steps in an agent run are mechanical (reading files, formatting, listing issues), then a fixed-premium-model approach overpays. A tiered router can reduce total spend by assigning these steps to cheaper models while preserving premium capacity for the small subset of tasks that need it.

The source provides a worked example (100K tokens/day) showing a large drop in monthly cost for a tiered mix vs running everything on a premium model.

### How to benchmark (recommended)
1. **Trace-based replay**: collect N real agent sessions; break into discrete steps (tool call planning, summarization, drafting, debugging).
2. **Offline labeler**: label steps by tier (or use the skill’s rubric).
3. **A/B routing simulation**:
   - A: always-premium model
   - B: tiered router (80/15/5 or learned distribution)
4. Metrics:
   - Total cost per session
   - Success rate / task completion score (human or automated rubric)
   - Optional: time-to-completion and p50/p95 latency

## Claimed improvements (from source)
- “**~10x cost reduction** with equivalent quality on the tasks that matter.”

## Confidence
- **Medium**: routing concept is standard, but the source provides concrete tier distribution + a cost math example and an implementation artifact (skill + tests). Exact savings depend heavily on your task mix and model pricing.
