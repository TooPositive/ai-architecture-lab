# ADR-0040: Add Context Gateway MCP Sandbox (Context Mode)

**Date**: 2026-04-24
**Status**: Accepted
**Pipeline**: pattern-lab

## Context

Source: “Stop Burning Your Context Window — We Built Context Mode” (https://mksg.lu/blog/context-mode). The article describes an MCP server that sits between Claude Code and tool outputs, executing tools in an isolated subprocess sandbox and only returning small stdout summaries to the LLM. It also indexes markdown/URL content into SQLite FTS5 with BM25 so the agent can retrieve exact snippets without injecting raw data into context, reporting up to a 98% reduction in context bloat (315 KB → 5.4 KB) across scenarios.

## Decision

Add Context Gateway MCP Sandbox (Context Mode)

## Consequences

This pattern/content has been added to the architecture lab repository.