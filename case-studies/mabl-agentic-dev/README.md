# mabl — System for AI agents to ship real code across 75+ repos
Source: https://www.mabl.com/blog/how-we-built-a-system-for-ai-agents-to-ship-real-code-across-75-repos

## Company / project
- **Company:** mabl
- **Project:** Internal platform + workflows enabling agentic development across many repos (“AI agents to ship real code across 75+ repos”) — Part 1 of 2
- **Article:** “How We Built a System for AI Agents to Ship Real Code Across 75+ Repos” (Apr 8, 2026)

## Problem they solved
Scaling “agentic coding” beyond individual developers to an organization with many repositories and frequent releases.

They report:
- **25 engineers managing 100+ repositories**
- **200+ PRs and 40+ production releases per month** (pre-acceleration)

Key scaling challenges:
- Cross-repo coordination (dependency awareness + merge ordering).
- Grounding agents in the same real-world validation activities humans do (tickets, UI verification, tests).
- Avoiding human review becoming the bottleneck as PR volume increases.

## Architecture choices & trade-offs (as described)
They describe an evolved, layered architecture:

### Layer 1 — “Cross Repo Base” (rules + dependency graphs)
- A centralized repository (“pseudo mono-repo above all repos”) containing:
  - Engineering guides, best practices, design patterns, workflows, cloud/internal tooling rules.
- Each repo maintains a **CLAUDE.md** with repo-specific constraints:
  - Purpose, architecture, dependencies, conventions, build commands, step-by-step workflows.
- A **Repo Coordination Graph** (reported **850+ lines**, covering **79 repos**) capturing:
  - Dependency graphs, Pub/Sub topic maps, DB table ownership, and prescribed release ordering.
- **Git worktrees** for parallel multi-repo work with hooks preventing edits in the wrong workspace.
- A “mandatory code grounding rule” requiring agents to verify paths and quote real code locations (no fabricated references).

**Trade-off:** building/maintaining this cross-repo context artifact is work, but reduces “tribal knowledge bottlenecks” and context drift.

### Layer 2 — Skill system (MCP servers + reusable “skills”)
- They integrated MCP servers and “close to **40 unique skills**,” including:
  - Atlassian MCP server (Jira/Confluence operations)
  - mabl Testing MCP server (trigger runs, parse results, manage environments)
  - Desktop Automation MCP server (Playwright-driven UI interactions + screenshots)
- Configured via workspace-level **.mcp.json** so developers and agents share identical tool access.
- Built **36+ reusable skills** invoked via slash commands (workflow, build, validation routing, planning, ops).
- A “validation router” automatically selects the right test suite based on file changes.

**Trade-off:** constraining tools/skills to what’s needed improves reliability vs “general purpose” tool sprawl, but requires ongoing curation.

### Layer 3 — Operations & governance
- **AI code review** workflow (“claude-pr-review”) that can REQUEST_CHANGES or COMMENT.
- **Auto-fix agents** for trivial CI failures (lint/import/type errors), explicitly avoiding complex fixes.
- Circuit breakers to avoid fix-fail loops (e.g., if last 2 commits tagged [auto-fix], stop; if author reverted an auto-fix, don’t retry same fix).
- PR ChatOps slash commands to trigger targeted pipelines.
- GitHub merge queue + Terraform-managed rulesets:
  - Org-level: **1 required human approval**, squash-only merges, etc.
  - Repo-level required checks, concurrency limits.

**Trade-off:** governance intentionally keeps humans in control (agents can’t approve merges), at the cost of some autonomy.

## Results & metrics (from the article)
- AI-assisted commits grew from **~10% (Aug 2025)** to **39% overall (Feb 2026)** and **60% in infrastructure repos**.
- PR throughput: **370+ PRs/month** in application repos by Feb 2026 (up from **230/month** in Aug 2025).

(Other performance metrics like defect rate or lead time are **not disclosed** in Part 1.)

## Comparison vs AgentHub (8 dimensions)
| Dimension | mabl system vs us | Notes (what the article actually says) |
|---|---|---|
| 1. Data Pipeline | **DIFFERENT** | Their “pipeline” is SDLC/DevOps oriented (tickets → code → tests → PR → merge), not content ingestion. |
| 2. Retrieval | **NOT DISCLOSED** | Not a RAG system; limited mention of Confluence/Jira querying via MCP, but no retrieval stack details (vector/BM25/etc.). |
| 3. Agents | **THEM AHEAD** | Org-scale agentic workflow spanning **75+ repos**, with state machine phases, worktrees, specialized agents (reviewer, auto-fix), and tool access parity with humans. |
| 4. Evaluation | **DIFFERENT** | They rely heavily on validation gates (test routing, UI screenshots) + AI code review; we rely on goal critic + quality scores for content/automation. |
| 5. Self-Improvement | **DIFFERENT** | They build reusable skills/rules as shared org memory (contribution model implied); we have an explicit evolution engine. No explicit automated self-mod loop described beyond auto-fix. |
| 6. Safety | **THEM AHEAD** | Clear governance: AI cannot approve merges; required human approvals; CI gates; circuit breakers on auto-fix; Terraform-managed rulesets. |
| 7. Cost | **NOT DISCLOSED** | No infra or LLM cost numbers disclosed; mentions token-efficiency indirectly (tooling helps), but no $ amounts. |
| 8. Deployment | **THEM AHEAD** | Deep integration with CI/CD, merge queues, per-repo checks, and production release cadence (40+ releases/month baseline). Our deployment is simpler (auto-deploy from git push). |

## Confidence
- **High** on the described layers and the quantitative repo/PR/commit metrics (explicit in the article).
- **Medium** on the exact number of skills/servers (captured text says “close to 40 unique skills” and “36+ reusable skills,” but full inventory isn’t reproduced here).
- **Low/Not disclosed** on their LLM models, prompt structure, and cost.
