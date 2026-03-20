# CLAUDE.md — Doot Agents

> **Purpose:** Project context for Claude Code. Read at the start of every session.
> **Last updated:** 2026-03-20

---

## What Is This

**Doot Agents** — an autonomous agent system that runs operations for Cloud Saathi, a fractional DevOps startup by Nihaan Mohammed and Ankit Jangir.

10 specialized agents coordinate through a Slack Orchestrator (hub-and-spoke). All agents start in "Draft + Approve" mode and graduate to full autonomy via a trust-scoring system.

## Founders

- **Nihaan Mohammed** — Group PM, non-technical. Handles product, strategy, GTM.
- **Ankit Jangir** — Co-founder, technical. Handles implementation.

## Stack

| Layer | Choice |
|---|---|
| Hosting | Hetzner CX23 (2 vCPU, 4GB RAM) |
| Orchestration | n8n self-hosted + cron |
| Agent Framework | OpenAI Agents SDK / raw Python |
| State | SQLite + Upstash Redis free tier |
| LLM Routing | Gemini Flash (80%) / Claude Haiku (15%) / Claude Sonnet (5%) |
| CRM | HubSpot free tier |
| Blog | Hashnode (blog.cloudsaathi.com) |
| Scheduling | Cal.com |

## Repo Structure

```
agents/          — Agent definitions (YAML)
tools/           — Tool implementations (Python)
flows/           — Orchestration workflows
runtime/         — Execution engine (router, policy, approval, state)
memory/          — SQLite schemas + memory store
config/          — YAML configs + .env template
integrations/    — MCP servers
observability/   — Health checks, cost tracking, digests
deploy/          — Docker, setup scripts, n8n workflows
tests/           — Test suite
docs/specs/      — Design specs
```

## Key Design Decisions

1. **Hub-and-spoke** — All agents communicate through Slack Orchestrator. No direct agent-to-agent communication.
2. **Trust graduation** — L0 (approve all) → L1 (approve on exception) → L2 (auto + audit) → L3 (auto + alerts). Per action type, not per agent.
3. **Multi-model routing** — Cheap models for simple tasks, expensive models only when reasoning quality matters.
4. **Approval via Slack** — Block Kit buttons in #agent-approvals. Timeout = auto-reject, never auto-approve.
5. **Permanent L0** — Emails, invoices, proposals, LinkedIn posts NEVER auto-execute without explicit config override.

## Hard Rules

1. **Never commit secrets.** API keys go in /etc/doot/secrets/ on VPS, never in git.
2. **Never bypass the policy engine.** All write actions go through the trust level check.
3. **Never auto-approve on timeout.** If no human responds in 2 hours, the action is rejected.
4. **Always log.** Every action in the audit trail. Append-only.
5. **Design spec is source of truth.** See `docs/specs/2026-03-20-doot-agents-design.md`.

## GitHub

- **Org:** Doot-Workspaces
- **Repo:** doot-agents
- **Related:** cloud-saathi (the startup's website/business repo)

## Communication

- **Slack:** Doot Workspace (#all-doot-workspace for now, dedicated channels TBD)
- **GitHub:** Issues + PRs on this repo
