# Benchmark Report — Background History Compaction Proxy (Context Gateway)

## Source
- **Context Gateway (Compresr)**
  - Docs: https://compresr.ai/docs/gateway
  - Repo: https://github.com/Compresr-ai/Context-Gateway

## Technique summary
Run a **transparent local proxy** between an agent and the LLM API that:
- Monitors token usage
- Pre-computes conversation summaries at a threshold (default 75%)
- Performs instant compaction when the limit is reached (summary already computed)
- Compresses large tool outputs on the fly
- Logs compaction metrics/events (JSONL)

## What metric improves?
- **Context utilization** (fewer tokens sent; avoid hard truncation).
- **Latency at compaction boundary**: avoids “stop-and-summarize” pauses by doing summarization in the background (claimed as “no more waiting” / “instant”).
- **Cost**: docs claim “Reduce token usage by 30–70%”.

## Claimed improvements (from source)
From the docs (https://compresr.ai/docs/gateway):
- “Cost savings: Reduce token usage by 30-70%.”
- “Compression happens in the background.”
- “Instant compaction when limit is reached — no waiting.”

## Theoretical comparison vs our baseline (RAG)
Our baseline focuses on retrieval (MRR@10=0.78; p50 latency=45ms for hybrid search over 246 docs). Context Gateway doesn’t change retrieval metrics directly. It impacts:
- **End-to-end cost**: fewer prompt tokens sent per request, especially in long agent sessions.
- **End-to-end latency**: avoids periodic compaction pauses and reduces payload size.

In a RAG agent, token costs are frequently dominated by conversation history + tool outputs. This technique targets that portion without changing Cosmos DB retrieval latency.

## How to benchmark (suggested)
- Replay a representative agent session trace (tool calls + responses) through:
  1) direct-to-LLM baseline
  2) LLM via Context Gateway proxy
- Measure:
  - tokens/request and total tokens
  - end-to-end p50/p95 latency
  - number of compaction events and pause time
  - cost per session

## Confidence
- **Medium**: claims are in vendor docs/repo, but not a peer-reviewed benchmark. Validate on our workloads.
