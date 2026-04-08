# ADR-0017: Add Context-Mode MCP Output Compression Gateway

**Date**: 2026-04-08
**Status**: Accepted
**Pipeline**: pattern-lab

## Context

Source describes Context Mode: an MCP server that sits between Claude Code and tool outputs to prevent large tool payloads from consuming the context window. It stores raw outputs out-of-band and returns a compressed representation plus handles, claiming 315KB → 5.4KB (98% reduction).

## Decision

Add Context-Mode MCP Output Compression Gateway

## Consequences

This pattern/content has been added to the architecture lab repository.