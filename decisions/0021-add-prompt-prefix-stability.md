# ADR-0021: Add Prompt Prefix Stability

**Date**: 2026-04-10
**Status**: Accepted
**Pipeline**: pattern-lab

## Context

Source describes latency blowups when Claude Code dynamically mutates the system prompt each turn (telemetry/git status/tool defs), which breaks prefix matching and invalidates KV cache on local backends. Pattern: keep long-lived prefix stable; move dynamic data later so KV caching works.

## Decision

Add Prompt Prefix Stability

## Consequences

This pattern/content has been added to the architecture lab repository.