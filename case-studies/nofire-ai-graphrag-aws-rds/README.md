# NOFire AI — Production-ready GraphRAG for Root Cause Analysis (AWS RDS/PostgreSQL)

Source: https://www.nofire.ai/blog/How-We-Built-a-Production-Ready-GraphRAG-for-AI-Root-Cause-Analysis-Using-PostgreSQL

## Company / project
NOFire AI — GraphRAG-based root cause analysis (RCA) + change-management intelligence platform.

## Problem they solved
Traditional observability tools struggle to explain incident causality in large AWS environments (services/dependencies/changes/alerts). NOFire AI frames incident triage as a **graph traversal** problem, aiming to reduce false leads, speed incident response, and improve first-touch accuracy.

## Architecture choices and trade-offs (as described)

### Data pipeline
- Continuously ingests operational events and relationships to keep a “continuously evolving knowledge graph” current.
- Graph models entities such as **services, Kubernetes workloads, databases, changes, alerts, investigations, incidents, postmortems**.
- Stored as tables/edges/metadata in PostgreSQL (AWS RDS/Aurora).

Trade-offs:
- Choosing a relational substrate avoids bespoke graph infra but requires careful schema design and recursive query performance tuning.

### Retrieval
- **Hybrid retrieval:**
  - **Vector similarity** (pgvector) to retrieve relevant nodes/entities (example SQL shown in article).
  - **Graph traversal** using **recursive SQL (WITH RECURSIVE)** for multi-hop dependency paths (“local search” 1-hop neighbors + “global patterns” multi-hop).
- Mentions production tuning: “vector indexing, path caching, join filtering.”

Trade-offs:
- PostgreSQL provides SQL-native access and predictable ops, but recursive graph queries at scale require tuning/caching to hit latency/throughput goals.

### Agents
- Not described as an explicit multi-agent runtime. The article describes GraphRAG as the retrieval interface used to ground GenAI outputs for incident workflows.

### Evaluation
- No formal eval harness described. The article reports operational metrics (see below).

### Self-improvement
- Mentions a feedback loop where “Causal AI identifies likely causes… Generative AI turns those insights into actionable output,” but does not describe an automated self-improvement engine (reflection/proposals/auto-apply).

### Safety / reliability controls
- Positioning is “production-first” and AWS-native; however, explicit safety controls (budget gates, prompt sanitization, governance) are **not disclosed**.

### Cost
- Rationale emphasizes “predictable cost at telemetry scale” via Postgres on AWS RDS vs graph DB alternatives, but actual spend is **not disclosed**.

### Deployment
- Deployed on AWS (RDS/Aurora). Specific deployment automation patterns (CI/CD, rollback, canaries) are **not disclosed**.

## Results and metrics
From the article’s “Key Results” section (reported outcomes):
- **Reduced false positives by over 65%**
- **Cut mean incident response time by 40–60%** (high-priority alerts)
- **Increased first-touch resolution accuracy by 3×**
- **9000+ resources in production**
- “Accelerated change intelligence workflows… up to **80% faster**”

(These are claimed in the blog; underlying measurement method not disclosed.)

## Comparison vs AgentHub (8 dimensions)

| Dimension | THEM vs US | Evidence / notes |
|---|---|---|
| 1. Data Pipeline | DIFFERENT | Their pipeline is infra-event/observability KG ingestion; ours is daily news/RSS/Reddit/HN ingestion. Both are pipelines, but for different data + cadence specifics for them aren’t fully quantified. |
| 2. Retrieval | THEM AHEAD | They do **GraphRAG**: pgvector + multi-hop graph traversal in Postgres (recursive SQL), plus caching/tuning; AgentHub does Cosmos hybrid search (vector+BM25) over docs, not graph traversal. |
| 3. Agents | US AHEAD | They do not describe a production multi-agent system; AgentHub has 15 scheduled + 8 command agents with goal loops. |
| 4. Evaluation | NOT DISCLOSED | They provide operational outcomes, but no eval framework details; AgentHub has goal critic + quality scoring. |
| 5. Self-Improvement | NOT DISCLOSED | Mentions causal+genAI feedback loop conceptually, but no automated reflection→proposal→auto-apply described. |
| 6. Safety | NOT DISCLOSED | No details on budget gates, prompt sanitization, or governance. |
| 7. Cost | NOT DISCLOSED | Claims “predictable cost” via Postgres/AWS RDS, but no numbers; AgentHub has $30–50/mo infra + $5/day LLM budget. |
| 8. Deployment | NOT DISCLOSED | AWS-native with RDS/Aurora; CI/CD/rollback not detailed; AgentHub has git push auto-deploy + health checks + rollback. |

## Confidence
High that the above reflects the article’s described architecture and metrics (direct quotes/claims). Medium confidence on any implied details not explicitly stated (marked not disclosed).
