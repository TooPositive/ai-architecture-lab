# ADR-0035: Add Context Gateway for MCP Tool Outputs (Context Mode)

**Date**: 2026-04-20
**Status**: Accepted
**Pipeline**: pattern-lab

## Context

Source: https://mksg.lu/blog/context-mode. The article describes a failure mode in MCP-based tool use where large tool outputs (e.g., Playwright snapshots, GitHub issue lists, logs) are appended directly into the LLM context, rapidly consuming the context window and increasing cost/latency. It proposes an MCP server (“Context Mode”) that sits between the client and tool outputs and compresses/abstracts tool results before they reach the model, reporting 315KB→5.4KB (~98%) reduction.

## Decision

Add Context Gateway for MCP Tool Outputs (Context Mode)

## Consequences

This pattern/content has been added to the architecture lab repository.