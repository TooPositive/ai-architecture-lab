# ADR-0036: Add Executor Trust Boundary for Agent Orchestration

**Date**: 2026-04-22
**Status**: Accepted
**Pipeline**: pattern-lab

## Context

Source: RUX - AI Orchestration Engine (GitHub README) https://github.com/rahulT-17/RUX_AI_Orchestration_Engine . RUX argues most toy agents use LLM→tool→response with little validation, causing silent failures when the model hallucinates action/tool names or emits malformed JSON. RUX introduces an explicit Executor as a deterministic trust boundary: anything before is probabilistic; anything after strict schema validation is deterministic, auditable, and safe. The README also highlights schema validation with extra="forbid", rate limiting/auth at API boundary, and optional independent Critic model.

## Decision

Add Executor Trust Boundary for Agent Orchestration

## Consequences

This pattern/content has been added to the architecture lab repository.