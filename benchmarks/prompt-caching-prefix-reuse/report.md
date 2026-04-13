# Benchmark: Prompt Prefix Caching (Input Token Discounts / KV Reuse)

## Source
- OpenAI API Docs — *Prompt caching*
  https://developers.openai.com/api/docs/guides/prompt-caching

## Technique
Structure your application prompts so a **large, stable prefix** repeats across requests (system prompt, tool schemas, policies, shared context), enabling provider-side prompt caching. The OpenAI API can apply **discounted pricing** to **input tokens that were recently seen** by the model (no explicit developer-managed cache required).

## What metric improves?
- **Cost**: reduced billed input-token cost for repeated prefixes.
- **Latency**: the guide states prompt caching can reduce latency (because repeated prefix work can be reused), though the most explicit, measurable claim is cost discounting.

## Theoretical comparison vs our baseline
Baseline: MRR@10=0.78, latency p50=45ms on retrieval (Cosmos DB vector + BM25, 246 docs).

Reasoning:
- This technique does **not** directly change retrieval quality (MRR) because it targets the **generation call**.
- It can reduce **end-to-end latency** if LLM prompt processing is a large portion of overall latency (especially with long tool definitions / system prompts).
- It can reduce **per-request cost** for agentic systems with big stable tool schemas or policies.

## Claimed improvements (from source)
- The documentation states the API provides **automatic discounts** when a request includes **input tokens the model has recently seen** (prompt caching), targeting repeated/shared prompt prefixes.

## How we would benchmark it
- Hold retrieval constant; measure:
  1) **Input-token billed cost** across repeated runs with identical prefix and varying user message suffix.
  2) **End-to-end p50/p95 latency** for the model call with and without a stable prefix (and by moving dynamic fields *after* the stable prefix).

## Confidence
High that this is benchmarkable (cost/latency) and that the source documents the mechanism; medium on magnitude for our specific workload (depends on prefix stability and request cadence).
