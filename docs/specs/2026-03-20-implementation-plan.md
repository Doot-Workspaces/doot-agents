# Doot Agents — Implementation Plan

> **Status:** Draft — Blocked on Ankit's approval
> **Depends on:** [Design Spec](2026-03-20-doot-agents-design.md) (PR #1)
> **Date:** 2026-03-20
> **Rule:** DO NOT start building until Ankit reviews and approves.

---

## Phase 1: Foundation (Week 1-2)

The skeleton: VPS, runtime engine, policy engine, Slack integration, LLM router. No agents yet — just the platform they run on.

### 1.1 Hetzner VPS Setup
**Owner:** Ankit
**Time:** 2-3 hours

- [ ] Create Hetzner Cloud account
- [ ] Provision CX23 in Singapore (sin1) region
- [ ] SSH access configured for both founders
- [ ] Basic hardening: UFW firewall, fail2ban, non-root user
- [ ] Docker + Docker Compose installed
- [ ] Domain: agents.cloudsaathi.com → VPS IP (DNS A record)

**Deliverable:** Running VPS accessible via SSH and domain.

### 1.2 Docker Compose Stack
**Owner:** Ankit
**Time:** 3-4 hours

```yaml
# docker-compose.yaml outline
services:
  n8n:
    image: n8nio/n8n
    ports: ["5678:5678"]
    volumes: ["./n8n-data:/home/node/.n8n"]
    environment:
      - N8N_BASIC_AUTH_ACTIVE=true

  doot-runtime:
    build: ./runtime
    volumes:
      - ./data/sqlite:/data
      - ./config:/config
    env_file: .env
    depends_on: [n8n]

  redis:
    image: redis:alpine
    ports: ["6379:6379"]
    volumes: ["./redis-data:/data"]
```

- [ ] Write Dockerfile for doot-runtime (Python 3.12 + dependencies)
- [ ] Write docker-compose.yaml with n8n, runtime, redis
- [ ] Test `docker compose up` runs all 3 services
- [ ] Set up Nginx reverse proxy for n8n UI (n8n.agents.cloudsaathi.com)

**Deliverable:** `docker compose up` brings up entire stack.

### 1.3 SQLite Schema
**Owner:** Ankit
**Time:** 2-3 hours

- [ ] Create `memory/schemas.sql` with all tables:
  - `trust_levels` — per action type trust graduation state
  - `audit_log` — immutable append-only action log
  - `pending_actions` — actions awaiting approval or rollback
  - `agent_state` — per-agent conversation/context state
  - `cost_tracking` — per-agent token usage and cost
- [ ] Write migration script to initialize DB
- [ ] Write `memory/store.py` — thin Python wrapper for SQLite operations

**Deliverable:** Working SQLite DB with all tables, accessible from Python.

### 1.4 LLM Router
**Owner:** Ankit
**Time:** 4-5 hours

- [ ] Create `runtime/router.py`
- [ ] Implement model selection logic:
  - Input: `task_type` (routing, moderate, complex) + `agent_id`
  - Output: model API call with correct provider
- [ ] Integrate 3 providers:
  - Gemini 2.0 Flash (via Google AI API) — for routing/classification
  - Claude Haiku 4.5 (via Anthropic API) — for moderate reasoning
  - Claude Sonnet 4.6 (via Anthropic API) — for complex reasoning
- [ ] Add per-call cost tracking (log to `cost_tracking` table)
- [ ] Add per-agent monthly budget caps from `config/budgets.yaml`
- [ ] Alert mechanism: log warning at 80%, raise exception at 100%

**Deliverable:** `router.call(agent_id, task_type, prompt)` returns LLM response and logs cost.

### 1.5 Policy Engine
**Owner:** Ankit
**Time:** 4-5 hours

- [ ] Create `runtime/policy.py`
- [ ] Implement trust level checker:
  - Input: `agent_id`, `action_type`
  - Output: current trust level (L0/L1/L2/L3)
  - Logic: read from `trust_levels` table
- [ ] Implement graduation logic:
  - On approval: increment `consecutive_approvals`
  - On rejection: reset to L0, set `last_rejection_at`
  - On auto-success: increment `consecutive_auto_ok`
  - Auto-graduate when thresholds met (50 for L1, 200 for L2)
- [ ] Implement permanent L0 list (configurable via `config/trust_policy.yaml`):
  - `send_email_external`
  - `publish_linkedin`
  - `generate_invoice`
  - `send_proposal`
- [ ] Implement demotion logic: any rollback → back to L0

**Deliverable:** `policy.check(agent_id, action_type)` returns trust level. `policy.record_outcome(action_type, "approved"|"rejected"|"auto_ok"|"rolled_back")` updates state.

### 1.6 Slack Approval Integration
**Owner:** Ankit (with Nihaan setting up Slack channels)
**Time:** 5-6 hours

**Nihaan's tasks (no code):**
- [ ] Create Slack channels: `#agent-approvals`, `#agent-activity`, `#agent-alerts`, `#agent-digest`
- [ ] Create Slack App in Doot Workspace with:
  - Bot token scopes: `chat:write`, `chat:write.customize`, `channels:read`, `channels:manage`
  - Enable Interactivity (for button clicks)
  - Set Request URL to `https://agents.cloudsaathi.com/slack/events`

**Ankit's tasks (code):**
- [ ] Create `runtime/approval.py`
- [ ] Implement `request_approval(agent_id, action_type, summary, details)`:
  - Build Block Kit message with action summary + Approve/Reject buttons
  - Post to `#agent-approvals`
  - Create `pending_actions` record
  - Return pending_action_id
- [ ] Implement Slack webhook handler for button clicks:
  - Parse `action_id` (approve/reject) and `pending_action_id`
  - Update `pending_actions` and `trust_levels` tables
  - If approved: execute the pending action
  - If rejected: log and notify agent
- [ ] Implement timeout: cron job checks `pending_actions` older than 2 hours → auto-reject
- [ ] Implement `/rollback` Slack command:
  - List recent executed actions with Undo buttons
  - On undo: execute compensating action, update `pending_actions` status

**Deliverable:** Agent calls `request_approval()` → Slack message appears → button click executes or rejects action.

### 1.7 Agent Execution Engine
**Owner:** Ankit
**Time:** 5-6 hours

- [ ] Create `runtime/engine.py`
- [ ] Implement base `Agent` class:
  ```python
  class Agent:
      agent_id: str
      name: str
      system_prompt: str
      tools: list[Tool]
      model_tier: str  # "routing", "moderate", "complex"

      def run(self, input: dict) -> dict:
          # 1. Load agent config from YAML
          # 2. Call LLM via router
          # 3. If LLM requests tool use → check policy → execute or request approval
          # 4. Log to audit trail
          # 5. Return result
  ```
- [ ] Implement tool execution with policy gate:
  - Before any tool call: `policy.check(agent_id, action_type)`
  - If L0: `approval.request_approval()` → wait for Slack response
  - If L1+: execute immediately, post notification
- [ ] Implement agent loading from YAML definitions (`agents/*.yaml`)
- [ ] Implement agent health check endpoint

**Deliverable:** `engine.run_agent("lead-qual", input)` loads agent, calls LLM, enforces policy, logs everything.

### 1.8 Orchestrator Agent (Agent 3)
**Owner:** Ankit
**Time:** 4-5 hours

- [ ] Create `agents/orchestrator.yaml` definition
- [ ] Implement orchestrator-specific logic in `flows/orchestrator.py`:
  - Listen for Slack messages in designated channels
  - Route requests to appropriate specialist agent
  - Aggregate daily digest (cron: 9am IST daily)
  - Health check all agents (cron: every 5 minutes)
- [ ] Set up n8n workflows:
  - Slack event → webhook → orchestrator
  - Cron triggers (daily digest, health checks)
  - Email inbox webhook (for inbound lead detection)

**Deliverable:** Slack Orchestrator running, responding to commands, health-checking.

### Phase 1 Total Estimate
**Calendar time:** ~2 weeks (assuming Ankit works ~3-4 hours/day on this)
**Critical path:** VPS → Docker → SQLite → Router → Policy → Approval → Engine → Orchestrator (sequential)

---

## Phase 2: Core Agents (Week 3-4)

Build the 5 specialist launch agents on top of the Phase 1 platform.

### 2.1 Google Workspace Integrations
**Owner:** Ankit
**Time:** 4-5 hours

- [ ] Create `tools/gmail.py` — send, read, draft, search
- [ ] Create `tools/calendar.py` — create/read/update events
- [ ] Create `tools/sheets.py` — read/write cells and ranges
- [ ] Create `tools/docs.py` — create from template, edit
- [ ] Set up Google Cloud project with OAuth credentials
- [ ] Store credentials in `/etc/doot/secrets/google.env`

### 2.2 HubSpot CRM Integration
**Owner:** Ankit
**Time:** 3-4 hours

- [ ] Create HubSpot free account for Cloud Saathi
- [ ] Create `tools/hubspot.py` — contacts, deals, pipeline stages
- [ ] Set up deal pipeline stages: Lead → Qualified → Discovery → Proposal → Negotiation → Won/Lost
- [ ] Store API key in `/etc/doot/secrets/hubspot.env`

### 2.3 Meeting Scheduler Agent (#10)
**Owner:** Ankit
**Time:** 4-5 hours

- [ ] Deploy Cal.com via Docker on VPS (meet.cloudsaathi.com)
- [ ] Create `tools/calcom.py` — create booking links, read bookings
- [ ] Create `agents/meeting_scheduler.yaml`
- [ ] Implement `flows/meeting_lifecycle.py`:
  - Trigger: meeting request via Slack or email
  - Step 1: Send Cal.com availability link
  - Step 2: 24h before meeting → send agenda email (draft, L0 approval)
  - Step 3: After meeting → accept notes/transcript paste → generate summary
  - Step 4: Post action items to Slack
  - Step 5: Send follow-up email (draft, L0 approval)
- [ ] Test full lifecycle end-to-end

### 2.4 Lead Qualification Agent (#4)
**Owner:** Ankit
**Time:** 4-5 hours

- [ ] Create `agents/lead_qual.yaml` with scoring prompt
- [ ] Define Ideal Customer Profile in agent system prompt:
  - Company stage: Seed to Series A
  - Team size: 5-50 engineers
  - Tech stack: AWS/GCP/Azure
  - Pain: No dedicated DevOps, manual deployments, no CI/CD
- [ ] Implement `flows/lead_qualification.py`:
  - Trigger: new contact in HubSpot, or manual Slack command
  - Step 1: Enrich lead data (company website, LinkedIn, team size)
  - Step 2: Score against ICP (0-100)
  - Step 3: If score > 70 → route to founder via Slack with context brief
  - Step 4: Update HubSpot contact with score and notes
- [ ] Set up n8n webhook for website contact form → lead qualification trigger

### 2.5 Proposal/SOW Generator Agent (#5)
**Owner:** Ankit
**Time:** 5-6 hours

- [ ] Create proposal templates in Google Docs:
  - Template 1: Fractional DevOps Retainer
  - Template 2: Infrastructure Audit
- [ ] Create `agents/proposal_gen.yaml`
- [ ] Implement `flows/proposal_flow.py`:
  - Trigger: Slack command with lead context or discovery call notes
  - Step 1: Pull lead data from HubSpot
  - Step 2: Select matching template
  - Step 3: Generate personalized scope, timeline, deliverables (Claude Sonnet)
  - Step 4: Create Google Doc draft from template
  - Step 5: Post to Slack for founder review (always L0)
- [ ] Test with sample lead data

### 2.6 Client Onboarding Agent (#7)
**Owner:** Ankit
**Time:** 4-5 hours

- [ ] Create `agents/client_onboarding.yaml`
- [ ] Create onboarding templates:
  - Welcome email template
  - Pre-kickoff questionnaire (Google Form)
  - Client folder structure (Google Drive template)
- [ ] Implement `flows/deal_to_onboarding.py`:
  - Trigger: deal marked "Won" in HubSpot, or manual Slack command
  - Step 1: Send welcome email draft (L0 approval)
  - Step 2: Create shared Google Drive folder
  - Step 3: Schedule kickoff call via Cal.com
  - Step 4: Send pre-kickoff questionnaire
  - Step 5: Post onboarding checklist to Slack

### 2.7 Invoice & Financial Tracker Agent (#6)
**Owner:** Ankit
**Time:** 4-5 hours

- [ ] Create `tools/invoice.py` — PDF invoice generation (from template)
- [ ] Create active contracts Google Sheet (client, amount, billing date, status)
- [ ] Create `agents/invoice_tracker.yaml`
- [ ] Implement `flows/invoice_cycle.py`:
  - Trigger: cron on billing dates (e.g., 1st of each month)
  - Step 1: Read active contracts from Google Sheet
  - Step 2: Generate invoice PDF for each due contract
  - Step 3: Post each invoice to Slack for approval (always L0)
  - Step 4: On approval → draft payment reminder email
  - Step 5: Track payment status in Google Sheet
  - Step 6: If overdue > 7 days → alert in `#agent-alerts`

### Phase 2 Total Estimate
**Calendar time:** ~2 weeks
**Can partially parallelize:** Google integrations (2.1) + HubSpot (2.2) can happen in parallel. Agent builds (2.3-2.7) are independent of each other once integrations are done.

---

## Phase 3: Marketing Agents (Month 2-3)

### 3.1 Pre-requisites (Nihaan — no code)
- [ ] Apply for LinkedIn Marketing Developer Platform access (2-4 week approval)
- [ ] Set up Google Workspace (nihaan@cloudsaathi.com, ankit@cloudsaathi.com) — $6/user/month
- [ ] Set up Hashnode blog (blog.cloudsaathi.com) with custom domain
- [ ] Create content calendar template in Google Sheets
- [ ] Define 10-15 target keywords for SEO (e.g., "fractional DevOps", "DevOps for startups")

### 3.2 LinkedIn Marketing Agent (#1)
**Owner:** Ankit
**Time:** 5-6 hours

- [ ] Create `tools/linkedin.py` — post, read engagement, comment
- [ ] Create `agents/linkedin_marketing.yaml`
- [ ] Implement content generation flow:
  - Cron: weekly content calendar generation (Monday 9am)
  - Daily: draft post for today from calendar (8am)
  - Post to Slack for approval (always L0)
  - On approval: publish via LinkedIn API (or manual paste if API not approved yet)
  - Track engagement metrics

### 3.3 Sales Outreach Agent (#2)
**Owner:** Ankit
**Time:** 5-6 hours

- [ ] Create `agents/sales_outreach.yaml`
- [ ] Create email sequence templates:
  - Sequence 1: Cold outreach (3 emails over 10 days)
  - Sequence 2: Warm intro follow-up (2 emails over 5 days)
  - Sequence 3: Re-engagement (1 email after 30 days)
- [ ] Implement multi-step email sequence with delays
- [ ] All emails always L0 (external-facing)
- [ ] Track open/reply rates (via Gmail read receipts or pixel tracking)

### 3.4 Content & SEO Agent (#8)
**Owner:** Ankit
**Time:** 5-6 hours

- [ ] Create `tools/hashnode.py` — publish, update, list posts
- [ ] Create `agents/content_seo.yaml`
- [ ] Implement content pipeline:
  - Monthly: research trending topics + keyword gaps
  - Weekly: generate 1 long-form blog post draft (Claude Sonnet)
  - Post to Slack for review (always L0)
  - On approval: publish to Hashnode
  - Repurpose into LinkedIn post brief → feed to Agent 1

### Phase 3 Total Estimate
**Calendar time:** ~2 weeks
**Blocker:** LinkedIn API approval (apply in Phase 1, should be ready by Phase 3)

---

## Phase 4: Intelligence & Optimization (Month 3+)

### 4.1 Competitor Intelligence Agent (#9)
- [ ] Create `agents/competitor_intel.yaml`
- [ ] Build competitor tracker (list of URLs, LinkedIn profiles)
- [ ] Weekly digest to `#agent-digest`
- [ ] Nice-to-have: job posting monitoring (hiring = growth signal)

### 4.2 Observability Dashboard
- [ ] Daily cost breakdown by agent posted to `#agent-digest`
- [ ] Trust level status report (which actions have graduated)
- [ ] Agent response time tracking
- [ ] Monthly cost trend analysis

### 4.3 Cost Optimization
- [ ] Enable Anthropic prompt caching for repeated system prompts
- [ ] Batch non-urgent tasks via batch APIs (50% discount)
- [ ] Analyze per-agent cost data and adjust model routing thresholds
- [ ] Target: reduce LLM spend by 30% from Phase 2 baseline

### 4.4 Advanced Features
- [ ] Multi-agent collaboration (Lead Qual → Proposal Gen handoff without human mediation)
- [ ] Agent performance scoring (which agents get the most rejections)
- [ ] Founder preference learning (track patterns in approvals/rejections to auto-calibrate)

---

## Summary Timeline

| Week | Phase | What's built | Milestone |
|---|---|---|---|
| 1-2 | Phase 1: Foundation | VPS, Docker, SQLite, Router, Policy, Approval, Engine, Orchestrator | Platform running, Orchestrator responding in Slack |
| 3-4 | Phase 2: Core Agents | Meeting Scheduler, Lead Qual, Proposal Gen, Onboarding, Invoice | 6 agents operational, first Slack approvals flowing |
| 5-8 | Phase 3: Marketing | LinkedIn, Sales Outreach, Content & SEO | Full marketing pipeline, blog live |
| 9+ | Phase 4: Optimize | Competitor Intel, cost optimization, observability | Cost reduced, some actions graduating to L1+ |

## Pre-requisites Checklist (Both Founders)

Before Phase 1 starts:

**Ankit:**
- [ ] Review and approve design spec (PR #1)
- [ ] Confirm tech stack choice (Approach B: LangGraph + Multi-Model)
- [ ] Create Hetzner Cloud account

**Nihaan:**
- [ ] Apply for LinkedIn Developer App + Marketing Developer Platform
- [ ] Set up Google Cloud project (for OAuth credentials)
- [ ] Create HubSpot free account
- [ ] Create Slack channels (#agent-approvals, #agent-activity, #agent-alerts, #agent-digest)
- [ ] Create Slack App with bot permissions
