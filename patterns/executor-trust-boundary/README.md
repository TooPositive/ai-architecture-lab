# Executor Trust Boundary (Untrusted LLM, Deterministic Executor)

## Source
- **"RUX — AI orchestration engine design with an untrusted LLM and a deterministic Executor trust boundary"** (Reddit post, summarized in internal KB)
  - KB node: `search-knowledge-0571dd16-0cb4-49c1-982f-9ac8e1ef8d85`

## Problem
Tool-using agents often fail silently when the model hallucinates tool names/arguments or outputs an invalid action plan. In the common pattern **LLM → tool → response**, there is no explicit trust boundary: probabilistic outputs directly trigger real side effects.

## Solution
Treat the LLM as an **untrusted planner**, and insert a **deterministic Executor** as the sole component allowed to invoke tools.

### Architecture
1. **Planner (LLM)** produces a *proposed* plan in a strict, machine-parseable format (e.g., JSON).
2. **Executor (deterministic)** validates and normalizes the plan against an allowlist + schemas:
   - Allowed tool names
   - Required/optional args + types
   - Policy constraints (rate limits, resource scopes, tenant permissions)
   - Rewrites/repairs minor issues or rejects with actionable errors
3. **Tool Adapter(s)** execute side-effecting calls only after Executor approval.
4. **Service layer** performs real business operations.

### Concrete implementation details
- Define a **Tool Contract** per tool: JSON Schema / OpenAPI / protobuf.
- Executor performs:
  - Schema validation (tool name + args)
  - Authorization checks (RBAC/ABAC, per-tool scopes)
  - Deterministic parameter defaults/normalization
  - Replay protection + idempotency keys for side effects
  - Observability: structured logs per plan step, rejection reasons
- Only the Executor can call tools; the Planner cannot.

## Trade-offs
- **Pros**
  - Prevents silent failures from hallucinated tools/actions (Executor rejects invalid actions explicitly).
  - Creates an audit trail of proposed vs executed actions.
  - Enables deterministic safety and compliance controls independent of model behavior.
- **Cons**
  - More up-front engineering: schemas, allowlists, validation logic, idempotency.
  - The agent may require additional turns when plans are rejected and need regeneration.
  - Rigid schemas can reduce agent flexibility unless you evolve contracts carefully.

## When to use / When NOT to use
### Use when
- Any tool call has **real side effects** (payments, account changes, infrastructure actions).
- You need **reliability measurement** and explicit failure modes.
- You operate in regulated environments requiring **policy enforcement + auditability**.

### Avoid when
- The agent is purely read-only and failures are harmless, and you need maximal iteration speed.
- You cannot commit to maintaining tool schemas/contracts (fast-changing prototypes).

## Implementation notes
- **Languages**: works well in strongly typed stacks (C#, Java, Go, Rust) but also fine in TypeScript/Python with JSON Schema.
- **Libraries**:
  - JSON Schema validation: `Ajv` (TS), `jsonschema`/`pydantic` (Py), `NJsonSchema` (C#).
  - Tool contracts: OpenAPI + codegen, or protobuf.
- **Effort**:
  - 1–3 days for a minimal allowlisted executor with schema validation and structured logging.
  - 1–2 weeks to add policy engine integration, idempotency, retries, replay protection, and tooling for contract evolution.
