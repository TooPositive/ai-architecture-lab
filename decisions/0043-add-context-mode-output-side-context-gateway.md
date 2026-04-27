# ADR-0043: Add Context Mode (Output-side Context Gateway)

**Date**: 2026-04-27
**Status**: Accepted
**Pipeline**: pattern-lab

## Context

Source article reports an MCP server (“Context Mode”) that sits between Claude Code and tool outputs to prevent raw tool output blobs from entering the LLM context window. It routes tool calls through a sandbox subprocess and returns only compact stdout artifacts; additionally, it can fetch+index content into SQLite FTS5 and retrieve exact snippets via BM25. Reported reduction: 315 KB of raw output → 5.4 KB (98%) in tested sessions. URL: https://mksg.lu/blog/context-mode

## Decision

Add Context Mode (Output-side Context Gateway)

## Consequences

This pattern/content has been added to the architecture lab repository.