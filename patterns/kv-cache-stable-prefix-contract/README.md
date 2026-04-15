# KV-Cache Stable Prefix Contract

## Source
- **Claude Code with local backends: KV cache invalidation caused by dynamic system-prompt injection (60-second issue) and settings.json fix** (Reddit PSA; indexed in our KB)
  - URL: *not captured in KB entry* (knowledge node `search-knowledge-7ab40724-38b2-49ad-ba9b-70f52013dd14`)

## Problem
Local inference backends (e.g., llama.cpp server / LM Studio) rely heavily on **prefix matching** to reuse the model **KV cache** across consecutive calls.

Some agent clients/IDEs mutate the beginning of the prompt on every request (telemetry headers, dynamic git status, changing tool definitions). This breaks prefix matching and causes **full KV cache invalidation**, forcing re-processing of a large system prompt (reported as ~20k+ tokens / ~62.6k characters of tool definitions) for even tiny follow-up tool calls. Result: extreme latency ("60-second" turns) and higher cost.

## Solution
Create an explicit **Stable Prefix Contract** between your agent runtime and the model gateway so the initial part of every request is byte-identical across turns.

### Architecture / Implementation details
1. **Split prompt into two regions**:
   - **Stable prefix** (must not change):
     - system instructions
     - tool schemas / function signatures
     - static policies
   - **Dynamic suffix** (allowed to change):
     - conversation turns
     - tool results
     - state, scratchpads

2. **Move dynamic metadata out of the prompt**:
   - Telemetry headers: send via HTTP headers or logging, not system prompt text.
   - Git status / repo context: fetch via a tool call and include only when needed, not injected every turn at the top.

3. **Pin tool definitions**:
   - Ensure tool schema JSON is generated once at startup and referenced consistently.
   - Avoid re-serializing maps with non-deterministic key order.
   - Normalize whitespace and ordering for stable hashing.

4. **Gateway enforcement** (recommended):
   - Add a middleware/proxy that computes a hash of the stable prefix and rejects/flags requests where the prefix changes unexpectedly.
   - Optionally rewrite prompts to restore the last stable prefix and move diffs into the suffix.

5. **Client-side mitigation** (per source):
   - Adjust the client configuration (`~/.claude/settings.json`) to reduce/disable dynamic injections so prefix matching works.

## Trade-offs
- **Pros**:
  - Large latency improvement on local backends due to KV-cache reuse.
  - Lower cost on repeated calls where providers price cached tokens differently or where less compute is used.
  - More predictable performance for agent loops (tool-use heavy).
- **Cons**:
  - You may lose "always-on" dynamic context (e.g., up-to-date git status) unless you explicitly request it via tools.
  - Requires discipline: any change to tool schemas/system policy becomes a cache-busting event.
  - If you enforce via gateway, you must maintain canonical prompt serialization.

## When to use / When NOT to use
### Use when
- You run **local** inference servers (llama.cpp/LM Studio/vLLM) where KV cache reuse matters.
- You have **agentic workflows** with many short successive calls (planner/executor/tool loops).
- Your system prompt includes large tool schemas/policies that are constant across turns.

### Avoid when
- Your application legitimately needs dynamic top-of-prompt context every turn and you cannot restructure it into tool calls.
- You use a provider/client that already handles prompt-prefix caching robustly and you have no latency/cost pressure.

## Implementation notes
- **Languages/libraries**: any; gateway easiest in Go/Node/.NET as an HTTP reverse proxy.
- **Effort**: ~0.5–2 days to implement canonicalization + prefix hashing + alerts; more if you add rewrite/auto-fix.
- **Key technique**: deterministic serialization of tool schemas and stable system prompt text.

## Classification
- **NEW_PATTERN** (operational pattern: making KV-cache/prefix-caching a first-class contract, motivated by concrete failure mode in the source).
