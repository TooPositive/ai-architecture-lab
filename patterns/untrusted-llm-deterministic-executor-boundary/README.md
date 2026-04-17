# Untrusted LLM + Deterministic Executor Boundary Pattern

## Source
- **RUX — AI orchestration engine design with an untrusted LLM and a deterministic Executor trust boundary (Reddit post)** (summarized in internal knowledge base)
  - (No public URL captured in the knowledge node; internal reference id: `search-knowledge-0571dd16-0cb4-49c1-982f-9ac8e1ef8d85`)

## Problem
Tool-using agents frequently fail in ways that are hard to detect:
- The LLM hallucinates tool/action names.
- Parameters are malformed or unsafe.
- Orchestration silently “sort of works” with wrong actions, creating unreliable automation.

A common root cause is treating the LLM output as executable without a **validation / trust boundary**.

## Solution
Treat the LLM as **untrusted** and introduce an explicit **Executor** component that forms the trust boundary between probabilistic reasoning and deterministic systems.

Concrete architecture (as described):
1. **Planner (LLM)** produces a *proposed* plan/tool invocation in a structured format.
2. **Executor (deterministic)** validates and normalizes the plan against a strict contract (schema + allowlist).
3. **Tool** executes only validated actions.
4. **Service** is the real side-effecting system (DB, filesystem, CI, cloud APIs).

Key implementation details:
- Executor enforces a schema/contract: required fields, types, allowed tool names, allowed parameter ranges.
- Executor rejects or requests repair when the plan is not compliant (fail-closed).
- Observability: record validation failures as reliability metrics for the agent.

## Trade-offs
- **Pros**: Dramatically improves safety and reliability; prevents silent hallucinated actions; enables measurable reliability KPIs (reject rate, repair rate, tool-call validity).
- **Cons**: More engineering work; some tasks require iterative “repair” loops; strict schemas can reduce agent flexibility; needs ongoing maintenance as tools evolve.

## When to use / When NOT to use
**Use when**:
- Tools can cause side effects (writes, purchases, deployments).
- You need auditable automation with predictable behavior.
- You want to measure and improve agent reliability over time.

**Don’t use when**:
- You’re prototyping low-stakes workflows and want maximum speed of iteration.
- The agent is purely conversational with no external actions.

## Implementation notes
- Languages: any. Executors are often easiest in strongly typed languages (TypeScript/Zod, C#/System.Text.Json + validators, Python/Pydantic).
- Libraries:
  - JSON Schema validation (AJV, Newtonsoft.Json.Schema, python-jsonschema)
  - Typed tool contracts (Pydantic, Zod)
- Effort: ~1–3 days for a basic executor + allowlist; longer if you add policy checks and repair loops.
