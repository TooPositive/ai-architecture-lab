Source: https://www.anthropic.com/engineering/multi-agent-research-system

# Anthropic — Multi-agent Research system (Claude Research)

## Company / project
- **Company:** Anthropic
- **Project:** Claude Research (multi-agent research feature)
- **Article:** *How we built our multi-agent research system* (Jun 13, 2025)
- **URL:** https://www.anthropic.com/engineering/multi-agent-research-system

## Problem they solved
Build a research capability that can handle open-ended, path-dependent investigations where the steps can’t be fully predicted in advance, and where useful information exceeds a single model context window.

## Architecture choices and trade-offs (from the article)
- **Orchestrator–worker multi-agent pattern:** a lead agent (LeadResearcher) plans and coordinates, then spawns multiple subagents to search in parallel and return compressed findings.
  - **Trade-off:** parallelism improves breadth/throughput but **burns tokens quickly**; not economical for low-value tasks.
- **Parallel subagents with separate contexts for “compression”:** subagents explore different directions with their own context windows, then summarize back to the lead.
  - **Trade-off:** reduces path dependency and increases coverage, but increases coordination complexity (duplication, gaps, runaway exploration).
- **Iterative research loop + tool use:** lead agent synthesizes results and decides whether to spawn more agents or stop.
  - **Trade-off:** agents can overshoot (search too long, spawn too many agents) without strong prompting/constraints.
- **Memory usage to persist plan across long contexts:** lead saves plan to “Memory” to keep critical intent when long interactions risk truncation (article mentions truncation beyond ~200k tokens).
- **Dedicated CitationAgent step:** after research, a citation agent identifies locations in documents for citations so final claims are attributable.
  - **Trade-off:** extra step/cost, but improves trustworthiness and auditability.
- **Prompt engineering as primary control surface:** Anthropic emphasizes prompt rules for delegation clarity, scaling effort, and avoiding failure modes (e.g., spawning 50 subagents for simple queries).

## Results and metrics
- **Internal eval:** Multi-agent system (lead Claude Opus 4 + Claude Sonnet 4 subagents) reportedly **outperformed single-agent Claude Opus 4 by 90.2%** on Anthropic’s internal research evaluation (breadth-first style queries).
- **Token economics:** agents typically use **~4×** more tokens than chat interactions; **multi-agent systems use ~15×** more tokens than chats.
- **Variance drivers (BrowseComp analysis):** token usage explains **80%** of performance variance; tool calls and model choice are also important; three factors explain **95%** of variance.

## Comparison vs AgentHub (8 dimensions)
Baseline reference: AgentHub as provided in the prompt.

1) **Data Pipeline** — **NOT DISCLOSED**
   - Article is about interactive research; no daily RSS/Reddit/HN ingestion pipeline described.

2) **Retrieval** — **DIFFERENT**
   - Anthropic emphasizes **multi-step web/tool search** rather than static RAG chunk retrieval; no mention of vector/BM25 store like Cosmos.

3) **Agents** — **THEM AHEAD**
   - Production multi-agent orchestration with parallel subagents + explicit orchestrator-worker pattern and specialized roles (LeadResearcher, Subagents, CitationAgent).

4) **Evaluation** — **THEM AHEAD**
   - They report internal research evals and analysis (e.g., 90.2% improvement; variance attribution). AgentHub’s eval is described, but no comparable quantitative uplift is stated.

5) **Self-Improvement** — **NOT DISCLOSED**
   - Mentions iterative prompt improvement and simulations, but no autonomous evolution engine described.

6) **Safety** — **NOT DISCLOSED**
   - Article excerpt does not provide budget gates/rate limiting/governance specifics.

7) **Cost** — **THEM AHEAD**
   - They quantify token multipliers (4× and 15×) and discuss economic viability constraints; AgentHub has explicit $/month + budget gates, but not token-economics measurements for multi-agent usage.

8) **Deployment** — **NOT DISCLOSED**
   - No CI/CD or rollback details for their production feature in the article excerpt.
