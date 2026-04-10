# ADR-0024: Add MCP Context Compression Proxy (Tool-Output Compaction Layer)

**Date**: 2026-04-10
**Status**: Accepted
**Pipeline**: pattern-lab

## Context

Source describes MCP tool calls dumping raw payloads into the LLM context window; Context Mode is an MCP server interposed to compress outputs (example: 315KB -> 5.4KB, 98% reduction) before they hit the model (https://mksg.lu/blog/context-mode).

## Decision

Add MCP Context Compression Proxy (Tool-Output Compaction Layer)

## Consequences

This pattern/content has been added to the architecture lab repository.