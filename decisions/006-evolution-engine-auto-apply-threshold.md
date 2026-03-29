# ADR-006: Evolution Engine Auto-Apply Threshold

## Status
Accepted

## Date
2026-03-20

## Context
AgentHub's evolution engine generates improvement proposals after each goal execution. Initially, all proposals required a confidence score of 0.8 or higher to be auto-applied. At this threshold, proposals accumulated — over 500 sat in "Proposed" status because the reflection step rarely assigned confidence above 0.75. The system was generating improvements but never applying them.

## Decision
Use tiered confidence thresholds based on the risk profile of each proposal type:
- **Skills**: Auto-apply at confidence >= 0.5 (cheap to create, cheap to revert, low blast radius)
- **Prompts**: Always require human review (prompt changes can subtly alter all agent behavior)
- **Code changes**: Require confidence >= 0.7 AND independent evaluator gate (expensive to revert, high blast radius)
- **Strategy/routing**: Always require human approval (affects system-wide prioritization)

## Alternatives Considered
1. **Uniform 0.8 threshold for all types** — Original approach. Rejected: 500+ proposals accumulated, system never improved.
2. **Auto-apply everything** — Remove gates entirely, apply all proposals. Rejected: a bad prompt change affects every agent call. The blast radius is too large.
3. **Time-based decay** — Lower threshold over time if proposal isn't applied. Rejected: age doesn't make a proposal better. A bad idea at 0.6 confidence is still bad after 14 days.

## Consequences
### Positive
- Skills are now actively applied (system actually improves)
- Code changes go through evaluator gate (quality maintained)
- Proposal backlog cleared from 500+ to near-zero

### Negative
- Auto-applied skills at 0.5 confidence means ~20% are low quality (cleaned up by deprecation sweep)
- Different thresholds for different types adds complexity
- Stale proposal expiration (14 days) means some good proposals are lost if not reviewed in time
