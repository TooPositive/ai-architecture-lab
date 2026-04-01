# Benchmark Report — Mixture-of-Models Routing (Hypernova)

## Source
- **Hypernova: Beating Single-Model Baselines with Mixture of Models** (Nordlys Labs, Jan 6 2026)
- https://nordlyslabs.com/blog/hypernova

## Technique summary
Route each task to the best-performing model for its **semantic cluster**, rather than using a single LLM for all tasks. Hypernova does:
1) Embed problem description (sentence-transformer)
2) Assign to nearest cluster centroid (vector similarity)
3) Select model with highest historical success rate in that cluster

## What metric improves?
- **Task success rate / accuracy** on SWE-bench Verified (software engineering tasks).
- Secondary (implied): cost/performance if you add cost-aware routing (mentioned as future work).

## Claimed improvements (from source)
The article states Hypernova “exceeds single-model baselines on SWE-bench” and shows example per-model success rates from the leaderboard:
- Claude Opus 4.5: 74.4%
- Gemini 3 Pro: 74.2%
- Claude Sonnet 4.5: 70.6%

It highlights complementarity: “65 tasks that Opus failed were solved by other models”, and “23 tasks that Opus failed were solved by Sonnet”, and “42 tasks Sonnet failed were solved by Opus”.

## Theoretical comparison vs our baseline (RAG)
Our baseline is retrieval-focused (MRR@10=0.78; latency p50=45ms; hybrid search on 246 docs using Cosmos DB vector + BM25). Hypernova is an **LLM routing** technique, so it does not directly change retrieval MRR or vector latency.

However, it can be benchmarked in our stack on:
- **End-to-end task success** (e.g., answer correctness, tool-task completion rate) for agent workflows.
- **Cost**: route routine requests to cheaper models while escalating hard requests to premium.
- **Latency**: may increase slightly due to embedding+lookup, but article claims this overhead is milliseconds vs LLM seconds.

If we apply it to RAG Q&A:
- Potential accuracy gain comes from using a stronger model *only* on queries detected as complex/ambiguous, while using cheaper models for routine queries.
- Potential cost reduction comes from shifting high-volume easy queries to cheaper models.

## How to benchmark (suggested)
1. Create a labeled eval set of queries/tasks with ground-truth answers and difficulty tags.
2. Train routing table: cluster embeddings of query text; compute per-cluster win-rate for each candidate model.
3. Compare:
   - Single-model baseline (one model for all queries)
   - Router policy (cluster → best model)
4. Report:
   - Accuracy / success rate
   - Cost per successful task
   - Latency distribution (router overhead + model response)

## Confidence
- **High** that semantic-cluster-based model routing is described as improving SWE-bench performance (explicitly stated).
- **Medium** on exact deltas vs single-model baselines because the article excerpt available to us did not include a precise numeric “Hypernova = X%” in text; confirm from the included figure in the article if needed.
