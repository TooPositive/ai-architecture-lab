# AI Architecture Lab

Continuously updated research, patterns, benchmarks, and architectural decisions for enterprise AI systems.

This repository is maintained by a combination of autonomous AI agents and manual research. Content is generated from analysis of 9,300+ collected articles, hands-on implementation experience building production AI systems, and continuous benchmarking against real workloads.

## What's Inside

### [Patterns](patterns/)
Implemented architecture patterns with diagrams, trade-offs, and benchmarks. Each pattern includes working code and honest assessment of when to use it (and when not to).

### [Benchmarks](benchmarks/)
Continuous benchmark results comparing RAG techniques, embedding models, retrieval strategies, and more. Updated weekly with real data from a production RAG pipeline.

### [Weekly Briefs](weekly/)
Weekly synthesis of AI architecture trends. Not a link dump — analysis of what's changing, why it matters, and what architects should pay attention to.

### [Tech Radar](radar/)
ThoughtWorks-style technology radar for AI architecture, updated monthly. Technologies classified as Adopt, Trial, Assess, or Hold based on observed adoption and hands-on experience.

### [Case Studies](case-studies/)
Comparative analysis of AI architectures — both our own systems and published architectures from other organizations. Honest comparison across 8 dimensions.

### [Decisions](decisions/)
Architecture Decision Records (ADRs) documenting the reasoning behind every significant technical choice. Auto-generated with each content update.

### [Compliance](compliance/)
Regulatory compliance guides for AI systems, starting with the EU AI Act (enforcement August 2026). Technical mappings from legal requirements to implementation patterns.

## Tech Stack Context

The patterns and benchmarks in this repo come from two real systems:

- **AgentHub** — 132K LOC C# autonomous AI system (Azure Functions + daemon VM, Cosmos DB, Semantic Kernel). Runs 24/7: collects articles, drafts content, proposes and implements its own improvements.
- **LogiCore** — Python RAG reference architecture with 12-phase enterprise design. Benchmarked against real workloads with hybrid search, reranking, and evaluation pipelines.

Patterns are language-agnostic where possible. Implementation notes reference both .NET/Semantic Kernel and Python/LangChain equivalents.

## How This Repo Updates

Most content is generated autonomously:
- **Daily**: Article collection and analysis (9,300+ articles from 35 sources)
- **Weekly**: Architecture briefs, benchmark runs, pattern detection
- **Monthly**: Tech radar refresh
- **On-demand**: Case studies, compliance guides

See [CONTRIBUTING.md](CONTRIBUTING.md) for details on the content pipeline.

## License

[MIT](LICENSE)
