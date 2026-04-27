# Executor Trust Boundary (Untrusted-LLM Orchestration)

## Source
- **RUX — AI orchestration engine design with an untrusted LLM and a deterministic Executor trust boundary (Reddit post)** (knowledge base entry: `search-knowledge-0571dd16-0cb4-49c1-982f-9ac8e1ef8d85`).
- URL: *not provided in the indexed source record* (the KB item is summarized from a Reddit post).

## Problem
Tool-using agents frequently fail silently when the LLM hallucinates tool/action names or produces malformed tool-call payloads. Without a hard trust boundary, the LLM effectively “presses buttons” directly, making reliability, validation, and auditability hard.

## Solution
Introduce an explicit **Executor** component as a **trust boundary** between probabilistic LLM planning and deterministic tool execution:

**Flow:** `Planner (LLM) → Executor (deterministic, schema-enforcing) → Tool → Service`

Concrete implementation details:
1. **Planner (LLM)** outputs a *plan proposal* (not executed directly), e.g. a list of steps with intent, target tool, and arguments.
2. **Executor** validates and normalizes the plan against:
   - an allowlist of tools/actions
   - a strict JSON Schema / typed contract per tool
   - policy checks (authZ, rate limits, PII rules, environment constraints)
3. Executor either:
   - rejects with structured errors (sent back to the LLM to repair), or
   - emits a canonical tool invocation (deterministic)
4. **Tool runner** executes the call, captures outputs, and returns results as structured data (not raw dumps unless required).

## Trade-offs
- **Pros**
  - Prevents “LLM presses buttons” failure mode; reduces hallucinated tool usage.
  - Makes executions auditable (validated request → deterministic call).
  - Enables reliability measurement (reject/repair rates, schema violation rates).
- **Cons**
  - Extra engineering: schemas, validators, policy engine, error taxonomy.
  - More round trips when the planner needs to repair invalid plans.
  - Tool surface must be well-specified; ad-hoc tools are harder to integrate.

## When to use / When NOT to use
**Use when:**
- You run agents that can mutate state (tickets, payments, infrastructure changes).
- You need compliance/audit logs of actions taken.
- You see frequent tool-call failures due to malformed arguments or tool name drift.

**Do NOT use when:**
- You only do read-only retrieval and failures are low impact.
- You are prototyping rapidly and the tool surface changes daily (unless you accept high maintenance).

## Implementation notes
- Languages: works well in **TypeScript** (zod/ajv), **Python** (pydantic/jsonschema), **.NET** (System.Text.Json + validators).
- Libraries:
  - JSON Schema validation (ajv / jsonschema / NJsonSchema)
  - Policy engine (OPA/Rego optional)
  - Structured tool interface (OpenAI tool schema, Anthropic tools, MCP tool descriptors)
- Effort: ~2–5 days for a small toolset (3–10 tools) to define schemas + executor + logging; longer for enterprise policy integration.

