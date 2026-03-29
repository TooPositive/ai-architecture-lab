# Evolution Engine

> Self-improvement cycle that reflects on execution traces, generates improvement proposals, and auto-applies low-risk changes while gating high-risk ones.

## Problem
AI systems degrade over time as the world changes but their prompts, skills, and code stay static. Manual improvement is slow and doesn't scale. But fully autonomous self-modification is dangerous — bad changes can cascade.

## Solution
After every goal execution, record an execution trace (what happened, what tools were called, what worked, what failed). A reflection step analyzes these traces and generates structured improvement proposals in four categories:

1. **Skills** (auto-apply at confidence >= 0.5) — reusable knowledge snippets
2. **Prompts** (require review) — changes to system prompts
3. **Code** (require evaluator gate + confidence >= 0.7) — implementation changes
4. **Strategy** (require human approval) — routing or prioritization changes

Low-risk proposals (skills) are applied immediately. High-risk proposals (code changes) spawn a separate goal that implements the change in an isolated worktree, runs tests, and submits for independent review.

## Architecture

See [diagram.mmd](diagram.mmd)

## Trade-offs

| Pro | Con |
|-----|-----|
| System improves without human intervention | Can diverge from intended behavior over time |
| Tiered risk gates match risk to review level | Auto-applied skills may accumulate noise |
| Execution traces provide audit trail | Reflection step adds cost to every goal |
| Proposals are structured and reviewable | Threshold tuning is an art, not a science |

## When to Use
- Long-running autonomous systems (weeks/months)
- Systems where you want measurable improvement over time
- When you have execution traces to learn from

## When NOT to Use
- Short-lived or stateless applications
- Systems where behavior must be fully deterministic
- When you can't afford the per-execution reflection cost

## Implementation Notes
**Confidence thresholds matter.** Started at 0.8 for everything — too conservative, 500+ proposals accumulated without being applied. Lowered skills to 0.5 (cheap to revert if wrong). Kept code at 0.7 with evaluator gate (expensive to revert if wrong).

**Deprecation sweep:** Periodically scan for skills that haven't been used in 14 days and deprecate them. Prevents skill bloat.

**Rate limiting:** Maximum 1 self-modification goal per day. Without this, the system would propose and implement changes faster than humans can review.

**Batch analysis:** Weekly analysis across 100+ traces finds patterns that single-trace reflection misses ("goals of type X always fail at step 3").

## Benchmarks
Benchmarks pending — tracking improvement rate over 90 days.

## Related Patterns
- [Evaluator Gate](../evaluator-gate-for-self-modification/) — independent review for code changes
- [Autonomous Goal Loop](../autonomous-goal-loop/) — how self-mod goals execute
- [Budget-Governed Execution](../budget-governed-agent-execution/) — cost control for reflection

## Sources
- "Reflexion: Language Agents with Verbal Reinforcement Learning" (Shinn et al., 2023)
- Internal AgentHub evolution engine design
