# Benchmark: Hierarchical Routing “Union-of-Experts” (UoE)

## Source
- **Union of Experts: Adapting Hierarchical Routing to Equivalently Decomposed Transformer** (arXiv:2503.02495, revised Sep 23 2025)
- https://arxiv.org/abs/2503.02495

## Classification
**BENCHMARKABLE**

## What metric improves?
- **Model quality per FLOP** (efficiency): lower perplexity at reduced FLOPs (language modeling).
- **Compute efficiency**: reduced FLOPs vs MoE baselines / efficient transformers with comparable or better accuracy.

## Technique summary
The paper proposes “Union-of-Experts (UoE)”:
- Decompose a non-MoE transformer into an equivalent set/group of experts.
- Apply **hierarchical routing** combining patch-wise data selection + expert selection.
- Extend MoE routing beyond FFNs to **attention blocks**.
- Provide a hardware-optimized parallelization scheme using batched matrix multiplications for expert compute.

## Theoretical comparison vs our baseline
**Our baseline** (given): RAG retrieval MRR@10=0.78, latency p50=45ms on 246 docs hybrid search.

UoE is a **model-architecture** efficiency technique, not directly a retrieval improvement. The plausible effect in an AI system would be:
- **Lower inference cost** or **lower latency** for the same (or better) model quality, which can allow:
  - more reranking steps
  - bigger context windows
  - more agent iterations
  - higher throughput
all without increasing cost.

A fair benchmark in our environment would measure:
- cost per generated token
- latency p50/p95 for generation
- quality on task suite (e.g., internal evals)

## Claimed improvements (from source abstract)
- Language modeling: **average perplexity reduction of 2.38** vs best-performing MoE method, using **76% of its FLOPs**.
- Long Range Arena: **≥0.68% higher average score** than comparison models, using **50% of the FLOPs** of the best MoE method.
- Image classification: **+1.75% average accuracy** over the best model at comparable FLOPs.

## Confidence
- **Medium**: claims are from the paper abstract; implementation and reproducibility would need full-paper review and code inspection.
