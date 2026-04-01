# ADR-0003: Add KV-Cache Prefix Stability (System-Prompt Hygiene)

**Date**: 2026-04-01
**Status**: Accepted
**Pipeline**: pattern-lab

## Context

From reddit post “PSA: Using Claude Code without Anthropic: How to fix the 60-second local KV cache invalidation issue.” (https://www.reddit.com/r/LocalLLaMA/comments/1s7tn5s/psa_using_claude_code_without_anthropic_how_to/): dynamic telemetry headers and git status updates injected into the system prompt can break prefix matching on local backends, flushing KV cache and forcing re-processing of large prompts. This ADR captures the pattern of keeping the prompt prefix stable and relocating dynamic context outside the cached prefix to restore KV-cache reuse.

## Decision

Add KV-Cache Prefix Stability (System-Prompt Hygiene)

## Consequences

This pattern/content has been added to the architecture lab repository.