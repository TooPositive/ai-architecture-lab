# ADR-0007: Add Context Virtualization via MCP Sandboxing + Event Indexing

**Date**: 2026-04-03
**Status**: Accepted
**Pipeline**: pattern-lab

## Context

Source describes Context Mode as an MCP server that keeps raw tool outputs out of the LLM context (315 KB to 5.4 KB; 98% reduction) and preserves session continuity by logging events in SQLite, indexing with FTS5, and retrieving via BM25 instead of re-injecting full history during compaction. See https://github.com/mksglu/context-mode

## Decision

Add Context Virtualization via MCP Sandboxing + Event Indexing

## Consequences

This pattern/content has been added to the architecture lab repository.