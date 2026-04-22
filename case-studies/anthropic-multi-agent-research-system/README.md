Source: https://www.anthropic.com/engineering/multi-agent-research-system

# Anthropic — Multi-agent Research system (Claude Research) — Architecture case study

## Company / project
- **Company:** Anthropic
- **Project:** Claude "Research" feature (multi-agent research system)
- **Article:** *How we built our multi-agent research system* (Jun 13, 2025)
- **URL:** https://www.anthropic.com/engineering/multi-agent-research-system

## Problem they solved
Anthropic wanted to enable an AI assistant to handle open-ended research tasks that require dynamic, path-dependent exploration (searching, following leads, and synthesizing), which is hard to capture in a fixed one-shot or linear pipeline. The system needed to search across the web and user-connected sources, operate for many turns, coordinate work, and still produce attributable answers with citations.

## Architecture choices and trade-offs
### Core architecture
- **Orchestrator–worker multi-agent pattern:** a **LeadResearcher** agent plans and coordinates the research loop and **spawns parallel Subagents** for different facets of a query.
- **Iterative research loop:** lead agent decides whether to continue research and can spawn additional subagents as it learns more.
- **Memory for persistence:** the LeadResearcher saves its plan to "Memory" to persist context in case the conversation exceeds a large context window and earlier tokens are truncated.
- **CitationAgent stage:** a dedicated **CitationAgent** processes the gathered documents and the research report to attach citations to specific locations/claims before returning the final answer.

### Tooling / reliability / evaluation approach
- **Prompt engineering as primary control lever:** they highlight failure modes (e.g., spawning too many subagents, endless searching, duplicated work) and address them largely via prompting rules and decomposition instructions.
- **Scaling rules:** embed rules in prompts to scale tool calls / number of subagents with query complexity to manage cost and reduce runaway behaviors.
- **Simulated runs for debugging prompts:** built simulations using their Console with the same prompts/tools to observe agent behavior step-by-step.

### Key trade-offs called out
- **Cost vs performance:** multi-agent systems consume far more tokens; they report typical **~15× more tokens than chats** (and agents generally **~4×** more tokens than chat interactions). This requires use cases where the value justifies cost.
- **Parallelism fit:** domains with heavy dependencies or needing shared context may fit poorly; they note coding tasks often have fewer truly parallelizable subproblems than research today.

## Results and metrics
- **Internal eval uplift:** multi-agent system (lead **Claude Opus 4** + subagents **Claude Sonnet 4**) outperformed single-agent Claude Opus 4 by **90.2%** on Anthropic's internal research evaluation.
- **Variance drivers (BrowseComp analysis):** token usage explained **80% of performance variance**; (tool calls, model choice) were other key factors; together three factors explained **95% of variance**.
- **Token usage:** multi-agent research systems use about **15× more tokens** than chats; agents typically use **~4× more tokens** than chat interactions.

---

# Comparison vs AgentHub (8 dimensions)
Baseline: AgentHub (Azure Functions + Cosmos hybrid search + scheduled agents + critic/evolution + safety gates).

| Dimension | Anthropic (Research) vs AgentHub | Notes (from article) |
|---|---|---|
| 1. Data Pipeline | **NOT DISCLOSED** | Article focuses on live multi-agent web/workspace search, not a daily ingestion pipeline. |
| 2. Retrieval | **DIFFERENT** | Not described as a Cosmos-style hybrid search over a fixed corpus; instead an iterative, tool-driven search with subagents. |
| 3. Agents | **THEM AHEAD** | Clear orchestrator–subagent design with parallel subagents and specific coordination mechanisms. |
| 4. Evaluation | **THEM AHEAD** | Explicit internal eval results (90.2% improvement) and analysis of performance variance; dedicated citation stage (quality/attribution). |
| 5. Self-Improvement | **DIFFERENT** | They iterate heavily via prompt engineering + simulations; no autonomous "evolution engine" described. |
| 6. Safety | **NOT DISCLOSED** | Mentions preventing runaway behavior (too many subagents, endless search) via prompt rules; broader governance/budget controls not detailed. |
| 7. Cost | **DIFFERENT** | Concrete token-multiplier discussion (15× tokens); dollars not disclosed. AgentHub has explicit infra + budget numbers. |
| 8. Deployment | **NOT DISCLOSED** | Production deployment/rollback/health-check mechanics not covered in the article. |
