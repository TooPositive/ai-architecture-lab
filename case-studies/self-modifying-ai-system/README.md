# Case Study: Self-Modifying AI System

**System**: AgentHub (125K lines C#, .NET 10, Semantic Kernel)
**Date**: March 28, 2026
**Event**: The daemon autonomously read an article, proposed an improvement, wrote 7 C# files with tests, built, tested, merged to main, and redeployed itself.

---

## Context

AgentHub is a personal AI agent platform running 24/7 on an Azure VM. It executes goals autonomously through a stateless-iteration-worker loop: poll Cosmos DB for queued goals, run one iteration, persist state, return. A separate Evolution Engine watches execution traces and proposes improvements.

Before this session, the system had a serious problem: **it was spending $42/day with near-zero useful output.** Goals would spiral into sub-goal chains, the planner would repeat failed approaches, and the code worker spent 10 reads for every 1 write.

### Before vs. After

| Metric | Before | After |
|---|---|---|
| Daily cost | $42/day (zero output) | $5-10/day (real output) |
| Goal completion rate | ~0% (spirals) | ~80% (1-3 iterations) |
| Code worker read:write ratio | 10:1 | 2.8:1 |
| Time to first code write | ~40 min | ~30 sec |
| Self-modifications deployed | 0 ever | 1 (GoalGraph + PolicyGate) |

---

## Architecture of Self-Modification

The self-modification pipeline has four stages, each with independent safety checks.

```
                    EvolutionEngine
                    (trace analysis)
                          |
                    reflects on execution traces,
                    proposes CodeChange improvements
                          |
                          v
                SelfModificationEngine
                (orchestration)
                          |
            +-------------+-------------+
            |             |             |
       PlanChanges   CodeTaskWorker  BuildRunner
       (LLM plan)   (LLM writes     (dotnet build +
                      actual code)    dotnet test)
                          |
                          v
                  Independent Evaluator
                  (separate LLM reviews diff)
                          |
                          v
                  SelfModSafetyGate
                  (blast radius + security scan)
                          |
                          v
                  PostCompletionDeployService
                  (merge, deploy, validate)
```

### Stage 1: Evolution Engine Proposes

The `EvolutionEngine` runs after goal execution completes. It collects the execution trace (tokens, cost, tool calls, errors) and asks a reflection agent to identify improvement opportunities. If the reflection produces a `CodeChange` proposal with confidence above 0.5, the engine auto-spawns a `DaemonExtension` goal to implement it.

Rate limit: maximum 1 self-mod goal per day. Proposals below $0.10 in execution cost are skipped entirely (not worth the reflection cost).

### Stage 2: SelfModificationEngine Orchestrates

The `SelfModificationEngine` receives the modification request and:

1. Checks the config kill-switch (`Daemon:AllowSelfModification`)
2. Generates a plan via LLM, classifying risk level
3. Blocks changes to critical paths:
   - `Program.cs` (both Functions and Daemon entry points)
   - `PromptSecurity.cs`, `SecretScrubber.cs` (security infrastructure)
   - The `SelfMod/` directory itself (no self-modifying the self-modifier)
   - `.github/`, `deploy/`, `*.csproj` (build/deploy infrastructure)
4. Routes changes to core abstractions (`Abstractions/`, `Extensions/`, `Infrastructure/`) through PR review instead of direct merge
5. Hands execution to the `CodeTaskWorker` with the plan

### Stage 3: Safety Gate

After code is written and tests pass, the `SelfModSafetyGate` runs two deterministic checks:

**Gate 1 -- Blast Radius**: Counts files changed via `git diff --stat`. Default threshold: 5 files. If exceeded, the goal is suspended and a Discord alert is sent for human approval. This is configurable (`SelfMod:MaxFilesAutoApprove`).

**Gate 2 -- Security Scan**: Splits the diff into per-file chunks, scans each `.cs` file for security patterns (injection risks, hardcoded secrets, missing parameterization). Any `critical` or `high` finding blocks the merge without human override.

### Stage 4: Deploy and Validate

`PostCompletionDeployService` merges the branch, deploys to the VM, and (introduced in this session) spawns a delayed validation goal 2 hours later to check for regressions. Deploy has a 1-hour force-timeout to prevent stuck deployments.

---

## What the System Built: GoalGraph + PolicyGate

On March 28, the daemon read an article about runtime observability patterns and proposed adding two components to its own runtime layer.

### GoalGraph

A directed graph tracking goal node lifecycle. Each node has a state (`Pending`, `Running`, `Completed`, etc.), creation/update timestamps, and iteration/step counters. All transitions are recorded in an audit log.

```csharp
public sealed class GoalGraph
{
    private readonly Dictionary<string, GoalNode> _nodes = new(StringComparer.OrdinalIgnoreCase);
    private readonly List<GoalEvent> _auditLog = new();

    public GoalNode AddNode(string nodeId, string name, DateTimeOffset createdAt)
    {
        if (_nodes.TryGetValue(nodeId, out var existing))
            return existing;

        var node = new GoalNode(nodeId, name, GoalNodeState.Pending, createdAt, createdAt, 0, 0);
        _nodes[nodeId] = node;
        _auditLog.Add(new GoalEvent(nodeId, "NodeAdded", null, GoalNodeState.Pending, null, null, createdAt));
        return node;
    }

    public GoalNode Transition(
        string nodeId, GoalNodeState nextState, string eventType,
        string? toolName, string? detail, DateTimeOffset occurredAt)
    {
        if (!_nodes.TryGetValue(nodeId, out var node))
            throw new InvalidOperationException($"Node '{nodeId}' was not found.");

        var updated = node with { State = nextState, UpdatedAt = occurredAt };
        _nodes[nodeId] = updated;
        _auditLog.Add(new GoalEvent(nodeId, eventType, node.State, nextState, toolName, detail, occurredAt));
        return updated;
    }
}
```

### PolicyGate

Enforces runtime constraints on goal execution: tool deny-lists (per-node or global), iteration caps, step caps, and duration limits. All configurable via `PolicyGateOptions`.

```csharp
public sealed class PolicyGate
{
    private readonly PolicyGateOptions _options;

    public bool CanInvokeTool(string nodeName, string toolName)
    {
        if (_options.ToolDenyList.TryGetValue(nodeName, out var deniedTools))
            if (deniedTools.Contains(toolName)) return false;

        if (_options.ToolDenyList.TryGetValue("*", out var globalDenied))
            return !globalDenied.Contains(toolName);

        return true;
    }

    public bool CanContinue(GoalNode node)
    {
        if (node.IterationCount >= _options.DefaultMaxIterations) return false;
        if (node.StepCount >= _options.DefaultMaxSteps) return false;
        return (DateTimeOffset.UtcNow - node.CreatedAt) <= _options.DefaultMaxDuration;
    }
}
```

The daemon wrote both files, their supporting types (`GoalNode`, `GoalEvent`, `GoalNodeState`, `PolicyGateOptions`), and corresponding unit tests. It then built, ran tests, merged to main, and redeployed. The daemon restarted with its own code and continued operating normally.

---

## What Went Wrong First

The self-modification capability was not the first thing built. Before the March 28 session, significant debugging was required.

### Problem 1: Goal Spirals

Goals would spawn sub-goals, which spawned sub-sub-goals, burning budget with no useful work. The fix was explicit obstacle classification: `CapabilityGap` spawns a `DaemonExtension` sub-goal, `InformationGap` spawns a `Research` sub-goal, and `InfrastructureObstacle` (missing tools, permissions, network issues) logs and escalates instead of spawning.

### Problem 2: Planner Groundhog Day

The planner would retry failed approaches without learning. Fix: Jaccard similarity dedup on failure messages. If a new plan's failure message is >80% similar to a previous failure, it skips retrying and escalates.

### Problem 3: Code Worker Read Addiction

The code worker spent 40 minutes reading files before writing anything (10:1 read:write ratio). Fix: inject a codebase map into the system prompt so the model knows file locations upfront, plus a stricter read budget.

### Problem 4: No Evaluator Independence

The same model that wrote the code also evaluated it. Fix: independent evaluator gate -- a separate LLM call reviews the diff before merge approval.

All four fixes were prerequisites. Without them, the self-modification loop would have spiraled and burned budget before producing anything useful.

---

## Comparison: Self-Modification Approaches

| Dimension | Self-Mod (this system) | Manual Review + Deploy | CI/CD Only | Feature Flags |
|---|---|---|---|---|
| Time to deploy | Minutes (autonomous) | Hours to days | Minutes (after merge) | Seconds (toggle) |
| Human review required | Only above blast radius threshold | Always | PR review | Usually none |
| Can introduce novel code | Yes | Yes | No (only merges existing) | No |
| Safety guarantees | Deterministic gates + LLM eval | Human judgment | Tests + linting | Gradual rollout |
| Rollback speed | 2h validation goal | Manual | Git revert | Toggle off |
| Risk of subtle bugs | Higher (LLM-generated code) | Lower (human review) | Low | N/A |
| Scope of changes | Limited by BlockedPaths + blast radius | Unlimited | Unlimited | Config only |
| Cost | LLM tokens for plan + write + eval | Developer time | CI compute | Minimal |

**Key tradeoff**: Self-modification trades human review time for increased risk of subtle bugs. The safety gates (blast radius, security scan, independent evaluator, blocked paths) are designed to keep the risk envelope small, but they are not a substitute for human judgment on complex architectural changes.

---

## Metrics Infrastructure

Self-modification metrics are tracked via `SelfModMetricsService`, persisted to Cosmos DB with a 7-day TTL.

Tracked per goal:
- Success/failure
- Duration (ms)
- Cost (USD)
- Build attempts
- Files changed

Aggregated on demand:
- Success rate
- Average and P95 duration
- Total cost
- Average build attempts

This data feeds back into the Evolution Engine's confidence scoring: if recent self-mods have a low success rate, the confidence threshold for auto-spawning increases.

---

## Open Questions

These are known gaps, not theoretical concerns. Each emerged from operating the system.

### Necessity Gate

The system can propose improvements that are technically sound but unnecessary. An article about graph databases might trigger a proposal to add graph traversal, even though the existing dictionary-based GoalGraph is sufficient. A necessity evaluator ("Is this actually needed? Does it duplicate existing capability? What is the ROI?") would reduce low-value churn.

### Security Depth

The current security scan is regex-based pattern matching on diffs. It catches obvious issues (string interpolation in queries, hardcoded keys) but misses subtle ones (logic bugs that expose data through indirect paths, TOCTOU races). A more thorough approach would involve sandboxed execution of the generated tests with fault injection.

### Long-Term Drift

If the system runs for months, small changes accumulate. Each individual change passes safety checks, but the aggregate effect could drift the architecture in an unintended direction. No mechanism currently tracks architectural intent vs. actual trajectory.

### Human-in-the-Loop Calibration

The blast radius threshold (default: 5 files) was set by gut feeling. Too low and legitimate multi-file changes get blocked constantly; too high and risky changes slip through. The optimal threshold depends on the maturity of the other safety gates and should be calibrated over time.

### Evaluator Gaming

The independent evaluator is a separate LLM call, but it uses the same model family. In theory, systematic biases in the model could cause the evaluator to approve changes that a human would reject. Diversity in evaluation (different model providers, static analysis tools, property-based testing) would reduce this risk.

---

## Key Takeaways

1. **Self-modification worked, but only after fixing the execution engine first.** The $42/day spiral problem had to be solved before self-mod was even attempted. The capability is a capstone, not a starting point.

2. **Blocked paths and blast radius limits are essential.** The system cannot modify its own safety infrastructure, entry points, or build pipeline. This is a hard constraint, not a tunable parameter.

3. **Independent evaluation matters.** Having the same model write and evaluate code is a rubber-stamp. A separate LLM call reviewing the diff caught issues the writing model missed.

4. **Metrics close the loop.** Without `SelfModMetricsService` tracking success rates and costs, there is no feedback mechanism to tell the Evolution Engine whether its proposals are actually improving the system.

5. **The honest risk: subtle bugs at scale.** One successful self-modification does not prove the system is safe for continuous autonomous operation. The open questions above represent real operational risks that need resolution before increasing autonomy.
