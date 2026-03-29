# AI Architecture Tech Radar -- April 2026

Assessment of technologies for building production AI systems, based on hands-on implementation in a 125K-line autonomous agent platform and analysis of 9,300+ collected articles.

---

## Adopt

Technologies proven in production. Recommended for teams building AI systems today.

### Techniques

**Hybrid RAG (Vector + Keyword Search)**
Pure vector search misses exact matches; pure keyword search misses semantic similarity. Combining both with reciprocal rank fusion consistently outperforms either alone. Our production system switched from keyword-only `Contains()` to `HybridSearchAsync()` and saw immediate retrieval quality improvements. The pattern is straightforward to implement and the tradeoff (slightly higher latency from two queries) is worth it.

**Structured Output with JSON Schema Enforcement**
Asking an LLM to "return JSON" and hoping for the best is unreliable. Schema-enforced structured output (via `response_format` or tool calling) eliminates parsing failures. Every agent-to-agent communication in our system uses typed C# models deserialized from structured output. The alternative -- regex parsing of free-text -- is fragile and a maintenance burden.

**Stateless Iteration Workers**
Run one iteration, persist all state to a durable store, return. No in-memory state survives restarts. This pattern makes autonomous agents crash-safe and rate-limit-resilient. Our goal loop polls Cosmos DB every 10 seconds, picks a queued goal, runs one iteration, and re-queues. The system has survived dozens of VM restarts without losing goal progress.

**Tiered Memory with TTL**
Not all agent memory should persist forever. Three tiers: permanent (core knowledge), 90-day (execution patterns), 30-day (transient observations). Cosmos DB TTL handles cleanup automatically. Without TTL, the knowledge base grows unbounded and retrieval quality degrades as noise accumulates.

**Prompt Caching (Static Prefix Ordering)**
Put large static content at the top of system prompts, dynamic content at the bottom. Azure OpenAI caches matching prefixes at 50% token cost discount. Reordering our prompts to follow this pattern reduced LLM costs measurably. The key rule: never put dynamic content before static -- it breaks the cache prefix.

### Platforms

**Cosmos DB for Agent State**
Document model fits agent state naturally (goals, traces, memories are all JSON documents). Change feed enables reactive patterns. TTL handles cleanup. Partition keys align with access patterns (goal ID, agent name, date). The main gotcha: cross-partition queries are expensive, so partition key design matters upfront.

**Azure Functions for Scheduled Pipelines**
Timer triggers for daily collection, weekly digests, and periodic maintenance. Consumption plan keeps costs near zero when idle. Not suitable for long-running autonomous loops (use a VM daemon for that), but excellent for batch pipelines that run on a schedule.

### Tools

**Semantic Kernel (Core Abstractions)**
Kernel + plugins + function calling provides a solid abstraction over LLM providers. The plugin model (native functions with `[KernelFunction]` attributes) maps well to tool calling. Auto function calling handles the invoke loop. Limitations: SK's built-in agents are basic; for complex orchestration we built our own goal loop on top of SK's chat completion.

### Languages & Frameworks

**.NET for AI Agents**
Less common than Python but production-viable. Strong typing catches bugs at compile time that Python finds at runtime. `Microsoft.Extensions.AI` provides a clean abstraction layer. Performance is excellent for the orchestration layer (the LLM call dominates latency, not the host language). The ecosystem gap vs. Python is real but narrowing.

---

## Trial

Worth investing in. Showing value in real projects, needs more mileage before a broad recommendation.

### Techniques

**Evolution Engine (Execution Trace Reflection)**
After each goal completes, a reflection agent analyzes the execution trace and proposes improvements (prompt changes, new skills, anti-patterns to avoid). This creates a feedback loop where the system gets better over time. In trial because: the quality of proposals varies widely, and without a necessity gate many proposals are low-value churn.

**DAG-Based Task Planning**
Decompose goals into a directed acyclic graph of tasks with dependency edges. Execute independent tasks in parallel (up to 3 concurrent). Each task runs in a git worktree for isolation. Works well for code generation goals. In trial because: the decomposition quality depends heavily on the LLM's understanding of the codebase, and re-planning after failures is not yet reliable.

**Confidence-Based Human Escalation**
When an agent's confidence score drops below a threshold (0.4 in our system), it pauses and sends a Discord alert instead of guessing. Reduces wasted computation on uncertain tasks. In trial because: confidence calibration is model-dependent and the threshold needs per-agent tuning.

**Cost Budget Governance**
Hard caps on per-goal cost ($5-$20 depending on type), daily cost ($50), and weekly cost ($300). Goals that exceed their budget are stopped. Philosophy: budgets are safety nets, not execution strategy -- killing a productive goal wastes all money already spent. In trial because: optimal budget levels are still being calibrated.

### Platforms

**MCP (Model Context Protocol) for Tool Integration**
Standardized protocol for connecting LLMs to external tools and data sources. Promising for reducing custom integration code. In trial because: the ecosystem is early, some servers are unreliable, and the serialization behavior has quirks (double-serialization bugs with some providers).

