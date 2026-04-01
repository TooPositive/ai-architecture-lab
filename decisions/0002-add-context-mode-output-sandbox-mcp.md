# ADR-0002: Add Context-Mode Output Sandbox (MCP)

**Date**: 2026-04-01
**Status**: Accepted
**Pipeline**: pattern-lab

## Context

From “Stop Burning Your Context Window — We Built Context Mode” (https://mksg.lu/blog/context-mode): MCP tool calls in Claude Code often dump large raw outputs into the LLM context, exhausting the context window. Context Mode proposes inserting an MCP server that routes tool execution through an isolated subprocess and only returns minimal stdout extracts, plus optional SQLite FTS5/BM25 indexing to retrieve exact blocks without pasting raw artifacts. This ADR adds this pattern to our architecture lab as an explicit output-sandboxing technique to preserve context and extend session duration.

## Decision

Add Context-Mode Output Sandbox (MCP)

## Consequences

This pattern/content has been added to the architecture lab repository.