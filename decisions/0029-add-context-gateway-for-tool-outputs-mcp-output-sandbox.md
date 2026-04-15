# ADR-0029: Add Context Gateway for Tool Outputs (MCP Output Sandbox)

**Date**: 2026-04-15
**Status**: Accepted
**Pipeline**: pattern-lab

## Context

Source: “Stop Burning Your Context Window — We Built Context Mode” (https://mksg.lu/blog/context-mode). The article describes an MCP server that sits between an agent and tool outputs so raw tool results never enter the LLM context window; instead, tool interactions run in an isolated sandbox subprocess and only small stdout/derived artifacts are returned. It also describes URL fetch+index into SQLite FTS5 and BM25 retrieval, claiming ~98% output reduction (315 KB→5.4 KB) and longer sessions before slowdown.

## Decision

Add Context Gateway for Tool Outputs (MCP Output Sandbox)

## Consequences

This pattern/content has been added to the architecture lab repository.