Source: https://huggingface.co/blog/rocketmoney-case-study

# Rocket Money — Scaling volatile ML models in production (transaction classification)

## Company / project
- **Company:** Rocket Money (formerly Truebill)
- **Project:** Transaction string → merchant/service classification powering transaction enrichment and subscription detection
- **Article:** *Rocket Money x Hugging Face: Scaling Volatile ML Models in Production* (Sep 19, 2023)
- **URL:** https://huggingface.co/blog/rocketmoney-case-study

## Problem they solved
Rocket Money needed to transform messy, truncated payment transaction strings into reliable merchant/service classes at high scale. Their legacy approach (regex normalizers + a growing decision table) became hard to maintain as the number of classes and overlaps increased.

They also needed a production serving solution that could handle **bursty**, high-volume load (their prior regex system processed **100M+ transactions/month**) with high availability and low latency, without having a dedicated MLOps team.

## Architecture choices and trade-offs (from the article)
- **Modeling shift:** from regex/decision tables → ML, eventually choosing **BERT-family** text classification for **4000+ classes** (offline evaluation first).
  - **Trade-off:** better generalization/maintainability vs added serving/monitoring complexity and outage risk when changing models/classes.
- **Labeling + data ops tooling:** used **Retool** to build labeling queues, gold-standard validation datasets, and drift detection monitoring tools.
  - **Trade-off:** faster internal tooling vs building bespoke labeling infrastructure.
- **Offline evaluation in a GCP warehouse:** bulk of initial testing/evaluation was offline in their **GCP warehouse**, building telemetry to measure performance at large class counts.
- **Managed model serving (buy vs build):** evaluated an in-house prototype hosting setup vs **AWS SageMaker** vs **Hugging Face Inference API**.
  - **Decision drivers:** they already used **GCP for data storage** and **Vertex Pipelines for training**, so exporting to AWS was “clunky and bug prone”; Hugging Face setup was quick and handled traffic within a week; after ~3 months evaluation they selected Hugging Face for hosting.
  - **Trade-off:** less control vs reduced ops burden and faster time to reliable scaling.
- **Load testing and gradual traffic ramp:** simulated worst-case load and gradually increased production traffic to hosted models before full cutover.
- **Monitoring/telemetry for production and model data:** built new telemetry to monitor model + production data because errors early in the pipeline could impact business metrics.
- **A/B experimentation against baseline system:** split new users 50/50 between old regex system and new model; assessed model performance alongside business metrics (e.g., paid retention/engagement) before ramping.
- **Caching layer before inference calls:** to manage inference cost, they added caching that reduces transaction cardinality by reusing prior inference results.
  - **Trade-off:** cache invalidation/coverage vs significant cost reduction; theoretical **93%** cache rate, achieved **85%** in production.

## Results and metrics
- **Scale:** model serving scaled to a run rate of **>1 billion transactions/month** (after full rollout).
- **Caching:** theoretical **93%** cache rate; **~85%** achieved in production.
- **Rollout:** ramped to **100%** traffic over **~2 months** (new users first, then existing users).
- **Business impact:** ML model “clearly outperformed” on retention vs old system (exact numbers **not disclosed**).
- **Reliability incidents (not quantified):** one outage when expanding class count in a production model; one caching issue returning results from previous model; both resolved and not repeated (per article).

## Comparison vs AgentHub (8 dimensions)
Baseline reference: AgentHub as provided in the prompt.

1) **Data Pipeline** — **DIFFERENT**
   - Rocket Money focuses on **transaction ingestion + labeling + drift monitoring**, not news/article ingestion.

2) **Retrieval** — **NOT DISCLOSED**
   - This is a classification + serving system; no RAG retrieval stack described.

3) **Agents** — **NOT DISCLOSED**
   - No multi-agent LLM orchestration.

4) **Evaluation** — **THEM AHEAD**
   - They describe gold datasets, offline eval telemetry at 4000+ classes, and online A/B experiment tied to business metrics (retention/engagement).

5) **Self-Improvement** — **DIFFERENT**
   - Continuous improvement via adding labels daily + ongoing tuning; not autonomous reflection/proposal/apply loops.

6) **Safety** — **NOT DISCLOSED**
   - Mentions reliability concerns (uptime/latency), but no governance/budget/prompt-safety mechanisms.

7) **Cost** — **THEM AHEAD**
   - They describe inference cost pressure and implemented caching with measured production cache rate (**85%**) to manage budget.

8) **Deployment** — **DIFFERENT**
   - Uses managed hosting (Hugging Face Inference API) and gradual ramp + monitoring; AgentHub is git-push auto-deploy on Azure Functions/Cosmos with rollback/health checks.
