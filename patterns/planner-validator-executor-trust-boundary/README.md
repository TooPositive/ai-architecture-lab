# Pattern: Planner→Validator→Executor trust boundary for tool-using agents

## Source
- Nathan Cole. **"How I Built an AI Orchestration Engine Without LangChain in 2026"** (RoboRhythms, Apr 3 2026)
  https://www.roborhythms.com/how-to-build-ai-orchestration-without-langchain-2026/

## Classification
**NEW_PATTERN** (the explicit *LLM-as-untrusted input* trust boundary implemented as a first-class Validator layer is not universal in day-to-day agent examples; the article gives a concrete minimal production architecture and code skeleton).

## Problem
Tool-using agents commonly follow `LLM → tool call → response` with minimal validation. In production this creates failure modes:
- The model hallucinates a tool name or argument schema → silent failures or undefined behavior.
- The model selects a dangerous tool/action (e.g., destructive endpoint) because nothing enforces scope/allowlists.
- Debugging is hard because you can’t reliably inspect *what the LLM decided* before execution.

## Solution
Introduce an explicit trust boundary between planning and execution.

### Architecture
1. **Planner LLM**: produces a *structured plan* (e.g., strict JSON) describing the intended tool + arguments and a short rationale.
2. **Validator (trust boundary)**: treats the plan as *untrusted input*.
   - Parse/validate structure (JSON parse, schema validation via Pydantic/Zod/etc.).
   - Enforce allowlists/denylists (tool names, argument ranges, resource constraints).
   - Optionally normalize arguments (defaults, coercions) and reject unsafe calls loudly.
3. **Executor**: deterministically calls the approved tool implementation (code you own) and returns results.

### Concrete implementation details (from the source)
- Constrain planner output with a system prompt: output JSON **only** with `{ tool, args, reasoning }`.
- Maintain a `TOOLS` registry keyed by tool name.
- Validator checks:
  - schema correctness (example uses **Pydantic** `BaseModel`)
  - tool allowlist (`call.tool in TOOLS`)
  - fail fast on parse/validation errors before any side effects occur

Minimal skeleton (paraphrased from the article):
- `plan = call_planner(user_query)`
- `validated = validate_plan(plan)`
- `result = execute(validated)`

## Trade-offs
- **More engineering upfront**: you must design tool schemas and validation rules.
- **Reduced flexibility**: strict JSON/schema constraints can cause more planner retries (parse failures) if prompts are weak.
- **Partial coverage**: validation can ensure only *allowed* calls run, but it doesn’t guarantee the plan is *correct*—only well-formed and permitted.
- **Additional latency**: validation is cheap, but retries on malformed JSON add latency.

## When to use / When NOT to use
### When to use
- Any agent with **side-effectful tools** (write APIs, delete endpoints, payments, infra actions).
- Regulated environments where you need explicit enforcement + auditability.
- Multi-tool agents where hallucinated tool names/args are common.

### When NOT to use
- Pure Q&A/chat without tools (no side effects).
- Very early prototypes where failure cost is low—though you’ll likely want this before production.

## Implementation notes
- Languages/libraries:
  - Python: Pydantic for schema validation (as shown in the source)
  - TypeScript: Zod / io-ts for schema validation
  - .NET: System.Text.Json + FluentValidation
- Effort:
  - 0.5–2 days for a minimal single-hop agent with 5–20 tools (schemas + allowlist checks + logging).
  - More if you add per-tool policy checks (rate limits, tenancy, RBAC) and observability.
- Operational notes:
  - Log `plan` and `validated_plan` separately for debugging/audits.
  - Treat any validation failure as a *first-class* error with retry/fallback behavior.
