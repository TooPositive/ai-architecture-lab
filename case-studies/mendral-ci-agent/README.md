# Mendral — Production CI investigation agent

Source: https://www.mendral.com/blog/how-we-built-our-ai-agent

## Company / project
- Company: Mendral
- Project: Mendral’s AI agent that investigates CI failures (GitHub App + Slack integration)

## Problem
Teams running large volumes of CI jobs struggle to diagnose and fix failures efficiently—especially with increased CI activity and flaky tests driven by faster code generation. Mendral’s focus is automated CI investigations and fixes, where the key signal is not only the codebase but also massive, historical CI log and execution data across runs and branches.

## Architecture choices & trade-offs (as described)
### 1) Data & logs as a first-class system input
- Mendral built a **log ingestion pipeline** that processes **billions of CI log lines per week** into **ClickHouse**, compressed **35:1**, and “queryable in milliseconds.”
- The agent can write its own **SQL** to investigate failures; they cite a typical investigation scanning **335K rows** across **3+ queries**, and at P95 scanning **940 million rows**.

**Trade-off:** building/operating a high-throughput log analytics backend (ClickHouse + ingestion) is heavier than relying purely on repository context, but enables investigations that general coding agents can’t do.

### 2) “One agent” product experience, “team of agents” internally
- Externally: one agent user experience via GitHub App + Slack.
- Internally: specialized agents coordinated via Mendral’s **Go backend**, using **all three Anthropic model tiers (Haiku, Sonnet, Opus)** with routing by task type:
  - **Opus**: root cause analysis + writing non-trivial fixes
  - **Sonnet**: fact collection, deduping issues, evidence gathering, SQL generation
  - **Haiku**: log parsing, classification, structured extraction at high throughput

**Trade-off:** complexity of multi-model routing and orchestration, but improves cost predictability and uses higher capability only where it matters.

### 3) Custom agent loop in Go (not an off-the-shelf framework)
- Mendral runs an agent loop on their **Go backend** for execution control, concurrency, and failure handling; they explicitly state they do **not** use LangChain/LangGraph/off-the-shelf agent frameworks.
- Loop: trigger → assemble context → LLM call → tool calls → iterate until conclusion or budget exhausted.

### 4) Tooling split: in-process tools vs sandboxed execution
- Fast, deterministic tools run in-process (Go functions): query ClickHouse, fetch GitHub metadata, repo structure, PR status, etc.
- Potentially dangerous operations (clone repos, run tests, apply patches, execute arbitrary code) run in isolated sandboxes: **Firecracker microVMs on Blaxel**.
- Performance/cost optimization: sandboxes boot **<125ms**, resume **<25ms**; sandboxes are **suspended between tool calls** so compute isn’t held during LLM inference.
- Sandboxes can also suspend for long waits (e.g., hours while CI runs), and resume with full state.

**Trade-offs:** microVM orchestration adds operational complexity, but provides strong tenant isolation and reduces idle compute cost during LLM latency/waits.

### 5) Durable execution for reliability
- Mendral uses **Inngest** for “durable execution” for both agent loop and ingestion pipeline.
- Work is broken into retryable steps with memoized state, so a failure on a downstream API call doesn’t force re-running expensive earlier steps (including long LLM calls).
- Handles rate limits by pausing and resuming using Retry-After + jitter.

**Trade-off:** adopting a durable execution platform changes how the system is structured (step boundaries, idempotency), but improves production reliability.

## Results / metrics
- Scale signals: “**200K+ CI jobs per week**” (customer context).
- Agent throughput: “**16,000+ CI investigations a month**” completed autonomously.
- Data system scale: “**billions of CI log lines per week**” ingested; **35:1** compression; typical investigation scans **335K rows**; **P95 940M rows**.
- Other metrics (accuracy, time saved, $ cost): not disclosed in the article.

## Comparison vs AgentHub (8 dimensions)
Baseline (AgentHub): Daily RSS/Reddit/HN/newsletter pipeline; Cosmos DB hybrid search; 15 scheduled + 8 command agents; critic agent; evolution engine; budget gates; ~$30–50/mo infra + $5/day LLM budget; git push deploy with health checks/rollback.

1) **Data Pipeline** — **THEM AHEAD**
- Evidence: high-scale operational data pipeline (billions of log lines/week) into ClickHouse with strong compression and queryability; far beyond AgentHub’s content ingestion scale (but different domain).

2) **Retrieval** — **DIFFERENT**
- Them: retrieval is primarily **analytical querying over structured log data** (SQL over ClickHouse) + GitHub metadata; not described as vector/BM25 hybrid.
- Us: hybrid text retrieval over articles (Cosmos DB vector + BM25).

3) **Agents** — **THEM AHEAD**
- Evidence: production multi-agent system coordinated via Go backend, with explicit multi-model routing (Haiku/Sonnet/Opus) and specialized responsibilities.

4) **Evaluation** — **NOT DISCLOSED**
- No explicit eval harness, automated scoring, or critic loop described (beyond operational reliability patterns).

5) **Self-Improvement** — **NOT DISCLOSED**
- They mention iterating on model routing over time, but no automated evolution engine described.

6) **Safety** — **THEM AHEAD**
- Evidence: strong isolation via **Firecracker microVMs** per sandbox session and destruction after use; hardware-level isolation emphasized. AgentHub notes sanitization/governance but not microVM isolation.

7) **Cost** — **DIFFERENT**
- Them: focuses on cost control via model tiering + sandbox suspend/resume; no $/month disclosed.
- Us: explicit infra budget and LLM budget gate numbers.

8) **Deployment** — **NOT DISCLOSED**
- Inngest + Go backend + microVMs implies substantial production deployment, but CI/CD, rollbacks, health checks not described.

## Confidence
High. First-party write-up with concrete components (ClickHouse, Firecracker, Blaxel, Inngest) and specific scale numbers.
