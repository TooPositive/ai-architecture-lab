# ADR-0004: Add Prefix-Stable System Prompt (KV-Cache Preservation)

**Date**: 2026-04-01
**Status**: Accepted
**Pipeline**: pattern-lab

## Context

Source reports Claude Code injects dynamic telemetry headers and git status into the system prompt each request, breaking prefix matching for local inference backends (llama.cpp / LM Studio), flushing KV cache, and forcing re-processing of a 20K+ token system prompt. Pattern adds an architectural rule: keep the system prefix bit-identical and move dynamic fields out of the prefix to preserve KV/prefix cache reuse.

## Decision

Add Prefix-Stable System Prompt (KV-Cache Preservation)

## Consequences

This pattern/content has been added to the architecture lab repository.