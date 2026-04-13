# Planner→Executor Trust Boundary (Untrusted LLM, Deterministic Tool Execution)

## Source
- *How I Built an AI Orchestration Engine Without LangChain in 2026* — Nathan Cole — RoboRhythms (2026-04-03)
  https://www.roborhythms.com/how-to-build-ai-orchestration-without-langchain-2026/

## Problem
Tool-using agents often run as **LLM → tool → response** with little validation. When the LLM hallucinates a tool name/args or decides on a destructive action, failures can be silent (wrong tool, malformed params) or unsafe (calling a DELETE endpoint).

## Solution
Split the agent runtime into **three explicit layers** with a hard trust boundary:

1. **Planner (LLM)**
   - Only produces a **structured plan** (e.g., JSON) describing the intended tool call.
   - Must not execute tools.
   - Prompt constrains output to a fixed schema (tool name ∈ allowlist, args dict, brief reasoning).

2. **Trust Boundary / Validator (deterministic code)**
   - Parse the planner output (e.g., `json.loads`).
   - Validate structure with a schema library (example in source uses **Pydantic**).
   - Enforce an **explicit allowlist** of tools and (optionally) argument constraints.
   - Fail loudly on any mismatch; optionally retry planner with tighter constraints.

3. **Executor (deterministic code)**
   - Maps validated `tool` → function reference and executes with `**args`.
   - Logs decisions, tool calls, and results for observability and debugging.

Concrete implementation sketch (adapted from the source):

- Register tools in a dictionary:
  - `TOOLS = {"get_weather": get_weather, "search_docs": search_docs}`
- Planner prompt outputs JSON only:
  - `{ tool: one of [...], args: dict, reasoning: str }`
- Validation:
  - `ToolCall(BaseModel): tool:str, args:dict, reasoning:str`
  - `if call.tool not in TOOLS: raise`
- Execute:
  - `result = TOOLS[call.tool](**call.args)`

## Trade-offs
- **More code paths**: you now own parsing, retries, and error handling instead of delegating to an orchestration framework.
- **Schema/allowlist maintenance**: every new tool requires updating validation contracts and tests.
- **Not a full sandbox**: this pattern prevents accidental/invalid tool calls, but does not fully isolate side effects. Destructive tools still need additional safeguards (authz, rate limits, dry-run, approvals).

## When to use / When NOT to use
**Use when**
- Any tool call has side effects (write/delete, payments, production changes).
- You’ve seen silent agent failures due to malformed tool names/arguments.
- You want minimal framework dependency and clear debuggability.

**Avoid when**
- You only need single-shot Q&A with no tool execution.
- You require fully sandboxed execution against untrusted external services (you’ll still need isolation, approvals, and least-privilege).

## Implementation notes
- **Languages**: straightforward in Python; pattern is portable to TypeScript/Go/C# (use JSON schema / Zod / Pydantic / System.Text.Json + validators).
- **Libraries**: Pydantic (Python), JSON Schema validators, structured logging (OpenTelemetry).
- **Effort**: ~0.5–2 days to implement a robust v1 (schema + retries + logging), plus ongoing work to add tool-specific guards/tests.
