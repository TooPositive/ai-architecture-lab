# Prefix-Stable System Prompt (KV-Cache Preservation)

## Source
- **"PSA: Using Claude Code without Anthropic: How to fix the 60-second local KV cache invalidation issue."** (Reddit r/LocalLLaMA)
  - URL: https://www.reddit.com/r/LocalLLaMA/comments/1s7tn5s/psa_using_claude_code_without_anthropic_how_to/

## Problem
When running agentic tooling (e.g., Claude Code) against a local inference backend (llama.cpp / llama-server, LM Studio, etc.), the client may inject small but changing data into the system prompt every request (telemetry headers, git status). This breaks prefix matching on the server, invalidates the KV cache, and forces re-processing of a large static prefix (the system prompt) on every tool call.

Symptoms described in the source:
- Prefix match is broken by dynamic prompt injection
- KV cache is flushed frequently ("60-second" invalidation issue)
- Large system prompts (e.g., 20K+ tokens) are re-evaluated repeatedly

## Solution
Architect the agent runtime so that the prompt prefix fed to the inference server remains bit-identical across turns. Move all dynamic or frequently changing information out of the system prefix and into:
1) request metadata (HTTP headers not included in prompt), or
2) a separate user/tool message appended after the stable prefix, or
3) tool state stored server-side (session id / thread id) rather than pasted into prompt.

Concrete implementation steps (as implied by the source):
- Audit the client that builds prompts and identify any non-deterministic fields in system content (timestamps, telemetry ids, git status, environment details).
- Configure the client to disable or freeze those fields (the post mentions a fix via ~/.claude/settings.json).
- Ensure the server uses KV-cache/prefix-cache and is configured to reuse it for identical prefixes.

## Trade-offs
- **Pros:**
  - Large latency reduction for multi-turn/tool-heavy loops when the static prefix is large (avoid repeatedly re-prefilling 10K–20K+ tokens).
  - Lower compute cost on local GPUs/CPUs; higher throughput.
- **Cons / Risks:**
  - If you remove dynamic context (e.g., repo status), you must reintroduce it via tools/messages, which can slightly increase complexity and require careful placement so the model still sees it when needed.
  - Requires discipline across the whole stack: any small change in the prefix (whitespace, ordering, auto-added headers) can negate caching.

## When to use / When NOT to use
**Use when:**
- You run local/open inference servers that support prefix/KV caching.
- Your agent uses a large static system prompt and makes frequent short tool calls.
- You observe high prefill time despite small incremental turns.

**Avoid when:**
- Your runtime/provider does not expose or benefit from KV/prefix caching.
- Your system prompt must legitimately change every turn (e.g., strict dynamic policy injection requirements).

## Implementation notes
- **Where:** client-side prompt construction (agent framework), plus server settings.
- **Languages/libraries:** any; relevant for llama.cpp/llama-server or similar backends with KV cache.
- **Effort:** ~0.5–2 days to instrument prompt generation, locate dynamic fields, and validate prefix identity (hashing) + measure latency.
- **Validation:** log a hash of the system prefix each request; confirm it stays constant; measure prefill time and tokens/s before/after.
