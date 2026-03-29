# ADR-002: Stateless Goal Iterations

## Status
Accepted

## Date
2026-02-28

## Context
AgentHub's goal loop originally accumulated conversation history across iterations. A goal would start, run an iteration, keep the full chat history, and continue. This caused spiral failures: context grew with each iteration, the agent lost focus, and goals regularly hit 40+ iterations burning $50+ with no useful output. The daily cost averaged $42 with near-zero completion rate.

## Decision
Make each iteration stateless. The goal loop engine runs exactly one iteration per call: loads the goal and current plan from Cosmos DB, makes a single LLM call with tools, persists state changes, and returns the goal to the queue. No conversation history carries over. The plan document serves as the memory mechanism — the agent updates it with progress notes.

## Alternatives Considered
1. **Sliding window history** — Keep the last N messages instead of full history. Rejected: N is hard to tune, and the fundamental problem is context accumulation, not context size.
2. **Summarized history** — Compress previous iterations into a summary. Rejected: adds an LLM call per iteration for summarization, and the summary can lose critical details.
3. **Conversation branching** — Fork the conversation at decision points. Rejected: significantly more complex, and we couldn't identify decision points reliably.

## Consequences
### Positive
- Goal success rate went from ~0% to ~80%
- Average cost per goal dropped from $8-50 to $1-5
- Crash recovery is automatic (goals re-queue with persisted state)
- Iterations are independently debuggable

### Negative
- Cannot do multi-step reasoning within a single goal
- Each iteration pays full prompt cost (no KV cache reuse across iterations)
- Plan document must be well-structured or the agent loses context
