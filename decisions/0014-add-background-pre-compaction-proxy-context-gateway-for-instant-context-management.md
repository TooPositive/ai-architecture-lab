# ADR-0014: Add Background pre-compaction proxy (Context Gateway) for instant context management

**Date**: 2026-04-06
**Status**: Accepted
**Pipeline**: pattern-lab

## Context

Compresr’s Context Gateway describes a transparent proxy that monitors token usage, pre-computes summaries at a threshold (75% by default), performs instant compaction at the limit, compresses large tool outputs, and logs compression events (JSONL). Sources: https://compresr.ai/docs/gateway and https://github.com/Compresr-ai/Context-Gateway

## Decision

Add Background pre-compaction proxy (Context Gateway) for instant context management

## Consequences

This pattern/content has been added to the architecture lab repository.