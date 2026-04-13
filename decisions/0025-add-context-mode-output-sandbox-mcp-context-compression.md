# ADR-0025: Add Context Mode Output Sandbox (MCP Context Compression)

**Date**: 2026-04-13
**Status**: Accepted
**Pipeline**: pattern-lab

## Context

Source describes 'Context Mode' as an MCP server that sits between Claude Code and tool outputs, keeping raw artifacts out of the LLM context by routing execution to an isolated subprocess and returning only compact stdout/results. It also indexes fetched content into SQLite FTS5 with BM25, chunked by headings while preserving code blocks. Reported reductions: 315 KB → 5.4 KB (98%) and longer effective session duration (~30 min → ~3 hours).

## Decision

Add Context Mode Output Sandbox (MCP Context Compression)

## Consequences

This pattern/content has been added to the architecture lab repository.