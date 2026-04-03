# Google Cloud + App Orchid — Multi-agent system for business forecasting

Source: https://cloud.google.com/blog/products/ai-machine-learning/how-we-built-a-multi-agent-system-for-superior-business-forecasting

## Company / project
Google Cloud + App Orchid — a multi-agent application for operational/business forecasting, combining Google’s prediction agent with App Orchid’s Data Agent under an orchestrator agent.

## Problem they solved
Enterprises struggle with forecasting due to siloed data, missing semantics, and labor-intensive data preparation. The system aims to improve forecasting accuracy and operational decision-making by pairing an enterprise data understanding agent with a specialized prediction agent.

## Architecture choices and trade-offs (as described)

### Data pipeline
- App Orchid **Data Agent** builds a “smart data layer” by unifying enterprise data: uses **Google Cloud Cortex Framework** + App Orchid’s semantic-layer tech to create a unified “knowledge foundation” / mapped landscape (described as a knowledge graph).

Trade-offs:
- Stronger semantic grounding and unified view, but depends on enterprise data integration work and governance constraints (details not provided).

### Retrieval
- Data Agent acts as an “intelligent query engine” over enterprise data/semantics and produces **high-context time-series datasets** for forecasting.
- Mentions MCP tooling for databases via “MCP Toolbox for Databases” (e.g., BigQuery) as part of Google ADK ecosystem.

Trade-offs:
- This is less “document RAG” and more semantic/time-series dataset construction; retrieval details (hybrid search, ranking, embeddings) are not specified.

### Agents
Described as a **3-agent** setup:
1. **Business forecasting agent** (user-facing) = orchestrator agent.
2. **App Orchid Data Agent** = enterprise data semantics + dataset builder.
3. **Google prediction agent** = forecasting models (TimesFM + Population Dynamics Foundation Model).

Orchestration and interop:
- Uses **A2A Protocol** for agent-to-agent communication.
- Built with **Google Agent Development Kit (ADK)**; deployable to managed runtime **Vertex AI Agent Engine** (also notes Cloud Run/GKE/etc. as options).
- Uses **Gemini models** for reasoning + tool use; notes that many offer **1M token** context windows (general capability mention in article, not specific to this deployment’s exact window).

Trade-offs:
- Multi-agent modularity supports specialization and potential vendor-agnostic agent collaboration via A2A, but introduces integration/coordination complexity and governance requirements.

### Evaluation
- No explicit evaluation framework described (offline/online eval, test sets, guardrails).

### Self-improvement
- Not described.

### Safety / governance
- Positioned as running through **Gemini Enterprise** with “enterprise-grade security, data privacy, and governance,” and deployed on **Vertex AI Agent Engine**; specifics (rate limits, policy enforcement, audit logging details) not described.

### Cost
- Not disclosed.

### Deployment
- Orchestrator agent deployed on **Vertex AI Agent Engine** and “registered” in **Gemini Enterprise**. Mentions ADK supports deployment to Cloud Run/GKE/etc. (capability statement).

## Results and metrics
- Qualitative benefits claimed (improved accuracy, efficiency, faster insights, reduced costs), but **no concrete metrics disclosed** in the article.

## Comparison vs AgentHub (8 dimensions)

| Dimension | THEM vs US | Evidence / notes |
|---|---|---|
| 1. Data Pipeline | THEM AHEAD | They describe an enterprise “smart data layer” (Cortex Framework + semantic layer + mapped knowledge foundation) to produce forecasting-ready datasets; AgentHub’s pipeline is news ingestion. Different domain, but theirs implies deeper enterprise data integration. |
| 2. Retrieval | DIFFERENT | Their “retrieval” is semantic dataset construction + database tooling; AgentHub is doc retrieval (Cosmos hybrid vector+BM25). Article doesn’t specify embeddings/ranking for them. |
| 3. Agents | DIFFERENT | They have a 3-agent orchestrator + 2 specialist agents with A2A protocol; AgentHub has many scheduled/command agents with autonomous goal loops. |
| 4. Evaluation | NOT DISCLOSED | No eval design described; AgentHub has goal critic + completion review. |
| 5. Self-Improvement | US AHEAD | No self-improvement loop described; AgentHub has evolution engine (reflection→proposals→auto-apply). |
| 6. Safety | NOT DISCLOSED | Mentions enterprise security/governance via Gemini Enterprise, but no concrete mechanisms/limits given. |
| 7. Cost | NOT DISCLOSED | No numbers. |
| 8. Deployment | THEM AHEAD | They deploy to **Vertex AI Agent Engine** and integrate with **Gemini Enterprise** (managed agent platform). AgentHub uses Azure Functions + Cosmos with git-push deploy; neither is strictly better, but a managed agent runtime is a notable difference and may provide stronger enterprise ops. |

## Confidence
Medium-high: the agent decomposition, A2A/ADK/Vertex AI Agent Engine details are explicitly described. Low-medium on data pipeline specifics beyond what’s stated (no concrete schemas/SLAs/latencies).
