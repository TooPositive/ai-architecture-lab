# ADR-0009: Add MCP Context Gateway (Sandboxed Tool Outputs + FTS Retrieval)

**Date**: 2026-04-03
**Status**: Accepted
**Pipeline**: pattern-lab

## Context

Source: “Stop Burning Your Context Window — We Built Context Mode” (https://mksg.lu/blog/context-mode). The article describes an MCP server that intercepts high-output tool calls, runs them in an isolated subprocess, and returns only compact stdout to the LLM, keeping raw artifacts out of context. It also persists session events into SQLite and rehydrates relevant state via SQLite FTS5 BM25 retrieval on compaction, enabling long tool-using sessions with dramatically reduced context consumption (example: 315KB raw output → 5.4KB, 98% reduction).

## Decision

Add MCP Context Gateway (Sandboxed Tool Outputs + FTS Retrieval)

## Consequences

This pattern/content has been added to the architecture lab repository.