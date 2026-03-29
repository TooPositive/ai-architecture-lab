# ADR-007: Declarative Trigger Rules Over Hardcoded Event Handlers

## Status
Accepted

## Date
2026-02-10

## Context
AgentHub's early pipeline wiring was hardcoded: when the daily collector finished, it called a specific method to spawn the insight synthesis goal. When the weekly digest finished, another hardcoded call spawned the content factory. Adding a new pipeline meant modifying code in multiple places, rebuilding, and redeploying.

## Decision
Replace hardcoded event handlers with declarative trigger rules. Each rule specifies:
- **ObservationType**: The event to match (e.g., `pipeline-completed:daily-collector`)
- **Condition**: Optional filters (min count, min confidence, LLM evaluation)
- **Action**: What to do when triggered (e.g., `create-goal` with a specific objective)
- **Cooldown**: Minimum time between firings (e.g., 20 hours for daily triggers)

Rules are loaded from `TriggerRuleLoader.GetDefaultRules()` (code-defined defaults) and can be extended via JSON config file.

## Alternatives Considered
1. **Event bus with subscribers** — Use a message broker (Azure Service Bus) with subscriber registration. Rejected: adds infrastructure dependency, and the daemon already runs in-process.
2. **Plugin system** — Load pipeline handlers from DLL plugins. Rejected: over-engineered for the current scale. Trigger rules are simpler.
3. **Cron-only scheduling** — Use scheduled agents for everything, no event-driven triggers. Rejected: some reactions need to happen immediately after a pipeline completes, not on a fixed schedule.

## Consequences
### Positive
- Adding new pipelines = adding a config entry (no code change for simple rules)
- Cooldowns prevent duplicate goal spawning
- Rules are inspectable (list all rules to understand system behavior)
- Easy to disable rules without removing code

### Negative
- Complex conditions still require code (LLM evaluation predicates)
- Rule debugging is harder than stepping through code
- Rule ordering/priority not yet implemented (could cause conflicts)
