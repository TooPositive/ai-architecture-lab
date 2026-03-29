# ADR-005: Budget Hierarchy for Autonomous Agents

## Status
Accepted

## Date
2026-03-15

## Context
Without budget controls, a single AgentHub goal burned $200 in one day on cryptocurrency research. The Executive Assistant kept spawning sub-goals, each making expensive LLM calls, with no mechanism to stop the spending. Daily costs were unpredictable — ranging from $5 to $71 with no correlation to useful output.

## Decision
Implement a hierarchical budget system:
- **Daily cap**: $50 (hard ceiling, system pauses at midnight reset)
- **Weekly cap**: $300 (prevents one bad week from destroying monthly budget)
- **Proactive cap**: 70% of daily ($35) for EA-initiated work
- **Per-goal cap**: $8 (research/content), $25 (code generation/DaemonExtension)
- **Per-operation cap**: $25 (no single LLM call can exceed this)
- **Alerts**: Discord notifications at 50%, 25%, and 10% remaining daily budget

## Alternatives Considered
1. **Single daily cap only** — Just limit total daily spend. Rejected: doesn't prevent one goal from consuming the entire budget while others starve.
2. **Token-based budgeting** — Count tokens instead of dollars. Rejected: token costs vary by model, and dollar amounts are what actually matter for financial planning.
3. **Manual approval per goal** — Human approves each goal's budget before execution. Rejected: defeats the purpose of autonomous 24/7 operation.

## Consequences
### Positive
- Monthly costs dropped from ~$1,260 to ~$300-450
- No more runaway goals (paused automatically at cap)
- Predictable spending enables financial planning
- Other goals continue even when one is paused

### Negative
- Important long-running tasks can be starved by their per-goal cap
- Budget tuning is ongoing (too tight = nothing runs, too loose = waste)
- Emergency override mechanism needed for critical tasks