### Tools

**Microsoft.Extensions.AI (MAF)**
Unified abstraction over chat clients, embeddings, and middleware. The middleware pipeline (logging, rate limiting, telemetry) is clean. In trial because: the library is newer than Semantic Kernel and the migration path from SK-native to MAF-native is still evolving.

**Git Worktree Isolation for Agent Coding**
Each code task gets its own worktree so parallel tasks do not conflict. Merge happens only after build + test pass. In trial because: merge conflicts between concurrent worktrees are possible and the conflict resolution is manual.

### Languages & Frameworks

**Semantic Kernel OpenAPI Plugin Import**
Import any `swagger.json` as SK kernel functions. Useful for connecting agents to existing REST APIs without writing custom plugins. Filter with `OperationSelectionPredicate` to avoid importing hundreds of irrelevant operations. In trial because: large APIs generate too many functions for the model's context, and the filtering requires manual curation.

---

## Assess

Explore with intent. Promising, not yet proven enough for production workloads.

### Techniques

**Self-Modifying AI Systems**
An AI system that can modify its own source code, build, test, merge, and deploy. We achieved this once successfully (see [case study](../case-studies/self-modifying-ai-system/)). In assess because: one success does not prove safety at scale. Open questions around necessity gating, long-term architectural drift, and evaluator independence remain unresolved.

**GraphRAG (Knowledge Graph + Retrieval)**
Building a knowledge graph from collected articles and using graph traversal for retrieval. Theoretically superior to flat vector search for multi-hop reasoning questions. In assess because: the graph construction cost is high, maintenance is non-trivial, and for most single-hop retrieval tasks hybrid RAG is sufficient.

**Multi-Agent Orchestration (Inter-Agent Delegation)**
Agents that can delegate tasks to other agents via a message bus. The infrastructure exists (agent discovery, message bus, routing) but agents do not yet actively use it. In assess because: the coordination overhead and failure modes of multi-agent systems are poorly understood in practice.

**Agentic Coding (LLM Writes Production Code)**
LLMs writing code that ships to production, not just prototypes. Our `CodeTaskWorker` does this daily. In assess (not trial) because: the read:write ratio problem (models reading excessively before writing) and the tendency to over-engineer require significant prompt engineering and guardrails to manage.

### Platforms

**Local LLM Hosting for Agent Workloads**
Running open-weight models locally for cost reduction on high-volume, low-complexity tasks. In assess because: the quality gap vs. frontier models is significant for code generation and complex reasoning, and the operational burden of GPU management is non-trivial for a solo developer.

### Tools

**OpenTelemetry for AI Agent Observability**
Structured metrics and traces for agent execution. We emit counters (iterations, stops, dropped events) and histograms (iterations to completion). In assess because: the AI-specific semantic conventions are still evolving and most observability tools do not have good visualizations for agent execution traces.

---

## Hold

Proceed with caution. Either superseded or the risks outweigh the benefits at this stage.

### Techniques

**Pure Vector Search Without Keyword Fallback**
Vector-only retrieval misses exact term matches and produces false similarities for out-of-domain content. Always pair with keyword search (hybrid RAG). We learned this the hard way: `IKnowledgeService.SearchAsync()` (keyword only) was replaced by `ISemanticSearchRepository.HybridSearchAsync()` (hybrid) -- going the other direction (vector only) would be equally wrong.

**Autonomous Deployment Without Safety Gates**
Letting an AI system deploy code without blast radius checks, security scans, or human escalation for large changes. Even with all our safety gates, autonomous deployment is the highest-risk component. Without gates, a single bad deployment could take down the system with no human aware until it is too late.

**Unstructured LLM Output for Agent Communication**
Passing free-text between agents and parsing with regex. Structured output with schema enforcement is strictly superior. The parsing failures, edge cases, and maintenance cost of regex-based extraction are not worth the "flexibility" of free text.

**Synchronous Multi-Turn Agent Sessions Without Compression**
Long multi-turn conversations without context compression. Token costs grow quadratically with conversation length. Our system uses `ContextCompressor` to summarize history when it exceeds a threshold (3,000 tokens for dependency context). Without compression, a 20-turn session can cost 10x what a compressed one does.

### Platforms

**Bing Search API**
Retired August 2025. Replaced by Brave Search API in our system, though the free tier of Brave has also been discontinued. For web search in AI agents, consider Azure AI Search or direct scraping with Playwright as alternatives.

### Tools

**LangChain for Production Systems**
High abstraction overhead, difficult to debug, and the Python-centric ecosystem does not translate to .NET. For .NET AI systems, Semantic Kernel and Microsoft.Extensions.AI provide better integration with the platform. For Python systems, LangChain may have value for prototyping but the abstraction layers become liabilities in production.

---

## Change Log

| Date | Change |
|---|---|
| 2026-04 | Initial radar publication |
