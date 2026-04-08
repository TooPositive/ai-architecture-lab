# Untrusted-LLM Executor Trust Boundary Pattern

## Source
- **"RUX — AI orchestration engine design with an untrusted LLM and a deterministic Executor trust boundary" (Reddit post)** (knowledge base node: `search-knowledge-0571dd16-0cb4-49c1-982f-9ac8e1ef8d85`)

## Problem
Tool-using agents often fail silently or unsafely because the LLM can hallucinate tool names/arguments, skip validation, or attempt disallowed actions. A common naive flow is **LLM → tool → response** with minimal guardrails.

## Solution
Introduce an explicit **Executor** component as a *trust boundary* between probabilistic LLM output and deterministic tool execution.

### Architecture
1. **Planner (LLM)** produces a proposed plan in a constrained, structured format (e.g., JSON).
2. **Executor (deterministic)** validates and normalizes the plan against:
   - an allowlist of actions/tools
   - strict JSON schema / function signature constraints
   - policy checks (authz, rate limits, data access scopes)
   - argument sanitization
3. **Tool** invocation occurs only from the Executor after validation.
4. **Service** performs real side effects (DB writes, network calls, deployments).

### Concrete implementation details
- Define a versioned **Plan schema** (e.g., `Plan{ steps:[{tool:"search", args:{...}}] }`).
- Executor responsibilities:
  - parse/validate against schema
  - map tool aliases → canonical tool IDs
  - reject unknown tools/fields
  - enforce step budget (max steps, max cost)
  - optionally require **preconditions** (e.g., confirm user intent for destructive actions)
- Emit structured execution logs for reliability measurement (accepted/rejected plans, failure reason taxonomy).

## Trade-offs
- **Pros**
  - Prevents hallucinated tool calls from executing (hard failure vs silent failure).
  - Creates a measurable reliability layer (accept/reject rates, failure modes).
  - Simplifies security review: only Executor needs to be trusted for side effects.
- **Cons**
  - Extra engineering: schema design, validation, policy engine, and tool adapters.
  - May reduce capability if schema is too restrictive (LLM can’t express novel strategies).
  - Requires ongoing maintenance as tool surface area evolves.

## When to use / When NOT to use
### Use when
- Your agent can trigger **side effects** (write operations, transactions, privileged APIs).
- You need **auditability** and deterministic enforcement for compliance/security.
- You observe failures from tool hallucination, malformed args, or ambiguous actions.

### Avoid when
- Pure read-only chat/RAG without tools or side effects (overkill).
- Rapid prototypes where reliability/security requirements are low.

## Implementation notes
- **Languages**: any; commonly implemented as a service/middleware layer (TypeScript, Python, C#, Go).
- **Libraries**:
  - JSON schema validation (Ajv for TS, jsonschema/pydantic for Python)
  - policy checks (OPA/Rego optional)
- **Effort**: ~2–5 engineer-days for a minimal version (schema + allowlist + logging), longer for policy/audit + complex tools.
