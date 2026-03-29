# Budget-Governed Agent Execution

> Hierarchical budget limits (daily, weekly, per-goal, per-operation) with automatic pause and resume for autonomous AI agents.

## Problem
Autonomous agents with access to paid APIs can burn through budgets in minutes. A single runaway goal — stuck in a loop, spawning sub-tasks, or generating enormous prompts — can exhaust a daily budget before other goals get a chance to run.

## Solution
Implement a hierarchical budget system where limits cascade from broad to narrow:

1. **Daily limit** ($50) — total system spend per day
2. **Weekly limit** ($300) — prevents one bad week from exhausting monthly budget
3. **Proactive limit** (70% of daily) — EA-initiated work vs reactive work
4. **Per-goal limit** ($8-25, varies by type) — caps individual goal spend
5. **Per-operation limit** ($25) — caps any single LLM call

When any limit is hit, the affected scope pauses automatically. Daily limit exhausted? System pauses until midnight. Per-goal limit hit? Goal is paused and re-queued for next budget cycle. Other goals continue unaffected.

## Architecture

See [diagram.mmd](diagram.mmd)

## Trade-offs

| Pro | Con |
|-----|-----|
| Prevents runaway costs (hard ceiling) | Can starve important long-running tasks |
| Predictable monthly spend | Complex to tune (too tight = nothing runs, too loose = waste) |
| Per-goal isolation (one runaway doesn't kill others) | Budget checks add latency to every LLM call |
| Automatic recovery (paused goals resume next cycle) | Emergency override needed for critical tasks |

## When to Use
- Any autonomous system making paid API calls
- Multi-agent systems where goals compete for resources
- Systems running 24/7 without constant human monitoring

## When NOT to Use
- Single-shot scripts (just set a total budget)
- Free-tier-only systems
- Systems with human-in-the-loop for every API call

## Implementation Notes
**Alert thresholds:** Send Discord alerts at 50%, 25%, and 10% remaining daily budget. The 10% alert is critical — it means only $5 left and something might be wrong.

**Emergency stop:** `ForceStop()` kills all active goals immediately. Wired to a Discord command for human override.

**Budget by goal type:**
| Goal Type | Budget Cap | Rationale |
|-----------|-----------|-----------|
| Research | $8 | Knowledge gathering, bounded scope |
| Content | $8 | Draft generation, single output |
| DaemonExtension | $25 | Code generation, needs more room |
| Utility | $5 | Maintenance tasks |

**Cooldown on re-queue:** When a goal is paused for budget, it waits 5 minutes before re-queuing. This prevents rapid re-queue/re-pause cycles.

**Post-crypto-epic learning:** Budget hierarchy was added after a single research epic about cryptocurrency burned $200 in one day. Per-type caps specifically prevent this class of failure.

## Benchmarks
| Metric | Before Budget System | After Budget System |
|--------|---------------------|---------------------|
| Daily spend (avg) | $42 (unpredictable) | $5-15 (controlled) |
| Peak daily spend | $71 | $50 (hard cap) |
| Runaway goals | ~30% of all goals | 0% (paused at cap) |
| Monthly cost | ~$1,260 | ~$300-450 |

## Related Patterns
- [Autonomous Goal Loop](../autonomous-goal-loop/) — budget checks happen at iteration boundaries

## Sources
- Internal AgentHub cost analysis (March 2026)
