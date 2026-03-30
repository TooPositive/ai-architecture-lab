# ADR-0001: Add MCP Context Sandbox Output Gating

**Date**: 2026-03-30
**Status**: Accepted
**Pipeline**: pattern-lab

## Context

Source: “Stop Burning Your Context Window — We Built Context Mode” (https://mksg.lu/blog/context-mode). The article describes an MCP server that sits between Claude Code and tool outputs, executing data-heavy operations in an isolated sandbox and returning only small stdout extracts/snippets to the LLM context. It reports large context reductions (e.g., 315KB→5.4KB; 98% reduction) and longer effective sessions by preventing raw tool outputs (Playwright snapshots, logs, issue lists) from entering the context window.

## Decision

Add MCP Context Sandbox Output Gating

## Consequences

This pattern/content has been added to the architecture lab repository.