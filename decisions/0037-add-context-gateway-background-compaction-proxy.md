# ADR-0037: Add Context Gateway: Background Compaction Proxy

**Date**: 2026-04-22
**Status**: Accepted
**Pipeline**: pattern-lab

## Context

Source: compresr — Context Gateway docs https://compresr.ai/docs/gateway . Describes a transparent proxy that monitors token usage, pre-computes summaries at a configurable threshold (default 75% of context limit), performs instant compaction when limit is reached (no waiting), compresses large tool outputs on the fly, and logs compression events. Intended to prevent context exhaustion and latency spikes in long-running tool-heavy agent sessions.

## Decision

Add Context Gateway: Background Compaction Proxy

## Consequences

This pattern/content has been added to the architecture lab repository.