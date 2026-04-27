# Strict-DAG Agent Orchestration (Deterministic Workflow Graph)

## Source
- **Synapse AI — Multi-Agent Orchestration Platform (README)**
- https://github.com/naveenraj-17/synapse-ai

## Problem
Typical agent frameworks run an open-ended loop (plan → act → observe → repeat). This makes production behavior hard to predict, test, and secure (e.g., unexpected tool calls, non-deterministic branching, and unclear termination).

## Solution
Model the agentic system as a **strict directed acyclic graph (DAG)** where:
- **Nodes** are steps (agents, tool calls, classifiers, validators, transformers).
- **Edges** are explicit allowed transitions.
- **Execution** follows an exact pre-defined path chosen by graph conditions; no unbounded ReAct loop.

Concrete implementation details grounded in the source repo:
- Define orchestrations as **strict DAGs** (“Execution follows the exact path you designed”).
- Allow **different models per node**: cheap/fast LLMs for routing/classification steps; larger LLMs for “hard” steps.
- Tools are attached via **MCP servers**, webhooks, or Python scripts (repo README explicitly lists MCP + scripts/webhooks).
- Run the orchestration via a server/CLI that executes nodes deterministically (repo provides `synapse start` etc.).

## Trade-offs
- **Pros**
  - Predictable control flow: easier debugging, reproducibility, and incident response.
  - Easier permissioning: tool access can be scoped per node/edge.
  - Benchmarkable: each node has measurable latency/cost and success rates.
- **Cons**
  - Less flexible for truly open-ended tasks; the DAG must anticipate branches.
  - More upfront design effort than a simple agent loop.
  - Graph changes require versioning/migrations (workflow-as-code).

## When to use / When NOT to use
- **Use when**
  - You need reliable automation with clear termination (ops workflows, back-office automation, code-mod pipelines).
  - Security/compliance requires explicit approval boundaries for tool use.
  - You want deterministic replay for debugging.
- **Do NOT use when**
  - Tasks are exploratory and you cannot predict the steps ahead of time.
  - The primary goal is ideation rather than execution reliability.

## Implementation notes
- **Languages/Libraries**: The referenced implementation is an open-source platform repo (Python + Node are listed as prerequisites).
- **Tools integration**: MCP servers via `uv`/`uvx` (repo mentions `uv` for running MCP servers), plus webhooks and Python scripts.
- **Effort**:
  - Proof-of-concept DAG executor: ~1–3 days.
  - Production orchestration UI + versioning + observability: weeks.

