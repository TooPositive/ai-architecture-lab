# Prompt Prefix Stability (KV-Cache Friendly System Prompt Design)

## Source
- **"Claude Code with local backends: KV cache invalidation caused by dynamic system-prompt injection (60-second issue) and settings.json fix"** (Reddit PSA, summarized in internal KB)
  - KB node: `search-knowledge-7ab40724-38b2-49ad-ba9b-70f52013dd14`

## Problem
Local/self-hosted inference backends (e.g., llama.cpp server, LM Studio) heavily rely on **prefix matching + KV cache reuse**. If your agent/client mutates the system prompt prefix on every turn (telemetry headers, dynamic tool definitions, git status, etc.), the backend cannot reuse cached attention keys/values and must re-process the entire prefix each request—causing extreme latency spikes (e.g., "~60-second" turns) and wasted compute.

## Solution
Design the orchestration layer so that the **prompt prefix is stable** across turns, enabling KV cache reuse.

### Architecture / implementation details
1. **Freeze the long-lived prefix**
   - Keep system instructions and tool definitions in a stable, deterministic order.
   - Do not inject per-turn telemetry/generation metadata into the system message.
2. **Move dynamic data out of the prefix**
   - Put ephemeral state (git status, timestamps, run IDs) into a *later* message (e.g., user/assistant message) or into tool-call args.
3. **Separate tool definitions from the prompt (if supported)**
   - Prefer API-level tool/function schemas that are not serialized into tokens every turn.
4. **Client configuration guard**
   - Provide a config flag (like the PSA’s `~/.claude/settings.json` mitigation) that disables dynamic injections when using local backends.
5. **Observability**
   - Trace request payloads and log prefix length + whether prefix cache hit occurs (if the backend exposes it).

## Trade-offs
- **Pros**
  - Large reductions in per-turn latency on KV-cache capable backends.
  - Lower compute cost and better throughput for multi-turn agents.
- **Cons**
  - Less flexibility for constantly changing tool sets/telemetry embedded in prompts.
  - Requires discipline and testing to keep serialization stable (ordering, whitespace, nondeterminism).

## When to use / When NOT to use
### Use when
- You run **local inference** or any backend that benefits from prompt-prefix caching.
- Your agent has long tool definitions / long system prompts reused across turns.

### Avoid when
- Your provider already abstracts this away and you don’t control prompt composition.
- Your workflow truly needs to mutate system-level policy each turn (rare).

## Implementation notes
- **Languages**: any; fix is mostly in your agent runtime/client.
- **Effort**: typically hours to a day to refactor prompt assembly and add a "stable prefix" mode + tracing.
