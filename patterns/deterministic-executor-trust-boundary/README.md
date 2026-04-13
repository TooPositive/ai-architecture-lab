# Deterministic Executor Trust Boundary (Untrusted LLM → Validated Actions)

## Source
- **I bulit an AI Orchestration engine without using LangChain - Here's what i learned** (Reddit r/LangChain)
- https://www.reddit.com/r/LangChain/comments/1saes4o/i_bulit_an_ai_orchestration_engine_without_using/

## Problem
In many agent systems, the LLM emits tool/action calls directly (often as free-form JSON). If the model:
- hallucinates an action name,
- provides invalid arguments,
- skips required steps,
- or silently fails,
then the system can fail unpredictably without validation or reliability measurement.

## Solution
Introduce an explicit **trust boundary** between probabilistic planning and deterministic execution:

**Planner (LLM, untrusted)** → **Executor (deterministic, schema-validated)** → **Tool** → **Service(s)**

Key implementation details described by the source:
- Treat *everything before the Executor as probabilistic* and *everything after as deterministic*.
- The **Executor schema is the contract** separating the two worlds.
- The Executor validates that the LLM output conforms to a strict action schema (action name, args, constraints).
- Only validated actions are allowed to trigger tool/service calls; invalid plans are rejected/returned for repair.

Practical design (recommended to make it operational):
- Define a small set of actions (e.g., `SearchDocs`, `GetCustomer`, `CreateTicket`, `RunQuery`) with typed parameters.
- Implement the Executor as a state machine that:
  1) parses model output,
  2) validates against JSON Schema / typed structs,
  3) enforces allow-lists + authZ policies,
  4) executes tool calls deterministically,
  5) records traces and error codes for reliability measurement.

## Trade-offs
- **More up-front design**: you must design schemas/contracts and keep them versioned.
- **Less flexibility**: ad-hoc tool usage is constrained by the action allow-list.
- **Extra loop on failures**: invalid plans require a repair prompt/iteration, adding latency.

## When to use / When NOT to use
**Use when:**
- Tool calls have real-world side effects (tickets, payments, deployments).
- You need measurable reliability (e.g., “% of runs with zero invalid actions”).
- You operate in regulated / security-sensitive environments where tool invocation must be controlled.

**Avoid when:**
- You are prototyping quickly and can tolerate occasional hallucinated tool calls.
- The agent only performs read-only tasks with minimal downside.

## Implementation notes
- **Schema validation**: JSON Schema (ajv), Pydantic (Python), Zod (TS), System.Text.Json + validators (.NET).
- **Execution engine**: deterministic workflow runner/state machine (Temporal, Durable Functions, LangGraph-style graph but with hard validation gates).
- **Effort**: ~3–7 engineering days for a small action set + validator + tracing; more for policy, auth, and audit logs.
