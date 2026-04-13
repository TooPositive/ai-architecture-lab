# ADR-0026: Add Deterministic Executor Trust Boundary (Untrusted LLM → Validated Actions)

**Date**: 2026-04-13
**Status**: Accepted
**Pipeline**: pattern-lab

## Context

Source describes building an orchestration engine (RUX) to address common agent failure modes: lack of validation/reliability measurement and silent failures when LLMs hallucinate action names. The core architecture is to treat the LLM as untrusted and introduce a deterministic Executor as a trust boundary, with an Executor schema acting as the contract between probabilistic planning and deterministic tool/service execution.

## Decision

Add Deterministic Executor Trust Boundary (Untrusted LLM → Validated Actions)

## Consequences

This pattern/content has been added to the architecture lab repository.