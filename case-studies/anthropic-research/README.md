# Anthropic — Multi-agent Research system (Claude Research)
Source: https://www.anthropic.com/engineering/multi-agent-research-system

## Company / project
- **Company:** Anthropic
- **Project:** Claude “Research” feature (multi-agent research system)
- **Article:** “How we built our multi-agent research system” (Jun 13, 2025)

## Problem they solved
Users ask open-ended questions that require **multi-step web/integration search**, iterative refinement, and synthesis across more information than fits in a single context window. A single-agent, single-pass RAG-style retrieval is too static for this class of tasks.

## Architecture choices & trade-offs (as described)
**High-level pattern:** orchestrator–worker (lead agent + parallel subagents) with post-processing for citations.

- **LeadResearcher (orchestrator):**
  - Analyzes the query, creates a strategy, and **spawns specialized subagents**.
  - Runs an **iterative loop**: synthesize findings → decide if more research is needed → spawn/refine subagents.
  - Persists the plan to **Memory** to survive long runs / context truncation (article cites 200k-token context and the need to keep the plan if truncated).

- **Subagents (workers):**
  - Execute **independent web searches** in parallel.
  - Use “interleaved thinking” to judge tool results, identify gaps, and adjust queries.
  - Return condensed findings to the LeadResearcher (parallel “compression”).

- **CitationAgent:**
  - After research completion, a separate agent processes the documents + research report to **attach citations** to specific claims.

- **Prompting as a primary control surface:**
  - They observed coordination failures (e.g., spawning ~50 subagents for simple queries; endless searching; excessive chatter).
  - They encoded **effort-scaling heuristics** into prompts (simple fact-finding vs comparisons vs complex research).
  - They emphasized **tool selection heuristics** and the importance of clear tool descriptions.

- **Evaluation approach:**
  - They used an **LLM judge** scoring outputs against a rubric (factual accuracy, citation accuracy, completeness, source quality, tool efficiency) and found a **single LLM call** with 0–1 scoring + pass/fail to be most consistent.
  - They started with a **small eval set (~20 queries)** early to catch large deltas from prompt changes.

- **Reliability / operations:**
  - Agents are long-running and stateful; they built **durable execution** that can resume from errors, plus deterministic safeguards (retry logic, checkpoints).
  - Added production tracing/observability to diagnose failure modes; they also monitor high-level interaction patterns without inspecting conversation contents (privacy).

**Trade-offs explicitly called out:**
- **Cost:** multi-agent architectures “burn through tokens fast.” They report agents use about **4×** more tokens than chat interactions, and multi-agent systems about **15×** more tokens than chats.
- **Fit limitations:** domains requiring shared context or high inter-agent dependency (e.g., many coding tasks) are less suitable today.

## Results & metrics (from the article)
- Multi-agent system (Claude Opus 4 lead + Claude Sonnet 4 subagents) outperformed single-agent Claude Opus 4 by **90.2%** on Anthropic’s internal research eval.
- In BrowseComp variance analysis: token usage alone explains **~80%** of performance variance; with tool calls + model choice, three factors explain **~95%**.
- Speed: adding parallelization (subagents + parallel tool calls) cut research time by **up to 90%** for complex queries.
- Cost: typical agent interactions use **~4×** tokens vs chat; multi-agent systems use **~15×** tokens vs chat.

## Comparison vs AgentHub (8 dimensions)
Baseline for comparison: AgentHub (RSS/Reddit/HN ingestion; Cosmos DB hybrid search; scheduled + command agents; critic; evolution engine; safety/cost gates; Azure Functions deployment).

| Dimension | Anthropic Research system vs us | Notes (what the article actually says) |
|---|---|---|
| 1. Data Pipeline | **DIFFERENT** | Focus is live research across web + Google Workspace + integrations, not a daily article ingestion pipeline. Details of their ingestion/indexing are not disclosed. |
| 2. Retrieval | **THEM AHEAD** | Uses multi-step search with parallel subagents; contrasts with static RAG. Specific vector/BM25 infra not disclosed. |
| 3. Agents | **THEM AHEAD** | Explicit orchestrator-worker multi-agent design with dynamic subagent spawning + parallel tool calls. |
| 4. Evaluation | **DIFFERENT** | They describe LLM-judge rubric + small eval set early. We have a goal critic + completion review; not directly comparable. |
| 5. Self-Improvement | **THEM AHEAD** | They mention a “tool-testing agent” that rewrites MCP tool descriptions and yielded a **40% decrease in task completion time** for future agents using the updated description. That’s a concrete self-improvement loop. |
| 6. Safety | **NOT DISCLOSED** | They discuss privacy-aware observability; no concrete budget gates / governance workflow described in the excerpt captured. |
| 7. Cost | **DIFFERENT** | They quantify token blow-up (4× / 15×) and note economic viability constraints; we publish explicit $ budgets and infra costs. |
| 8. Deployment | **NOT DISCLOSED** | Mentions “deployment needs careful coordination” but no concrete CI/CD architecture or rollback mechanisms in captured text. |

## Confidence
- **High** for multi-agent architecture, token multipliers, and eval metrics (directly stated in the article).
- **Medium** for the “Memory” mechanism details (mentioned, but implementation specifics are not described).
- **Low/Not disclosed** for storage/retrieval stack, security controls, and deployment mechanics.
