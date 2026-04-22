# Executor Trust Boundary for Agent Orchestration

## Source
- **RUX - AI Orchestration Engine** (GitHub README)
- https://github.com/rahulT-17/RUX_AI_Orchestration_Engine

## Problem
Tool-using agents often follow **LLM → tool → response**, which fails in production when the model:
- hallucinates tool/action names or arguments
- emits malformed JSON
- attempts unsafe state-changing actions
- silently fails without an auditable contract

## Solution
Introduce an explicit **Executor** component as a *trust boundary* between probabilistic LLM output and deterministic tool execution.

### Architecture (concrete)
1. **Planner (LLM)**: converts the user message into an *intended action* (structured JSON).
2. **Executor (Deterministic)**:
   - validates the planner output against a strict schema (e.g., Pydantic with `extra="forbid"`)
   - normalizes/repairs minor format issues
   - rejects unknown action types, unknown fields, missing required args
   - enforces auth/quotas/guardrails
3. **Tool Adapter**: converts validated action into an internal tool call (stable interface).
4. **Domain Service**: executes business rules (idempotency, invariants, permissions).
5. **Repository/DB**: persists state changes.
6. **Observability + Outcome Tracking**: store action request, validation result, tool result, and user feedback.
7. **Critic** (optionally different model): independently critiques planner output/outcome.

### Implementation details
- Model outputs *only* a JSON action object (no free-form) for action-taking paths.
- Executor schema examples:
  - `action_type: "create_expense" | "update_project" | ...`
  - `args: { ... }`
  - strict enum validation
- RUX also describes a **three-layer planner**:
  - Layer 1: deterministic greeting handling
  - Layer 2: LLM extracts structured intent
  - Layer 3: conversational response

## Trade-offs
- **More engineering**: you must define and maintain schemas/contracts per tool/action.
- **Reduced flexibility**: new actions require schema + adapter changes (but that is also a safety feature).
- **Possible false rejects**: strict validation can reject near-miss outputs; you may need a repair step.
- **Latency overhead**: additional validation + logging steps (usually small vs model/tool latency).

## When to use / When NOT to use
**Use when**:
- tools mutate state (tickets, money, infra, permissions)
- you need audit logs and reproducible failures
- you see hallucinated tool names/args causing silent breakage

**Do NOT use when**:
- agent is pure Q&A with no external side effects
- prototyping with minimal reliability requirements

## Implementation notes
- **Languages**: Python (FastAPI + Pydantic) is a straightforward fit; pattern is language-agnostic.
- **Libraries**: Pydantic/JSON Schema, OpenTelemetry, rate limit middleware, RBAC/auth.
- **Effort**: ~2–7 days to implement for a small tool set (schema + executor + logging); longer if you add outcome-driven confidence/critique loops.
