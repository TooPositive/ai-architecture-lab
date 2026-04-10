# ADR-0022: Add Intent-conditioned Context Gateway (proxy-level context compaction)

**Date**: 2026-04-10
**Status**: Accepted
**Pipeline**: pattern-lab

## Context

Compresr’s Context Gateway pattern proposes inserting a transparent proxy between an agent and the LLM API that (1) monitors token usage, (2) pre-computes conversation compaction in the background at a threshold (docs show 75% default), and (3) compresses large tool outputs conditioned on the tool-call intent, with logs for observability. Sources: https://compresr.ai/docs/gateway and https://news.ycombinator.com/item?id=47367526

## Decision

Add Intent-conditioned Context Gateway (proxy-level context compaction)

## Consequences

This pattern/content has been added to the architecture lab repository.