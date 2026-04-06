# ADR-0015: Add MCP output sandbox for context compression (Context Mode)

**Date**: 2026-04-06
**Status**: Accepted
**Pipeline**: pattern-lab

## Context

Source describes an MCP server (“Context Mode”) that sits between Claude Code and tool outputs so raw tool payloads (snapshots/logs/issues/HTML) never enter the LLM context. It uses a PreToolUse hook to route tool calls through an isolated subprocess sandbox and returns only compact stdout/structured output. For doc retrieval it chunks markdown by headings (keeping code blocks intact), indexes in SQLite FTS5, and returns BM25-ranked exact snippets; claimed 315KB→5.4KB (98% reduction) and longer effective sessions.

## Decision

Add MCP output sandbox for context compression (Context Mode)

## Consequences

This pattern/content has been added to the architecture lab repository.