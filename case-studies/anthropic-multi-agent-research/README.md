Source: https://www.anthropic.com/engineering/multi-agent-research-system

# Anthropic — Multi-agent Research system (Claude Research) — architecture case study

## Company / project
- **Company:** Anthropic
- **Project:** Claude **Research** (multi-agent research feature)
- **Primary write-up:** https://www.anthropic.com/engineering/multi-agent-research-system

## Problem they solved
Anthropic wanted a production system that can handle **open-ended, multi-step research** tasks where the steps can’t be predetermined. A single-agent, one-shot RAG-style pipeline was insufficient because research requires iterative searching, branching, and synthesis across many sources and tools (web, Google Workspace, integrations).

## Architecture choices and trade-offs
### High-level design
- **Multi-agent orchestrator/worker pattern**:
  - A **LeadResearcher** agent plans the research strategy and manages the loop.
  - The lead **spawns multiple subagents** to explore different aspects of the query **in parallel**, each with its own context window and tool-use loop.
  - A **CitationAgent** post-processes the collected documents + report to attach citations to specific claims.

### State + memory management
- The LeadResearcher **saves its plan to “Memory”** to preserve intent/strategy when the conversation exceeds the (very large) context window and content gets truncated.
- Production reliability emphasis: agents are long-running and stateful, so Anthropic highlights the need for **durable execution**, **retry logic**, and **regular checkpoints** to avoid catastrophic failure on transient tool issues.

### Tooling + prompting guardrails
- Anthropic treats **prompt engineering** as the primary lever for multi-agent coordination behaviors (e.g., preventing “spawn 50 subagents” failure modes).
- They embed explicit heuristics into prompts for:
  - scaling effort vs. query complexity (e.g., small fact-finding vs. complex research)
  - starting with broad searches then narrowing
  - selecting tools appropriately (tool descriptions matter)
- They introduced **two levels of parallelism**:
  1) lead agent spawns multiple subagents in parallel
  2) subagents make multiple parallel tool calls
  This reduced research time (they report up to 90% reduction for complex queries).

### Evaluation approach
- Anthropic reports using an **LLM-as-judge** evaluation with rubric dimensions including factual accuracy, citation accuracy, completeness, source quality, and tool efficiency.
- They emphasize that agent evaluation cannot rely on checking for a single “correct path,” because agents can take different valid sequences of steps.

### Key trade-offs
- **Performance vs. cost**: multi-agent systems consume substantially more tokens; Anthropic reports multi-agent usage can be ~15× chat token usage, and agents generally ~4× chat interactions. They frame multi-agent as economically viable only when task value is high enough.
- **Reliability complexity**: non-determinism + long-running state makes debugging and deployment harder, requiring strong observability and checkpointing.

## Results and metrics
- **Quality / capability:** Anthropic reports that a multi-agent setup with **Claude Opus 4 (lead) + Claude Sonnet 4 (subagents)** outperformed single-agent Claude Opus 4 by **90.2%** on an internal research eval.
- **What drives performance (their analysis):** token usage alone explained **80%** of performance variance on their BrowseComp evaluation; number of tool calls and model choice were the other major factors (together explaining 95%).
- **Latency:** introducing parallelism cut research time by **up to 90%** on complex queries.
- **Cost:** they report multi-agent systems use about **15×** more tokens than chats (and agents about **4×**).

## Comparison to AgentHub (8 dimensions)
Baseline: AgentHub (RSS/Reddit/HN ingestion + Cosmos hybrid search + scheduled agents + critic eval + self-improvement + safety gates + cost/deploy).

| Dimension | Them vs Us | Evidence / notes (from source) |
|---|---|---|
| 1. Data Pipeline | **DIFFERENT** | Anthropic’s write-up focuses on interactive research across web/Workspace/tools; it does not describe a daily RSS/news ingestion pipeline. |
| 2. Retrieval | **THEM AHEAD** | They emphasize multi-step search (not static retrieval) with subagents that iteratively use search tools; exact vector/BM25 stack not disclosed. |
| 3. Agents | **THEM AHEAD** | Production orchestrator-worker multi-agent pattern with parallel subagents and explicit coordination guardrails. AgentHub has multi-agents, but fewer details on parallel subagents + citation agent pipeline. |
| 4. Evaluation | **THEM AHEAD** | Explicit LLM-judge rubric + internal evals reported (90.2% improvement); citation accuracy is a first-class eval dimension. |
| 5. Self-Improvement | **DIFFERENT** | They mention models can help rewrite prompts/tool descriptions (e.g., tool-testing agent reducing completion time by 40%), but not a full autonomous “evolution engine” like AgentHub’s reflection→proposal→auto-apply loop. |
| 6. Safety | **NOT DISCLOSED** | Article discusses guardrails and privacy-preserving observability at a high level, but not budget gates, governance, or specific safety controls. |
| 7. Cost | **THEM AHEAD (for measurement), DIFFERENT (for target)** | They provide concrete token multipliers (4×/15×) and discuss economic viability; no $ figures disclosed. AgentHub provides explicit $ budget numbers. |
| 8. Deployment | **THEM AHEAD** | They discuss production reliability requirements (checkpointing/resume, tracing, deployment coordination) at a system level; implementation details not fully disclosed but production maturity is clear. |

## Confidence
**High** that the architecture and metrics above are accurately represented, because they come directly from Anthropic’s engineering write-up (primary source).
