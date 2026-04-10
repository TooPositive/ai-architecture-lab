# Benchmark: Prompt-prefix stabilization to enable KV-cache / prefix-matching reuse (local inference)

## Source
- **Claude Code with local backends: KV cache invalidation caused by dynamic system-prompt injection (60-second issue) and settings.json fix** (Reddit PSA; internal KB summary)
  (KB node: `search-knowledge-7ab40724-38b2-49ad-ba9b-70f52013dd14`)

## What metric improves?
- **Latency** (especially p95/p99): enabling prefix matching / KV-cache reuse prevents re-processing the full system prompt every request.
- **Cost / compute** (for self-hosted inference): reduces repeated forward-pass compute over unchanged prompt prefixes.
- **Context utilization (effective)**: fewer repeated tool-definition tokens per turn if the client stops mutating the system prefix.

## Technique
Many local backends (e.g., llama.cpp servers) can reuse computation when the **prompt prefix is identical** across requests (KV cache / prefix matching).

The source reports a concrete failure mode:
- Claude Code injects **changing telemetry headers** and **git status updates** into the system prompt on every request, which makes the prefix different each turn.
- That breaks prefix matching, **flushes the KV cache**, and forces re-processing of a large system/tool-definition prompt from scratch.

Mitigation proposed in the source:
- Adjust `~/.claude/settings.json` to reduce/disable the dynamic prompt mutations so prefix matching works again.

## Theoretical comparison vs our baseline
Baseline (your environment):
- Retrieval: **MRR@10=0.78**
- Latency: **p50=45ms**
- Hybrid search over **246 docs** using Cosmos DB vector + BM25

This technique does not target retrieval quality. It targets **model-side runtime** overhead in agent loops.

Expected impact (reasoning):
- If the system prompt/tool schema is large (the source cites ~20K+ tokens / 62,600 chars of tool definitions per turn), then breaking cache reuse causes repeated full-prefill cost.
- Stabilizing the prompt prefix should reduce per-turn compute to mostly **decode** + minimal delta-prefill, improving p50/p95 latency for tool-heavy agent sessions.

How it relates to your baseline:
- Your p50=45ms is likely dominated by retrieval + network; in an agentic workflow with multiple short turns, local LLM “prefill” may dominate.
- This technique is benchmarkable by measuring end-to-end agent turn latency with/without prefix stabilization under identical tool schemas.

## Claimed improvements (from source)
- The source describes a “**60-second** issue” when KV cache is invalidated by dynamic system-prompt injection and indicates that the configuration fix restores normal performance.
  (Qualitative claim; no controlled benchmark numbers provided.)

## How to benchmark it in our lab
Measure with a fixed agent script that performs N repeated tool-call turns, holding the tool schema constant:
1. **Unstable prefix**: intentionally inject a changing header/date/git status line into the system prompt each turn.
2. **Stable prefix**: keep the system prompt identical across turns.

Metrics:
- LLM latency p50/p95 per turn (and total wall time).
- Tokens processed per request (input tokens billed/processed).
- CPU/GPU utilization for self-hosted backend.

Confidence: **Medium** (clear mechanism; source is anecdotal and lacks published measurements).
