# ADR-0023: Add KV-Cache Stable Prefix Contract (Prevent Prefix-Cache Invalidation)

**Date**: 2026-04-10
**Status**: Accepted
**Pipeline**: pattern-lab

## Context

Source describes Claude Code mutating the system prompt each request (telemetry headers, git status), which breaks prefix matching on local backends and flushes KV cache; mitigating via settings.json and keeping the prompt prefix stable so caching works (KB node search-knowledge-7ab40724-38b2-49ad-ba9b-70f52013dd14).

## Decision

Add KV-Cache Stable Prefix Contract (Prevent Prefix-Cache Invalidation)

## Consequences

This pattern/content has been added to the architecture lab repository.