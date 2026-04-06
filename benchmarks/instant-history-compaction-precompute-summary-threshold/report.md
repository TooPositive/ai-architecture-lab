# Benchmark report: Precompute summary at threshold to make compaction instant (Context Gateway)

## Source
- Compresr. **"Context Gateway" documentation**
  https://compresr.ai/docs/gateway

## Technique summary
Instead of compacting only when the context limit is reached (blocking the user), trigger **background summarization** at a threshold (docs: default 75%). When the limit is reached, swap in the precomputed summary to achieve *instant* compaction with no pause.

## What metric improves?
- **Latency / responsiveness at the context limit**: eliminates the "compaction pause" on the critical path.
- **Context utilization**: reduces token usage by replacing history with a summary.
- **Cost**: doc claims token usage reduction (docs state "Reduce token usage by 30–70%").

## Theoretical comparison vs our baseline
Baseline (given):
- Retrieval: MRR@10=0.78
- Latency p50=45ms
- Hybrid search on 246 docs

This technique is orthogonal to retrieval quality; it improves agent loop UX and potentially end-to-end latency when context becomes large.

Reasoning:
- If your p50=45ms is measured on short turns, it won’t capture the main benefit. The benefit appears as a large reduction in p95/p99 at the moments compaction would have occurred.
- As conversations grow, summarization in-line can add seconds. Precomputation moves that cost off the critical path, so user-facing turn latency remains closer to "normal" even near context limits.

## Claimed improvements (from the source)
- Docs state: "Compression happens in the background" and "Instant compaction when limit is reached — no waiting".
- Docs state cost savings: "Reduce token usage by 30–70%".

## How to benchmark against our baseline
1. Use a fixed workload: long-running agent session with periodic large tool outputs.
2. Compare:
   - A: naive compaction only at limit (blocking summarization call)
   - B: threshold precompute at 75% + instant swap
3. Metrics:
   - turn latency distribution (p50/p95/p99), specifically turns near the context limit
   - total tokens sent to LLM
   - number of compaction operations and their time

## Confidence
**Medium**: Technique is clearly described by the vendor docs; numeric improvements (30–70% token reduction) are claimed in docs but are workload-dependent and not independently validated in the source.
