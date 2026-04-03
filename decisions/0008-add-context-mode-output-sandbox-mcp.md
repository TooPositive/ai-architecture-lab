# ADR-0008: Add Context Mode Output Sandbox (MCP)

**Date**: 2026-04-03
**Status**: Accepted
**Pipeline**: pattern-lab

## Context

Source: “Stop Burning Your Context Window — We Built Context Mode” (https://mksg.lu/blog/context-mode). The article describes an MCP server that routes tool outputs through a sandboxed subprocess so raw outputs never enter the LLM context, claiming 98% reduction (315 KB → 5.4 KB) and longer usable sessions. Decision: add this pattern to pattern-lab as an architecture technique for context window optimization via output offloading + local FTS index.

## Decision

Add Context Mode Output Sandbox (MCP)

## Consequences

This pattern/content has been added to the architecture lab repository.