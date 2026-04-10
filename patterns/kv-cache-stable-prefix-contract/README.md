# KV-Cache Stable Prefix Contract (Prevent Prefix-Cache Invalidation)

## Source
- **"Claude Code with local backends: KV cache invalidation caused by dynamic system-prompt injection (60-second issue) and settings.json fix"** (Reddit PSA; summarized in internal KB node `search-knowledge-7ab40724-38b2-49ad-ba9b-70f52013dd14`).

## Problem
Prefix/KV caching in local inference backends (e.g., llama.cpp servers) relies on the *initial prompt prefix staying identical* across turns. Some agent shells/CLIs mutate the system prompt every request (telemetry headers, git status, tool definitions, etc.), which breaks prefix matching, flushes the KV cache, and forces the backend to re-process a large system prompt on every tool call. This can turn sub-second turns into "60-second" latency.

## Solution
Introduce an explicit **Stable Prefix Contract** between your agent runtime and the inference backend:

1. **Split prompts into stable vs dynamic segments**
   - Stable: system instructions, tool schemas, policies, static developer prompt.
   - Dynamic: telemetry, git status, recent messages, tool results.

2. **Freeze the stable prefix bytes**
   - Ensure the stable segment is identical across turns (byte-for-byte), including whitespace and ordering.
   - Do not inject request-specific metadata into the system prompt.

3. **Move changing data out of the prefix**
   - Send changing metadata via HTTP headers, out-of-band logging, or a separate side channel.
   - If the client cannot avoid injection, configure it (e.g., `~/.claude/settings.json` in the source) to disable/limit dynamic prompt mutations.

4. **Cache-aware tool packaging**
   - Avoid re-sending large tool definition payloads every turn.
   - If your framework forces this, consider a proxy that replaces repeated tool schemas with stable references (e.g., `toolset_id`) and expands server-side.

### Concrete implementation
- **Gateway/Proxy** in front of the model server:
  - Receives `system_static` once (or hashes it into `prefix_id`).
  - For each request: sends the same `system_static` prefix + only the changing messages.
  - Rejects requests where `system_static_hash` changes unexpectedly (protects caching + reproducibility).
- **Client discipline**:
  - Deterministically serialize tools (stable sort keys, stable JSON formatting).
  - Keep "environment state" (git status, traces) out of system prompt.

## Trade-offs
- **Pros**
  - Large latency reduction on local backends due to KV/prefix cache reuse (source describes repeated re-processing of ~20K+ token system prompts when invalidated).
  - Lower compute cost and higher throughput for tool-heavy agents.
  - Improves reproducibility/debuggability: stable system prompts make prompt diffs meaningful.
- **Cons**
  - Less convenient: frameworks often treat system prompt as the dump site for runtime metadata.
  - Requires careful deterministic serialization (whitespace/order changes can defeat caching).
  - If you truly need dynamic system policy updates, you must intentionally break the contract and accept cache misses.

## When to use / When NOT to use
- **Use when**
  - Running agents on **local inference** (llama.cpp/LM Studio/vLLM) where prefix caching materially affects latency.
  - Tool-heavy workflows that would otherwise resend large tool schemas per turn.
  - You need predictable latency for interactive shells (IDE agents, code assistants).
- **Do NOT use when**
  - You cannot control the client/runtime prompt construction (SaaS agent shells with hardcoded injection).
  - The system prompt must change every request for compliance/safety reasons.

## Implementation notes
- **Languages**: any (proxy can be Node/Go/Python/.NET).
- **Libraries**:
  - Reverse proxy: FastAPI, Express, ASP.NET minimal APIs, Envoy/Lua.
  - Deterministic JSON: `orjson` (Python), `System.Text.Json` with stable options (.NET), `serde_json` (Rust).
- **Effort**: ~0.5–1 day to add a proxy + prompt-hash enforcement; longer if refactoring a framework that always re-emits tools.
