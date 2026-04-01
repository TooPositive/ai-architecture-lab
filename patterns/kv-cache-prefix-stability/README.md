# KV-Cache Prefix Stability (System-Prompt Hygiene)

## Source
- **PSA: Using Claude Code without Anthropic: How to fix the 60-second local KV cache invalidation issue.**
- https://www.reddit.com/r/LocalLLaMA/comments/1s7tn5s/psa_using_claude_code_without_anthropic_how_to/

## Problem
When running an agent (e.g., Claude Code) against a **local inference backend** (llama.cpp / llama-server / LM Studio), repeated requests should benefit from **prefix/KV caching**. However, if the agent injects **dynamic content** into the system prompt on each request (telemetry headers, git status, timestamps), the prompt prefix changes, **breaking prefix matching** and forcing the backend to re-process a large system prompt (e.g., 20K+ tokens) every call. This increases latency and compute dramatically.

## Solution
Enforce **prefix stability** for the “static” part of prompts so KV caching can reliably reuse the cached prefix across turns.

Implementation details (from the source):
- Identify dynamic system-prompt injections (telemetry headers, git status updates).
- Disable or move these dynamic fields out of the static system prompt.
- Configure the agent (via its settings file) so that repeated requests reuse an identical system prompt prefix, enabling **prefix matching** and preventing frequent KV-cache flushes/invalidations.

## Trade-offs
- **Less ambient context**: removing git status/telemetry from the system prompt may reduce automatic awareness (you may need explicit tool calls to fetch state).
- **Observability changes**: telemetry may need to be moved to logs rather than the model prompt.
- **Backend dependence**: benefits are largest on backends that implement prefix/KV cache reuse well; cloud APIs may hide this.

## When to use / When NOT to use
**Use when:**
- You serve local models and observe repeated “re-prefill” costs or high latency for tool calls.
- Your system prompt is large (policy + tools + instructions) and is re-sent frequently.

**Do NOT use when:**
- You rely on per-request dynamic system instructions for safety/compliance and cannot safely relocate them.
- You’re using an API/provider where prompt caching is abstracted or not impacted.

## Implementation notes
- **Where to fix**: agent config/settings (example in source: `~/.claude/settings.json`).
- **Effort**: hours to diagnose + adjust; add monitoring to detect prompt prefix drift over time.
- **Optional add-on**: a “prompt normalizer” middleware that strips known-dynamic fields from the cached prefix and places them in a separate metadata channel/log.
