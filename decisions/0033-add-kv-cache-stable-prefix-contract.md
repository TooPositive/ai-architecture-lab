# ADR-0033: Add KV-Cache Stable Prefix Contract

**Date**: 2026-04-17
**Status**: Accepted
**Pipeline**: pattern-lab

## Context

Source describes a real-world failure mode: dynamic telemetry headers and git-status updates injected into the system prompt on each request break prefix matching and flush the KV cache on local inference backends, causing extreme latency. Pattern codifies a stable-prefix contract and prompt/tool serialization discipline to preserve KV-cache reuse.

## Decision

Add KV-Cache Stable Prefix Contract

## Consequences

This pattern/content has been added to the architecture lab repository.