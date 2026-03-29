# Evaluator Gate for Self-Modification

> Use an independent LLM call to review code diffs before self-modification merges. Separate the judge from the coder.

## Problem
When an LLM writes code and then evaluates its own code, it rubber-stamps. The same biases that produced the code also approve the code. Self-modification without independent review leads to quality degradation over time.

## Solution
After a self-modification goal produces code changes, a completely separate LLM call reviews the diff. This evaluator:
1. Receives only the `git diff` (not the full conversation or reasoning)
2. Reviews against 5 criteria: correctness, safety, test coverage, style, necessity
3. Returns APPROVE or REJECT with specific feedback
4. If REJECT, feedback is fed back to the coder for revision

The evaluator uses a fresh chat client — not the same session, not the same system prompt, not the same context. It sees raw code changes and judges them on merit.

## Architecture

See [diagram.mmd](diagram.mmd)

## Trade-offs

| Pro | Con |
|-----|-----|
| Catches real issues the coder missed | Adds latency (extra LLM call per self-mod) |
| Independent judgment (no rubber-stamping) | Adds cost (~$0.50-1.00 per review) |
| Structured feedback enables iterative improvement | Can be overly conservative (false rejections) |
| Audit trail of all reviews | Evaluator can also hallucinate approval |

## When to Use
- Any system that modifies its own code
- CI/CD pipelines with LLM-generated code
- Automated PR review bots

## When NOT to Use
- Manual code review is already in place (redundant)
- Changes are trivial (config-only, no logic)
- Cost of the extra LLM call exceeds the risk of bad code

## Implementation Notes
**Evaluator prompt is critical.** The prompt should be specific about what to look for: security vulnerabilities, breaking changes, test coverage gaps, style violations. Vague prompts ("is this code good?") lead to rubber-stamping.

**Infrastructure failure handling:** If the evaluator LLM call fails (timeout, rate limit), default to APPROVE — don't block deployments on infrastructure issues. Log the failure for manual follow-up.

**5 review criteria:**
1. **Correctness** — Does the code do what the objective says?
2. **Safety** — Any security risks? Injection, secret exposure, auth bypass?
3. **Test coverage** — Are new behaviors tested?
4. **Style** — Does it match existing codebase patterns?
5. **Necessity** — Is this change actually needed, or is it unnecessary complexity?

**Diff-only review:** The evaluator sees `git diff main...HEAD`, not the full codebase. This forces it to judge the changes on their own merit rather than getting lost in context.

## Benchmarks
| Metric | Without Gate | With Gate |
|--------|-------------|-----------|
| Bad merges caught | 0% (all auto-merged) | ~70% of problematic changes |
| False rejection rate | N/A | ~15% (overly conservative) |
| Cost per review | $0 | ~$0.50-1.00 |
| Time per review | 0s | 15-30s |

## Related Patterns
- [Evolution Engine](../evolution-engine/) — generates the proposals that this gate reviews

## Sources
- "Constitutional AI" concept (Anthropic) — self-critique patterns
- Internal AgentHub evaluator gate design
