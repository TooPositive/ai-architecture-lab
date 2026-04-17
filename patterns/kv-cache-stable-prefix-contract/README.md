# KV-Cache Stable Prefix Contract Pattern

## Source
- **Claude Code with local backends: KV cache invalidation caused by dynamic system-prompt injection (60-second issue) and settings.json fix** (Reddit PSA; summarized in internal knowledge base)
  - (No public URL captured in the knowledge node; internal reference id: `search-knowledge-7ab40724-38b2-49ad-ba9b-70f52013dd14`)

## Problem
Some agent shells/CLIs inject *changing* metadata (telemetry headers, git status, dynamic tool payloads) into the **system/prefix** on every request. On local inference backends (llama.cpp / LM Studio / llama-server), this breaks **prefix matching** and flushes the model **KV cache**, forcing the backend to re-process large prompt prefixes for every tool call. Result: extreme latency spikes (e.g., ~60s turns) and wasted compute.

## Solution
Establish a **Stable Prefix Contract** across turns so the model sees a byte-identical (or token-identical) prefix section, enabling KV-cache reuse.

Architecture/implementation steps:
1. **Split prompts into (A) stable prefix and (B) volatile suffix**.
   - Stable prefix: system instructions, tool schemas/JSON schema, formatting rules.
   - Volatile suffix: current user input, retrieved passages, tool outputs, ephemeral telemetry.
2. **Move dynamic metadata out of the system prompt**.
   - Don’t inject telemetry headers, git status, timestamps, etc. into the prompt prefix.
   - Put them into an external log channel, tracing system, or tool outputs that are *not* replayed into the model prefix.
3. **Freeze tool definitions**.
   - Keep tool schema ordering stable; avoid re-serializing with nondeterministic key order.
   - Version toolsets explicitly (e.g., `toolset_version=17`) and only change when necessary.
4. **Client-side enforcement** (example from source): adjust the CLI/app configuration (e.g., `~/.claude/settings.json`) to reduce/disable dynamic prompt mutations so prefix caching works.

## Trade-offs
- **Pros**: Major latency reduction on local backends; lower CPU/GPU utilization; more predictable p95 latency; improved throughput.
- **Cons**: Reduced “helpfulness” from dynamic context (e.g., auto-including git status) unless provided through separate tool calls; requires stricter prompt/tool serialization discipline; tool/schema updates invalidate cache by design.

## When to use / When NOT to use
**Use when**:
- Running agents against **local** inference servers relying on prefix matching/KV cache reuse.
- You have long system prompts/tool schemas that are reused across many short turns.

**Don’t use when**:
- You only make single-shot calls (no multi-turn reuse).
- Your provider already offers managed prompt caching and you can’t control the prompt prefix contents.

## Implementation notes
- Works with: llama.cpp/llama-server, LM Studio, vLLM (prefix caching), any stack where prompt-prefix caching exists.
- Key tactic: deterministic prompt builder (stable ordering, stable whitespace, stable JSON).
- Effort: ~0.5–2 days to implement in an agent shell; more if you must refactor tool injection.
