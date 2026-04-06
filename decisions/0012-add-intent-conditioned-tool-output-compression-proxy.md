# ADR-0012: Add Intent-Conditioned Tool Output Compression Proxy

**Date**: 2026-04-06
**Status**: Accepted
**Pipeline**: pattern-lab

## Context

Sources describe Context Gateway as a transparent proxy between agents and LLM APIs that compresses tool outputs conditioned on tool-call intent, pre-computes background history compaction at a configurable context threshold (e.g., 75%), and supports expand() to recover original tool outputs. This addresses context-window exhaustion/latency and quality degradation in long agent sessions. Source: https://news.ycombinator.com/item?id=47367526 ; https://github.com/Compresr-ai/Context-Gateway ; https://compresr.ai/docs/gateway

## Decision

Add Intent-Conditioned Tool Output Compression Proxy

## Consequences

This pattern/content has been added to the architecture lab repository.