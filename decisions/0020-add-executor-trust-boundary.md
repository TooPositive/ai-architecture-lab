# ADR-0020: Add Executor Trust Boundary

**Date**: 2026-04-10
**Status**: Accepted
**Pipeline**: pattern-lab

## Context

Source describes the RUX orchestration engine approach: treat the LLM as untrusted and introduce an Executor trust boundary that deterministically validates tool plans against schemas/allowlists before executing side effects (Planner -> Executor -> Tool -> Service).

## Decision

Add Executor Trust Boundary

## Consequences

This pattern/content has been added to the architecture lab repository.