# ADR-0032: Add Context Sandbox + FTS5 Tool-Output Indexing (MCP)

**Date**: 2026-04-17
**Status**: Accepted
**Pipeline**: pattern-lab

## Context

Source describes Context Mode, an MCP server that intercepts tool calls, prevents large raw outputs from entering the LLM context, indexes outputs locally in SQLite FTS5 (BM25), and injects small, prioritized session snapshots on compaction to preserve continuity. This ADR adds a reusable architecture pattern implementing that approach for long-running agent sessions.

## Decision

Add Context Sandbox + FTS5 Tool-Output Indexing (MCP)

## Consequences

This pattern/content has been added to the architecture lab repository.