# Benchmark: Bandit-Feedback Model Routing (BaRP)

## Source
- **Paper:** Learning to Route LLMs from Bandit Feedback: One Policy, Many Trade-offs
- **URL:** https://arxiv.org/html/2510.07429

## Technique summary
BaRP frames model selection as a **preference-conditioned contextual bandit**: for each prompt, a router chooses among candidate LLMs, observes feedback only from the chosen model (bandit feedback), and updates the policy to optimize a weighted objective (quality vs cost). At inference time, an operator can dial a **preference vector** (quality-weight vs cost-weight) without retraining.

## What metric improves?
Primarily:
- **Cost-effectiveness** (quality achieved per unit cost)
- **Accuracy / task score** under a fixed budget
Secondarily (deployment-dependent):
- **Latency** if router selects smaller/faster models for easier prompts

## Claimed improvements (from the paper)
The paper reports that BaRP:
- Outperforms strong *offline* routers by **at least 12.46%** on in-distribution tasks.
- Outperforms the largest single LLM by **at least 2.45%** (and 3.81% in-distribution).
- Improves generalization to unseen tasks (reported larger margins vs offline routers).

(These numbers are as stated in the paper abstract/introduction; full benchmark setup is described in the paper.)

## Theoretical comparison vs our baseline (reasoned)
**Our baseline (given):** MRR@10=0.78, latency p50=45ms, hybrid search on 246 docs using Cosmos DB vector + BM25.

BaRP does not change retrieval directly; it changes **which LLM** answers the post-retrieval prompt. Expected impacts:

1) **Cost:**
- If your workload has many “easy” queries, BaRP should route them to cheaper models while reserving expensive models for “hard” queries. Net effect: lower average token spend while keeping quality near a strong-model baseline.

2) **Latency:**
- Latency may improve if cheaper models are also faster, but may worsen if the router triggers escalations (fallback to larger model) or if router overhead is significant. In most systems router overhead is small vs LLM latency.

3) **Answer quality (post-retrieval):**
- Quality can improve over “always use one model” by exploiting complementary strengths (e.g., one model better at tool-use/formatting, another better at reasoning).
- In a RAG system, routing can partially compensate for retrieval noise: for borderline retrieval cases, route to a stronger model that can better synthesize imperfect context.

4) **Retrieval metrics (MRR@10):**
- Not directly affected. However, you can include “retrieval confidence features” (e.g., top-1/top-5 similarity gaps, BM25 score stats) into the router; then the router’s policy may choose stronger models when retrieval looks weak, improving *end-to-end* QA success even if MRR@10 is unchanged.

## How to benchmark in our environment
To make this benchmarkable against our RAG baseline, define an end-to-end evaluation suite:
- Dataset: representative QA queries over the same 246-doc corpus.
- Features into router: query length, detected intent, retrieval score distribution, number of chunks, tool-use requirement, etc.
- Candidate models: local small model + mid + strong cloud model(s).
- Metrics:
  - End-to-end accuracy (task success / judge score)
  - Cost per query
  - Latency p50/p95
  - Pareto frontier (quality vs cost; quality vs latency)

## Notes / caveats
- Paper emphasizes **bandit-feedback realism**: in deployment you only see feedback from the chosen model, not from all alternatives. This matches production constraints better than offline supervised routers.
- Requires an online feedback signal (explicit ratings, implicit success metrics, or downstream task score).
