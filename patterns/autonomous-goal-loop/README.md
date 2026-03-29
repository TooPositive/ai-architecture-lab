# Autonomous Goal Loop

> Stateless goal iteration where each cycle is independent — goal state persists to database, preventing spiral failures and enabling crash recovery.

## Problem
Autonomous AI agents that accumulate conversation history across iterations tend to spiral: each iteration adds context, the context grows, the agent loses focus, and costs explode. A 3-iteration task becomes 40 iterations burning $50+ with no useful output.

## Solution
Treat each iteration as a fresh, stateless operation. The goal loop engine:
1. Reads the goal and its current plan from the database
2. Runs exactly one iteration (one LLM call with tools)
3. Persists any state changes (plan updates, observations) back to the database
4. Returns the goal to the queue for the next iteration

No conversation history accumulates. Each iteration sees only: the original objective, the current plan, and the current state. If the system crashes mid-iteration, the goal simply re-queues on restart with its last persisted state.

## Architecture

See [diagram.mmd](diagram.mmd)

## Trade-offs

| Pro | Con |
|-----|-----|
| Crash-safe — any failure just re-queues | Cannot do multi-step reasoning within a single goal |
| Prevents context spiral (fixed context per iteration) | Each iteration pays full prompt cost (no history reuse) |
| Easy to debug (each iteration is self-contained) | Agent "forgets" between iterations unless plan is updated |
| Predictable cost per iteration | More iterations needed for complex tasks |

## When to Use
- Long-running autonomous agents (hours/days)
- Systems where crash recovery matters
- Budget-constrained execution (predictable per-iteration cost)
- Multi-goal systems where goals compete for resources

## When NOT to Use
- Interactive chat (users expect conversation memory)
- Tasks requiring deep multi-step reasoning in a single session
- One-shot tasks where a single LLM call suffices

## Implementation Notes
The key insight is persisting the **plan** as the memory mechanism. Instead of conversation history, the agent updates its plan after each iteration: "Step 1: Done. Step 2: In progress — found 3 articles, need to analyze." The next iteration reads this plan and picks up where it left off.

**Goal state machine:** `Queued -> Active -> (Completed | Failed | Blocked | Cancelled)`

Each iteration has a budget cap (typically $8 for research goals, $25 for code generation). If the cap is hit mid-iteration, the goal is paused and re-queued for the next budget cycle.

Max iterations per goal (typically 5-10) prevents infinite loops. If max is reached, the goal is marked `Failed` with the last plan state preserved for debugging.

## Benchmarks
| Metric | With History (before) | Stateless (after) |
|--------|----------------------|-------------------|
| Avg iterations per goal | 15-40 (spiral) | 1-3 |
| Cost per completed goal | $8-50 | $1-5 |
| Goal success rate | ~0% (spirals) | ~80% |
| Recovery after crash | Manual restart | Automatic re-queue |

## Related Patterns
- [Budget-Governed Execution](../budget-governed-agent-execution/) — budget checks at iteration boundaries
- [Evolution Engine](../evolution-engine/) — self-improvement goals use this loop

## Sources
- Internal learnings from AgentHub goal loop redesign (March 2026)
