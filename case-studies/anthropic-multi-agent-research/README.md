# Anthropic — Multi-agent Research system (Claude Research)

Source: https://www.anthropic.com/engineering/built-multi-agent-research-system

## Company / project
- Company: Anthropic
- Project: Research feature (multi-agent research system within Claude)

## Problem
Enable an AI system to perform open-ended research: multi-step web/tool use, breadth-first exploration, and synthesis across information that can exceed a single context window. Anthropic highlights that linear, one-shot pipelines struggle with dynamic, path-dependent research where intermediate findings change the next steps.

## Architecture choices & trade-offs (as described)
**High-level pattern:** orchestrator–worker multi-agent system.

- **LeadResearcher orchestrator agent**
  - Analyzes the user query, creates a research strategy, and spawns subagents to investigate in parallel.
  - Persists its plan to **external Memory** to survive very long runs / context truncation (Anthropic notes context can exceed **~200,000 tokens** and get truncated).
  - Iterates: synthesizes subagent findings; decides to refine strategy or spawn more subagents; exits once sufficient information is gathered.

- **Specialized Subagents (workers)**
  - Each gets a specific research task (objective + output format + boundaries).
  - Independently runs web searches and evaluates tool results (Anthropic mentions “interleaved thinking”).
  - Returns *compressed findings* to the lead agent (framed as “search is compression”; parallel agents increase capacity because each has its own context window).

- **CitationAgent post-processing step**
  - After the research loop, a dedicated agent processes the report + gathered documents to attach citations at specific locations, to ensure claims are attributed.

- **Prompting as the primary control lever**
  - Anthropic reports coordination failure modes during prototyping (e.g., spawning too many subagents, endless browsing, duplicated work) and emphasizes prompt engineering to manage these.
  - They embed “effort scaling rules” in prompts to adapt number of subagents / tool calls to query complexity.

**Trade-offs noted**
- **Pros:** strong for parallelizable research; mitigates context-window limits by parallel context windows + compression; can dynamically adapt strategy mid-run.
- **Cons:** expensive token usage; not all domains fit (they note many coding tasks have fewer truly parallelizable subtasks and real-time coordination remains hard).

## Results / metrics
- Internal evaluation: multi-agent system (**Claude Opus 4** lead + **Claude Sonnet 4** subagents) outperformed single-agent **Claude Opus 4** by **90.2%** on Anthropic’s internal research eval.
- BrowseComp analysis: three factors explained **95%** of performance variance: token usage (explains **80%** alone), plus number of tool calls and model choice.
- Token economics: agents use ~**4×** more tokens than chats; multi-agent systems use ~**15×** more tokens than chats.

## Comparison vs AgentHub (8 dimensions)
Baseline (AgentHub): Daily RSS/Reddit/HN/newsletter pipeline; Cosmos DB hybrid search (vector + BM25); 15 scheduled + 8 command agents; critic agent; evolution engine; budget gates; ~$30–50/mo infra + $5/day LLM budget; git push deploy with health checks/rollback.

1) **Data Pipeline** — **DIFFERENT**
- Them: not disclosed as a daily ingestion pipeline; described as tool-based research (web, Google Workspace, integrations) rather than a fixed daily feed collection.

2) **Retrieval** — **DIFFERENT**
- Them: emphasizes **dynamic multi-step search via agents** versus “static retrieval” RAG; no specific vector DB/BM25 stack or embedding model details disclosed in the article.

3) **Agents** — **THEM AHEAD**
- Evidence: mature orchestrator–worker multi-agent design with explicit lead/subagent roles + citation agent, used in a production feature.

4) **Evaluation** — **THEM AHEAD**
- Evidence: reports internal research eval (+90.2%) and BrowseComp factor analysis (95% variance explanation; token usage 80%).

5) **Self-Improvement** — **NOT DISCLOSED**
- They describe iterative prompt engineering and simulations during development, but no ongoing automated self-improvement/evolution engine described.

6) **Safety** — **NOT DISCLOSED**
- The article discusses cost implications and coordination controls, but doesn’t specify budget gates, governance, sanitization, or rate limiting mechanisms.

7) **Cost** — **DIFFERENT**
- Evidence: provides relative token multipliers (4×, 15×) but no direct $/month. AgentHub has explicit infra budget ranges and $/day LLM budget.

8) **Deployment** — **NOT DISCLOSED**
- It’s clearly production (“Research feature”) but deployment pipeline/rollback/health checks are not described.

## Confidence
High. This is a first-party engineering write-up with concrete architecture diagrams and quantitative claims.
