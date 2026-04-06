# ADR-0013: Add Planner→Validator→Executor trust boundary for tool-using agents

**Date**: 2026-04-06
**Status**: Accepted
**Pipeline**: pattern-lab

## Context

Source (RoboRhythms, Apr 3 2026) argues most agent stacks lack validation and fail silently when LLM hallucinates tool/action names or selects unsafe actions; proposes treating LLM output as untrusted input and inserting a deterministic validation layer between Planner LLM and Executor. https://www.roborhythms.com/how-to-build-ai-orchestration-without-langchain-2026/

## Decision

Add Planner→Validator→Executor trust boundary for tool-using agents

## Consequences

This pattern/content has been added to the architecture lab repository.