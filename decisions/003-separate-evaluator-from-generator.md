# ADR-003: Separate Evaluator from Generator for Self-Modification

## Status
Accepted

## Date
2026-03-28

## Context
AgentHub's self-modification pipeline had the same LLM session that wrote code also evaluate whether the code should be merged. Early testing showed this consistently resulted in rubber-stamping — the model approved its own code ~95% of the time, including changes with obvious issues (missing error handling, unused imports, unnecessary complexity).

## Decision
Use a completely separate LLM call with a fresh chat client to review the `git diff` before merging self-modifications. The evaluator receives only the diff and a review prompt with 5 criteria (correctness, safety, test coverage, style, necessity). It has no access to the original conversation, reasoning, or objective that produced the code.

## Alternatives Considered
1. **Same model, different prompt** — Add a "now be critical" instruction to the existing session. Rejected: same context biases still apply, model is primed to agree with itself.
2. **Different model** — Use a cheaper model for evaluation. Rejected: cheaper models miss subtle issues. The cost of one bad merge exceeds the cost of evaluation.
3. **Rule-based checks only** — Static analysis (lint, build, test) without LLM review. Rejected: catches syntax issues but not architectural problems, unnecessary complexity, or security risks.

## Consequences
### Positive
- Catches ~70% of problematic changes that would have been auto-merged
- Creates an audit trail of all reviews (stored in Cosmos DB)
- Forces code changes to stand on their own merit (no context bias)

### Negative
- ~15% false rejection rate (evaluator is sometimes overly conservative)
- Adds $0.50-1.00 per review
- If evaluator LLM fails (timeout/rate limit), defaults to APPROVE to avoid blocking deployments
