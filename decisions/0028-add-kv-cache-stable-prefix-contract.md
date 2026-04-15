# ADR-0028: Add KV-Cache Stable Prefix Contract

**Date**: 2026-04-15
**Status**: Accepted
**Pipeline**: pattern-lab

## Context

Source describes Claude Code becoming extremely slow with local backends because it injects changing telemetry headers and git-status updates into the system prompt each request, breaking prefix matching and flushing KV cache. Pattern formalizes a stable-prefix contract and moving dynamic metadata out of the prompt to preserve KV-cache reuse.

## Decision

Add KV-Cache Stable Prefix Contract

## Consequences

This pattern/content has been added to the architecture lab repository.