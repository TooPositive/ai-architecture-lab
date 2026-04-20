# Benchmark report: MCP Context Gateway ("Context Mode") for context utilization + cost

## Source
- *Stop Burning Your Context Window — We Built Context Mode*
- https://mksg.lu/blog/context-mode

## Technique summary
Insert an MCP server between the agent and tools that **compresses tool outputs** (summaries/structured extraction + optional raw artifact handles) so the agent does **not** paste large payloads into the LLM prompt.

The source claims **315 KB → 5.4 KB (~98% reduction)** in tool-output context footprint.

## What metric improves?
- **Context utilization**: fewer input tokens consumed per tool call / per session.
- **Latency**: lower end-to-end latency if the dominant cost is LLM input processing (especially for long sessions).
- **Cost**: lower token-billed input and/or fewer truncation retries.

## Theoretical comparison vs our baseline
**Our baseline**: MRR@10=0.78, latency p50=45ms, hybrid search on 246 docs using Cosmos DB vector + BM25.

This technique does **not directly change retrieval** (so MRR@10 is not expected to move), but it can improve the *agent system* performance envelope:
- In a tool-heavy agent, prompt size frequently dominates latency and cost, not retrieval time.
- If tool outputs are large, a 98% reduction in tool-output bytes should reduce input tokens roughly proportionally, which can materially reduce LLM processing time and avoid context truncation that causes downstream errors.

**Expected direction vs baseline:**
- Retrieval accuracy: **≈ no change** (unless truncation currently removes retrieved evidence, in which case it may indirectly improve answer quality).
- Latency p50: **down** when prompt-token processing dominates (model-dependent).
- Cost: **down** when prompt caching or repeated prefixes are insufficient to offset large appended outputs.

## Claimed improvements (from source)
- **315 KB becomes 5.4 KB** for captured tool outputs (**~98% reduction**).

## How to benchmark in our lab
Suggested A/B experiment:
1. Use an agent workflow with repeated large tool outputs (e.g., browser snapshots or log retrieval).
2. Run baseline (no gateway) vs gateway.
3. Measure:
   - total input tokens per task
   - wall-clock latency per step + p50/p95
   - task success rate under a fixed context window
   - downstream answer quality when long sessions previously truncated evidence

## Confidence
**Medium.** The source provides concrete size-reduction numbers, but not a standardized benchmark methodology or model/latency measurements.
