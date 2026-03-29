# ADR-004: Two Cosmos DB Accounts for Privacy Isolation

## Status
Accepted

## Date
2025-11-01

## Context
AgentHub stores both personal data (financial records, health notes, family information) and agent-accessible data (articles, knowledge nodes, goals) in Cosmos DB. Autonomous agents run 24/7 and make LLM calls with data from the database. Any data an agent can read could end up in an LLM prompt, which means it could appear in logs, error messages, or even published content.

## Decision
Maintain two separate Cosmos DB accounts:
- **agenthub-personal**: Personal/sensitive data. Firewall restricted to Mac IP only. Accessible only via Claude Code (manual sessions). Contains exact financial data, health records, family information.
- **agenthub-cosmos**: Agent-accessible data. Open to Azure Functions + daemon VM. Contains articles, knowledge nodes, goals, content drafts. Personal data appears only as anonymized summaries (`ctx-agent-*` prefix) — e.g., "financial situation: stable" instead of exact numbers.

## Alternatives Considered
1. **Single account with RBAC** — Use Cosmos DB role-based access control to restrict agent access to specific containers. Rejected: RBAC is container-level, not document-level. An agent with access to a container sees everything in it.
2. **Encryption at field level** — Encrypt sensitive fields, give agents only the decryption key for non-sensitive fields. Rejected: adds significant complexity, and any bug in key management exposes everything.
3. **Separate containers in same account** — Put personal data in separate containers with stricter access. Rejected: firewall rules are account-level, not container-level. Can't restrict one container without restricting all.

## Consequences
### Positive
- Agents physically cannot access personal data (network-level isolation)
- Even if an agent is compromised, attack surface is limited to non-sensitive data
- Clear mental model: personal account = human only, agent account = agents + human

### Negative
- Two accounts to manage (twice the config, twice the monitoring)
- Anonymized summaries must be manually maintained (they can go stale)
- Cross-account queries are impossible (can't join personal + agent data)
