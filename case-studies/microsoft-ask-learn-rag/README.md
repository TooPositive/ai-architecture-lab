Source: https://devblogs.microsoft.com/engineering-at-microsoft/how-we-built-ask-learn-the-rag-based-knowledge-service/

# Microsoft — “Ask Learn” (RAG-based knowledge service) — architecture case study

## Company / project
- **Company:** Microsoft (Skilling org / Microsoft Learn / Microsoft Q&A)
- **Project:** **Ask Learn** (RAG-based knowledge service powering Q&A Assist and grounding for Copilot for Azure)
- **Primary write-up:** https://devblogs.microsoft.com/engineering-at-microsoft/how-we-built-ask-learn-the-rag-based-knowledge-service/

## Problem they solved
Microsoft needed to provide **reliable, verifiable, low-latency answers** to user questions in Microsoft Q&A (and later as a shared API for Copilot for Azure), grounded in Microsoft Learn documentation at very large scale. “Naïve RAG” was insufficient for Microsoft’s quality bar (accuracy, relevance, verifiability), and they also had to deal with continuous documentation updates and the inherent non-determinism of LLM outputs.

## Architecture choices and trade-offs
### High-level system architecture
- They implemented an **“Advanced RAG”** approach rather than naïve RAG, adding substantial logic and extra LLM calls around retrieval and generation.
- They designed a **service-oriented architecture** with:
  - a top-level **orchestration layer**
  - multiple **specialized services** for key responsibilities in the system

### Knowledge Service (indexing + vector store)
- A core internal service (“**Knowledge Service**”) is responsible for:
  - **chunking** Microsoft Learn technical docs
  - creating **embedding vectors** per chunk
  - storing vectors + chunks + metadata in a **vector database**
- Major data engineering challenge: keep embeddings up to date as **hundreds of technical writers** update **hundreds of thousands of documents**; the vector store must be continuously refreshed.
- Vector DB is accessed via a **web API** and consumed by the orchestration layer.

### Retrieval & inference pipeline (“Advanced RAG”)
Microsoft explicitly calls out additional stages beyond naïve retrieval:
- **Pre-retrieval**: query rewriting, query expansion, query clarification
- **Post-retrieval**: re-ranking, expanding chunks for more context, filtering low-quality/irrelevant/redundant chunks, compressing results
- **Post-generation**: additional checks to ensure outputs meet quality and safety/operational guidelines

### Evaluation and reliability (handling non-determinism)
- They highlight non-determinism as a major engineering problem and start by building a **“golden dataset”** (curated/annotated Q&A pairs + ground-truth sources) to benchmark changes in indexing, retrieval, prompts, etc.
- They describe migrating evaluation tooling from custom Python notebooks to **Prompt flow** evaluation flows over time.

### Operational / organizational constraints
- Privacy constraints: by default Microsoft employees can’t see customer prompts/outputs. They introduced an explicit customer-consent mechanism to share chat history when providing feedback, enabling root-cause analysis (up to ~30 minutes per response, per the post).

### Key trade-offs
- **Complexity vs. quality**: Advanced RAG requires many extra steps and LLM calls, increasing system complexity to meet reliability and verifiability needs.
- **Freshness vs. cost/ops**: continuously updating embeddings/index to track frequent doc changes increases pipeline complexity and compute cost.

## Results and metrics
- **Not disclosed** for quantitative retrieval metrics / latency / cost in the article excerpted via fetch.
- Qualitative result: after a few months, they launched **Microsoft Q&A Assist** (May 2023), then exposed the same underlying RAG system as a web API for Copilot for Azure (“Ask Learn”).

## Comparison to AgentHub (8 dimensions)
Baseline: AgentHub (RSS/Reddit/HN ingestion + Cosmos hybrid search + scheduled agents + critic eval + self-improvement + safety gates + cost/deploy).

| Dimension | Them vs Us | Evidence / notes (from source) |
|---|---|---|
| 1. Data Pipeline | **THEM AHEAD** | They run a continuous doc-ingestion + re-embedding pipeline for Microsoft Learn with frequent updates (“hundreds of thousands of documents”). AgentHub’s pipeline is news/article oriented, not doc publishing at this scale. |
| 2. Retrieval | **THEM AHEAD** | They implement “Advanced RAG” with pre/post retrieval processing, reranking, chunk expansion/filtering/compression. Specific DB tech not named in snippet, but the pattern is explicit. |
| 3. Agents | **DIFFERENT** | Architecture is orchestrated services + RAG pipeline; not an agentic multi-iteration goal loop like AgentHub. |
| 4. Evaluation | **THEM AHEAD** | Explicit golden dataset benchmark; evaluation for retrieval quality, groundedness, relevance, harms; later moved to Prompt flow evaluation flows. AgentHub has a goal critic, but fewer details about groundedness/harms eval harness. |
| 5. Self-Improvement | **NOT DISCLOSED** | No autonomous self-improvement loop described; improvements appear engineering-driven via evals/feedback. |
| 6. Safety | **DIFFERENT / PARTIALLY DISCLOSED** | Mentions safety/ethical/operational guidelines and a custom harms evaluation tool; privacy constraints are substantial and drive feedback collection design. Budget/rate-limit style controls not described. |
| 7. Cost | **NOT DISCLOSED** | No cost numbers in the article excerpt. |
| 8. Deployment | **DIFFERENT** | Service-oriented architecture + web API reuse across products; concrete CI/CD/rollback details not described. |

## Confidence
**Medium-High**. The write-up is a primary Microsoft engineering blog post; however, some implementation details (exact vector DB, embedding model, infra sizing, concrete metrics) are not disclosed in the portions fetched.
