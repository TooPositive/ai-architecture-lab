# ADR-0034: Add Untrusted LLM + Deterministic Executor Boundary

**Date**: 2026-04-17
**Status**: Accepted
**Pipeline**: pattern-lab

## Context

Source proposes addressing common agent reliability failures by treating LLM outputs as untrusted and inserting an explicit deterministic Executor as the trust boundary. The Executor validates/normalizes the plan into an allowed, well-formed tool invocation path (Planner → Executor → Tool → Service), preventing silent failures from hallucinated tool/action names.

## Decision

Add Untrusted LLM + Deterministic Executor Boundary

## Consequences

This pattern/content has been added to the architecture lab repository.