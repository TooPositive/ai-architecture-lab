# ADR-0027: Add Planner→Executor Trust Boundary

**Date**: 2026-04-13
**Status**: Accepted
**Pipeline**: pattern-lab

## Context

Source: RoboRhythms tutorial (Nathan Cole, 2026-04-03) describes replacing framework-style agent orchestration with a three-layer architecture: Planner LLM produces structured JSON, a deterministic trust-boundary validates via schema (e.g., Pydantic) + allowlist, then an Executor executes tools. Motivation: prevent silent failures and unsafe tool calls when LLM hallucinates tool/action names or arguments.

## Decision

Add Planner→Executor Trust Boundary

## Consequences

This pattern/content has been added to the architecture lab repository.