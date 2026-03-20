# Doot Agents — Design Specification

> **Status:** Draft v2 — Pending Ankit's review (spec review fixes applied)
> **Authors:** Nihaan Mohammed, Claude (AI architect)
> **Date:** 2026-03-20
> **Repo:** `Doot-Workspaces/doot-agents`
> **Slack thread:** [#all-doot-workspace](https://dootworkspacegroup.slack.com/archives/C0AHMHFUL01/p1773950740817889)

---

## 1. Problem Statement

Cloud Saathi is a 2-person fractional DevOps startup. Nihaan and Ankit cannot simultaneously:
- Generate leads and qualify them
- Write proposals and SOWs
- Send outreach emails and follow up
- Schedule meetings and extract action items
- Invoice clients and track payments
- Create content for LinkedIn and blog
- Onboard new clients professionally
- Monitor competitors

Hiring is not an option at this stage. Autonomous agents are.

---

## 2. Solution Overview

**Doot Agents** is a hub-and-spoke autonomous agent system that handles Cloud Saathi's operational workload. 10 specialized agents coordinate through a central Slack Orchestrator. All agents start in "Draft + Approve" mode (human-in-the-loop) and graduate to full autonomy based on a trust-scoring system.

### Design Principles

1. **Approval-first.** Nothing leaves Cloud Saathi's walls without a founder clicking Approve. Trust is earned, not assumed.
2. **Cost-obsessive.** Infrastructure under $5/month. LLM costs under $50/month. Route 80% of tasks to the cheapest capable model.
3. **Eat your own dog food.** You're a DevOps company. Self-host everything. Your agent infra IS your portfolio piece.
4. **One hub, many spokes.** Slack Orchestrator is the single source of truth. All agents report through it. Debug from one place.
5. **Rollback everything.** Every write action is reversible. Irreversible actions (sending emails) require perpetual human approval.

---

## 3. Agent Roster

### Agent Naming Convention

Agents use **slug names** in code, logs, and messages (e.g., `orchestrator`, `lead-qual`, `proposal-gen`). Numeric IDs below are for document reference only.

### 3.1 Launch Agents (Day 1) — 6 agents

#### Agent 1: Slack Orchestrator (Hub) — `orchestrator`
- **Role:** Central coordinator for all agents
- **Responsibilities:**
  - Route incoming requests to the right specialist agent
  - Surface approval requests via Slack Block Kit buttons
  - Aggregate daily digest of all agent activity
  - Monitor agent health and flag failures
  - Enforce the graduation/trust model
- **Inputs:** Slack messages, webhooks from other agents, cron triggers
- **Outputs:** Slack messages (approvals, digests, alerts), agent dispatch commands
- **Tools:** Slack API, internal agent registry, SQLite state store
- **Model tier:** Gemini Flash (routing is cheap reasoning)

#### Agent 2: Lead Qualification & Scoring — `lead-qual`
- **Role:** Score and route inbound leads
- **Responsibilities:**
  - Monitor inbound signals (website contact form, LinkedIn DMs, email replies)
  - Score leads based on: company stage, tech stack, team size, budget signals
  - Route qualified leads to Sales Outreach agent with context brief
  - Update CRM (HubSpot free tier) with lead data and score
- **Inputs:** Webhook from website form, Gmail inbox scan, LinkedIn notifications
- **Outputs:** Lead score + context brief → Slack for review, CRM update
- **Tools:** HubSpot API, Gmail API, webhook listener
- **Model tier:** Claude Haiku (needs some reasoning for scoring)
- **Approval gate:** CRM updates auto-execute; lead routing to Sales agent needs approval at L0

#### Agent 3: Proposal & SOW Generator — `proposal-gen`
- **Role:** Generate custom proposals from templates + lead context
- **Responsibilities:**
  - Take discovery call notes (pasted into Slack or from meeting summary)
  - Pull matching template from template library
  - Generate tailored scope, timeline, pricing, deliverables
  - Output as Google Doc draft
- **Inputs:** Discovery call notes, lead context from CRM, template library
- **Outputs:** Google Doc draft proposal → Slack for founder review
- **Tools:** Google Docs API, Google Drive API, HubSpot API
- **Model tier:** Claude Sonnet (complex reasoning — proposals are high-value)
- **Approval gate:** Always L0 (proposals go to clients — never auto-send)

#### Agent 4: Invoice & Financial Tracker — `invoice-tracker`
- **Role:** Automate billing cycle for retainer clients
- **Responsibilities:**
  - Generate invoices on schedule (monthly retainers)
  - Track payment status
  - Send payment reminders at configurable intervals
  - Maintain cash flow dashboard in Google Sheets
  - Flag overdue invoices to Slack
- **Inputs:** Active contracts list (Google Sheets), payment confirmations
- **Outputs:** Invoice drafts (PDF), payment reminders (email draft), Slack alerts
- **Tools:** Google Sheets API, Gmail API (for reminders), PDF generation
- **Model tier:** Gemini Flash (templated, low reasoning)
- **Approval gate:** Invoice generation and payment reminders always L0 (financial = always human-approved)

#### Agent 5: Client Onboarding — `client-onboard`
- **Role:** Execute post-close onboarding sequence
- **Responsibilities:**
  - Send welcome email from template
  - Create shared Slack channel for client (if applicable)
  - Schedule kickoff call via Cal.com
  - Send pre-kickoff questionnaire (Google Form)
  - Create client workspace in Notion/Google Drive
- **Inputs:** "Deal closed" trigger from CRM or manual Slack command
- **Outputs:** Welcome email draft, calendar invite, shared channel, questionnaire link
- **Tools:** Gmail API, Google Calendar API, Google Forms API, Google Drive API
- **Model tier:** Gemini Flash (mostly templated actions)
- **Approval gate:** Welcome email at L0; internal setup (folder creation, channel) can be L1

#### Agent 6: Meeting Scheduler & Follow-up — `meeting-sched`
- **Role:** Eliminate scheduling friction and capture meeting outcomes
- **Responsibilities:**
  - Send availability links (Cal.com) when a meeting is requested
  - Send pre-meeting agenda 24h before
  - After meeting: generate summary from notes/transcript
  - Extract action items and post to Slack
  - Send follow-up email with summary to attendees
- **Inputs:** Meeting request (Slack or email), calendar events, meeting notes/transcript
- **Outputs:** Calendar invites, agenda emails, summary + action items to Slack, follow-up email drafts
- **Tools:** Google Calendar API, Gmail API, Cal.com API
- **Model tier:** Claude Haiku (summarization needs moderate reasoning)
- **Approval gate:** Follow-up emails at L0; internal summaries to Slack at L1

### 3.2 Month 3 Agents — 3 agents

#### Agent 7: LinkedIn Marketing — `linkedin-mkt`
- **Role:** Build Cloud Saathi's LinkedIn presence
- **Responsibilities:**
  - Generate content calendar (weekly)
  - Draft posts (thought leadership, case studies, DevOps tips)
  - Schedule posts at optimal times
  - Monitor engagement and suggest reply drafts
- **Inputs:** Content calendar, trending DevOps topics, past performance data
- **Outputs:** Post drafts → Slack for approval, engagement reports
- **Tools:** LinkedIn API, content research (web search)
- **Model tier:** Claude Haiku (creative writing needs moderate quality)
- **Approval gate:** All posts at L0 (public-facing)

#### Agent 8: Sales/Partnership Outreach — `sales-outreach`
- **Role:** Cold email sequences to prospects and potential partners
- **Responsibilities:**
  - Generate personalized outreach emails from lead context
  - Execute multi-step email sequences (initial, follow-up 1, follow-up 2)
  - Track open/reply rates
  - Route positive replies to Lead Qualification agent
- **Inputs:** Qualified lead list from CRM, email templates, personalization context
- **Outputs:** Email drafts → Slack for approval, sequence status reports
- **Tools:** Gmail API, HubSpot API
- **Model tier:** Claude Haiku (personalization needs moderate reasoning)
- **Approval gate:** All outbound emails at L0 (client-facing = always human-approved)

#### Agent 9: Content & SEO — `content-seo`
- **Role:** Drive organic search traffic to cloudsaathi.com
- **Responsibilities:**
  - Research trending DevOps keywords and topics
  - Generate long-form blog posts (SEO-optimized)
  - Repurpose blog content into LinkedIn posts (feed to Agent 1)
  - Monitor search rankings and suggest optimizations
- **Inputs:** Keyword research data, Google Search Console data, competitor content
- **Outputs:** Blog post drafts, SEO reports, content briefs for LinkedIn agent
- **Tools:** Hashnode API (blog.cloudsaathi.com), Google Search Console API, web search
- **Model tier:** Claude Sonnet (long-form content needs high quality)
- **Approval gate:** Published content at L0; internal research/briefs at L1
- **Platform:** Hashnode (free, custom domain support, full SEO control, dev audience, markdown native)

### 3.3 Nice-to-Have — 1 agent

#### Agent 10: Competitor & Market Intelligence — `competitor-intel`
- **Role:** Monitor competitive landscape
- **Responsibilities:**
  - Track competitor websites, pricing pages, LinkedIn activity
  - Monitor job postings (hiring signals = growth signals)
  - Synthesize weekly intelligence digest
- **Inputs:** Competitor URLs, LinkedIn profiles, job boards
- **Outputs:** Weekly digest to Slack
- **Tools:** Web scraping, LinkedIn data, Google Alerts
- **Model tier:** Gemini Flash (mostly data collection + light synthesis)
- **Approval gate:** No approval needed (internal intelligence only)

---

## 4. Architecture

### 4.1 Hub-and-Spoke Topology

```
                    +------------------------+
                    |   Slack Orchestrator    |
                    |       (THE HUB)        |
                    |                        |
                    | - Routes requests      |
                    | - Surfaces approvals   |
                    | - Monitors health      |
                    | - Enforces trust model |
                    +------------------------+
                   /    |    |    |    |     \
                  /     |    |    |    |      \
           +------+ +------+ +------+ +------+ +------+ +------+
           |Lead  | |Prop/ | |Inv/  | |Client| |Meet  | |Linke-|
           |Qual  | |SOW   | |Fin   | |Onbrd | |Sched | |dIn   |
           |  #4  | | #5   | | #6   | |  #7  | | #10  | |  #1  |
           +------+ +------+ +------+ +------+ +------+ +------+
                                                    |
                                              +------+ +------+
                                              |Sales | |Cont/ |
                                              |Out#2 | |SEO#8 |
                                              +------+ +------+
                                                         |
                                                    +------+
                                                    |Comp |
                                                    |Intel|
                                                    | #9  |
                                                    +------+
```

### 4.2 Data Flow

```
[Inbound Signal] → [Webhook / Cron / Slack Command]
       ↓
[Slack Orchestrator] → identifies which agent should handle
       ↓
[Specialist Agent] → does the work (draft, research, generate)
       ↓
[Approval Gate] → checks trust level for this action type
       ↓
  ┌─ L0: Post to Slack with Approve/Reject buttons → wait for human
  ├─ L1+: Auto-execute → post notification to Slack
  └─ Execute action (send email, update CRM, publish post)
       ↓
[Audit Log] → record action, input, output, who approved, timestamp
       ↓
[State Update] → update SQLite with new state
```

### 4.3 Communication Protocol

Agents communicate exclusively through the Orchestrator via a message queue pattern:

```python
# Every agent message follows this schema
{
    "from_agent": "lead-qual",
    "to_agent": "orchestrator",       # Always goes through hub
    "action": "route_to_sales",
    "payload": {
        "lead_id": "L-0042",
        "score": 85,
        "context": "Series A startup, 12 engineers, no DevOps, AWS"
    },
    "requires_approval": true,
    "trust_level_required": "L0",
    "timestamp": "2026-03-20T14:30:00Z"
}
```

---

## 5. Tech Stack

### 5.1 Infrastructure

| Layer | Choice | Cost | Why |
|---|---|---|---|
| **Hosting** | Hetzner CX23 (2 vCPU, 4GB RAM, 40GB NVMe) | ~$4/mo | Best price/performance. EU-based. 20TB traffic. Comfortably runs 10 agents + n8n + SQLite. |
| **Orchestration** | n8n self-hosted + cron | $0 | Visual workflow builder for triggers/webhooks. 400+ integrations. Free when self-hosted. |
| **Agent Framework** | OpenAI Agents SDK (or raw Python) | $0 | Works with any LLM despite the name. Low token overhead (unlike CrewAI's 56% overhead). Built-in tracing + guardrails. |
| **State/Memory** | SQLite on VPS | $0 | Zero ops, survives restarts, surprisingly performant. Primary store for conversation logs, task state, agent memory. |
| **Hot Cache** | Upstash Redis (free tier) | $0 | 256MB, 500K commands/month. For sub-millisecond agent coordination if needed. |
| **Tool Integration** | Direct API calls + open-source MCP servers | $0 | Write direct integrations for core tools. Use community MCP servers for Slack, Gmail, Calendar, GitHub. |

### 5.2 External Services (Free Tiers)

| Service | Purpose | Tier | Cost |
|---|---|---|---|
| **HubSpot** | CRM — lead tracking, deal pipeline | Free (up to 1M contacts) | $0 |
| **Cal.com** | Scheduling links | Self-hosted or free cloud | $0 |
| **Hashnode** | Blog (blog.cloudsaathi.com) | Free with custom domain | $0 |
| **Google Analytics / Plausible** | Website analytics | Free tier | $0 |
| **Google Sheets** | Cash flow dashboard, templates | Free (via Google account) | $0 |
| **Google Docs** | Proposal/SOW output | Free (via Google account) | $0 |

### 5.3 LLM Strategy — Multi-Model Routing

| Task Type | Model | Cost (per M tokens) | % of workload |
|---|---|---|---|
| Routing, classification, simple templating | Gemini 2.0 Flash | $0.10 input / $0.40 output | 80% |
| Moderate reasoning (scoring, summarization, personalization) | Claude Haiku 4.5 | $1.00 input / $5.00 output | 15% |
| Complex reasoning (proposals, long-form content, strategy) | Claude Sonnet 4.6 | $3.00 input / $15.00 output | 5% |

**Estimated monthly LLM cost:** $20-50 depending on volume.

**Cost optimization tactics:**
- Batch non-urgent work via batch APIs (50% discount on all major providers)
- Prompt caching on Claude (90% reduction on repeated context)
- Plan-and-Execute pattern: expensive model creates strategy, cheap model executes steps

### 5.4 Monthly Cost Summary

| Component | Cost |
|---|---|
| Hetzner VPS | ~$4 |
| n8n, SQLite, Redis, MCP servers | $0 |
| HubSpot, Cal.com, Hashnode, Google tools | $0 |
| LLM API costs | $20-50 |
| **Total** | **$24-54/month** |

---

## 6. Approval & Trust Model

### 6.1 Approval Flow

```
Agent wants to perform action
       ↓
Policy Engine checks: what trust level does this action type have?
       ↓
  ┌─ L0 (Draft + Approve):
  │   → Create PendingAction record
  │   → Send Slack Block Kit message to #agent-approvals
  │   → Message includes: action summary, affected data, Approve/Reject buttons
  │   → Wait for button click (timeout: 2 hours → auto-reject)
  │   → On Approve: execute action, log result
  │   → On Reject: log rejection, notify agent
  │
  ├─ L1 (Approve-on-exception):
  │   → Execute action immediately
  │   → Post notification to Slack (not a gate)
  │   → Log result
  │
  ├─ L2 (Full auto + audit):
  │   → Execute action immediately
  │   → Log result (no per-action Slack notification)
  │   → Include in daily digest
  │
  └─ L3 (Full auto + alerts):
      → Execute action immediately
      → Log result
      → Slack only on errors/anomalies
```

### 6.2 Trust Graduation

Graduation is **per action type**, not per agent. An agent can have some actions at L0 and others at L2.

| Level | Name | Graduation criteria | Demotion criteria |
|---|---|---|---|
| L0 | Draft + Approve | Default for all actions | Any rollback → demote to L0 |
| L1 | Approve-on-exception | 50 consecutive approvals, 0 rejections | 1 rejection → back to L0 |
| L2 | Full auto + audit | 200 consecutive auto-executions, 0 rollbacks | 1 rollback → back to L0 |
| L3 | Full auto + alerts | Manual founder decision only | Manual founder decision |

**Trust state is stored in SQLite:**

```sql
CREATE TABLE trust_levels (
    action_type TEXT PRIMARY KEY,     -- e.g., "send_email", "update_crm"
    agent_id TEXT NOT NULL,
    current_level INTEGER DEFAULT 0,  -- 0, 1, 2, 3
    consecutive_approvals INTEGER DEFAULT 0,
    consecutive_auto_ok INTEGER DEFAULT 0,
    last_rejection_at TIMESTAMP,
    last_rollback_at TIMESTAMP,
    graduated_at TIMESTAMP,
    demoted_at TIMESTAMP
);
```

### 6.3 Permanent L0 Actions

Some actions NEVER graduate beyond L0:
- Sending emails to external recipients (prospects, clients, partners)
- Publishing LinkedIn posts
- Generating/sending invoices
- Creating proposals/SOWs
- Any action involving money

These can only be moved to L1+ by explicit founder override in config.

---

## 7. Security Model

### 7.1 Core Principle

**Approval enforcement lives OUTSIDE the agent.** Agents cannot modify their own trust levels, access the policy engine's rules, or bypass the approval middleware.

### 7.2 Architecture

```
[Agent] → [Policy Engine / Middleware] → [External API]
              ↓
        Checks trust level
        Injects credentials at runtime
        Logs to audit trail
```

- **Agents never hold credentials.** API keys and tokens are stored in environment variables on the VPS, injected by the middleware at execution time.
- **Least-privilege by default.** Agent API tokens have read-only access. Write access is granted per-action by the policy engine.
- **Sandboxed execution.** Each agent runs in its own process. No agent can access another agent's memory or modify shared config.
- **Immutable audit log.** Append-only SQLite table. Every action recorded with: timestamp, agent_id, action_type, input_hash, output_hash, approval_source (human/auto), trust_level_at_time.

### 7.3 Credential Management

```
VPS Environment:
├── /etc/doot/secrets/          # API keys (chmod 600, root-owned)
│   ├── gmail.env
│   ├── hubspot.env
│   ├── linkedin.env
│   ├── slack.env
│   ├── calendar.env
│   └── llm_providers.env
├── /etc/doot/config/           # Non-secret config (chmod 644)
│   ├── agents.yaml             # Agent definitions
│   ├── trust_policy.yaml       # Default trust levels per action type
│   └── routing.yaml            # Model routing rules
```

- Secrets are NEVER committed to git
- Secrets are NEVER passed through Slack messages
- Secrets are NEVER stored in agent memory/context

### 7.4 Audit Trail Schema

```sql
CREATE TABLE audit_log (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    timestamp DATETIME DEFAULT CURRENT_TIMESTAMP,
    agent_id TEXT NOT NULL,
    action_type TEXT NOT NULL,
    target TEXT,                       -- e.g., email address, CRM record ID
    input_summary TEXT,                -- Truncated, never contains secrets
    output_summary TEXT,
    approval_source TEXT,              -- "human:nihaan", "human:ankit", "auto:L1", "auto:L2"
    trust_level_at_time INTEGER,
    status TEXT,                       -- "pending", "approved", "rejected", "executed", "rolled_back"
    rollback_of INTEGER REFERENCES audit_log(id),
    cost_tokens INTEGER,              -- Token count for LLM call
    cost_usd REAL                     -- Estimated cost
);
```

---

## 8. Rollback System

### 8.1 Internal State Rollback

LangGraph-style checkpointing for agent internal state:
- Every execution step is checkpointed to SQLite
- Can inspect any prior checkpoint
- Can fork execution from any checkpoint
- Used for debugging "what happened" and replaying with different inputs

### 8.2 External Action Rollback

For actions that affect the real world:

| Action Type | Rollback Strategy |
|---|---|
| CRM field updated | Restore previous value from PendingAction record |
| Google Sheet row modified | Restore previous value from snapshot |
| Calendar event created | Delete the event |
| Email sent | Cannot undo — this is why emails stay at L0 |
| LinkedIn post published | Delete the post (but impressions can't be undone — hence L0) |
| Invoice sent | Send correction notice — hence L0 |

### 8.3 PendingAction Table

```sql
CREATE TABLE pending_actions (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    agent_id TEXT NOT NULL,
    action_type TEXT NOT NULL,
    target TEXT,
    old_value TEXT,                    -- JSON snapshot of previous state
    new_value TEXT,                    -- JSON snapshot of new state
    status TEXT DEFAULT 'pending',     -- "pending", "executed", "rolled_back"
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    executed_at DATETIME,
    rolled_back_at DATETIME,
    rolled_back_by TEXT               -- "nihaan", "ankit", "system"
);
```

### 8.4 Slack Rollback Command

Founders can type `/rollback` in Slack to see recent actions and reverse any of them. The Orchestrator presents a list of recent PendingActions with "Undo" buttons.

---

## 9. Observability

### 9.1 Slack Channels

| Channel | Purpose | Who posts |
|---|---|---|
| `#agent-approvals` | Approval requests (Block Kit buttons) | All agents |
| `#agent-activity` | Real-time feed of all agent actions | Orchestrator |
| `#agent-alerts` | Errors, failures, budget warnings | Orchestrator |
| `#agent-digest` | Daily summary of all agent activity | Orchestrator (cron) |

### 9.2 Cost Monitoring

- Per-agent token usage tracked in audit_log
- Daily cost rollup posted to `#agent-digest`
- Hard monthly budget cap per agent in config
- Alert at 80% of monthly budget
- Auto-pause agent at 100% of budget

### 9.3 Health Checks

- Orchestrator pings each agent every 5 minutes
- If agent doesn't respond in 30 seconds → alert to `#agent-alerts`
- If agent fails 3 consecutive health checks → auto-restart + alert

---

## 10. Repo Structure

```
doot-agents/
├── agents/                      # Agent definitions (YAML)
│   ├── orchestrator.yaml        # Slack Orchestrator (Hub)
│   ├── lead_qual.yaml           # Lead Qualification & Scoring
│   ├── proposal_gen.yaml        # Proposal/SOW Generator
│   ├── invoice_tracker.yaml     # Invoice & Financial Tracker
│   ├── client_onboarding.yaml   # Client Onboarding
│   ├── meeting_scheduler.yaml   # Meeting Scheduler & Follow-up
│   ├── linkedin_marketing.yaml  # LinkedIn Marketing (Month 3)
│   ├── sales_outreach.yaml      # Sales/Partnership Outreach (Month 3)
│   ├── content_seo.yaml         # Content & SEO (Month 3)
│   └── competitor_intel.yaml    # Competitor Intelligence (Nice-to-have)
├── tools/                       # Tool implementations
│   ├── gmail.py                 # Gmail send/read/draft
│   ├── slack.py                 # Slack message/button/channel
│   ├── calendar.py              # Google Calendar events
│   ├── hubspot.py               # CRM operations
│   ├── sheets.py                # Google Sheets read/write
│   ├── docs.py                  # Google Docs create/edit
│   ├── linkedin.py              # LinkedIn post/engage
│   ├── calcom.py                # Cal.com scheduling
│   └── hashnode.py              # Blog publish
├── flows/                       # Orchestration workflows
│   ├── lead_to_proposal.py      # Lead Qual → Proposal Gen flow
│   ├── deal_to_onboarding.py    # Deal Close → Client Onboarding flow
│   ├── meeting_lifecycle.py     # Request → Schedule → Summary → Follow-up
│   ├── content_pipeline.py      # Research → Draft → Review → Publish
│   └── invoice_cycle.py         # Generate → Approve → Send → Track
├── runtime/                     # Execution engine
│   ├── engine.py                # Agent execution loop
│   ├── router.py                # Multi-model LLM routing
│   ├── policy.py                # Trust level enforcement
│   ├── approval.py              # Slack approval gate
│   └── state.py                 # State management (SQLite)
├── memory/                      # Agent memory
│   ├── store.py                 # SQLite-backed memory store
│   └── schemas.sql              # Trust levels, audit log, pending actions
├── config/                      # Configuration
│   ├── agents.yaml              # Master agent registry
│   ├── trust_policy.yaml        # Default trust levels
│   ├── routing.yaml             # LLM model routing rules
│   ├── budgets.yaml             # Per-agent monthly token budgets
│   └── .env.example             # Template for secrets
├── integrations/                # External service connectors
│   └── mcp_servers/             # MCP server configs
├── observability/               # Monitoring
│   ├── health.py                # Agent health checks
│   ├── cost_tracker.py          # Token usage and cost tracking
│   └── digest.py                # Daily digest generator
├── deploy/                      # Deployment
│   ├── Dockerfile               # Container for VPS deployment
│   ├── docker-compose.yaml      # Full stack (agents + n8n + redis)
│   ├── setup.sh                 # Hetzner VPS bootstrap script
│   └── n8n/                     # n8n workflow exports
├── tests/                       # Test suite
│   ├── test_policy.py           # Trust level enforcement tests
│   ├── test_approval.py         # Approval flow tests
│   ├── test_routing.py          # LLM routing tests
│   └── test_agents/             # Per-agent integration tests
├── docs/
│   └── specs/
│       └── 2026-03-20-doot-agents-design.md  # This document
├── .gitignore
├── .env.example
├── requirements.txt
├── CLAUDE.md                    # Claude Code project instructions
└── README.md                    # Project overview
```

---

## 11. Implementation Phases

### Phase 1: Foundation (Week 1-2)
- Set up Hetzner VPS
- Deploy n8n self-hosted
- Build runtime engine (execution loop, state management, SQLite schemas)
- Build policy engine (trust levels, approval gates)
- Build Slack integration (Block Kit approvals, channels)
- Build LLM router (Gemini Flash / Claude Haiku / Claude Sonnet)
- Deploy Orchestrator agent

### Phase 2: Core Agents (Week 3-4)
- Meeting Scheduler & Follow-up (Agent 10)
- Lead Qualification & Scoring (Agent 4)
- Proposal/SOW Generator (Agent 5)
- Client Onboarding (Agent 7)
- Invoice & Financial Tracker (Agent 6)
- HubSpot CRM integration
- Google Workspace integrations (Sheets, Docs, Calendar, Gmail)

### Phase 3: Marketing Agents (Month 2-3)
- LinkedIn Marketing (Agent 1)
- Sales/Partnership Outreach (Agent 2)
- Content & SEO (Agent 8)
- Hashnode blog setup (blog.cloudsaathi.com)
- LinkedIn API integration

### Phase 4: Intelligence & Polish (Month 3+)
- Competitor Intelligence (Agent 9)
- Advanced observability dashboard
- Trust level analytics
- Cost optimization (prompt caching, batch APIs)

---

## 12. Success Criteria

| Metric | Target | Timeframe |
|---|---|---|
| Agents operational | 6 launch agents running | Week 4 |
| Monthly infra cost | Under $55 total | Ongoing |
| Proposal generation time | Under 15 minutes (vs 3-5 hours manual) | Week 4 |
| Meeting follow-up rate | 100% (vs ~60% manual) | Week 4 |
| Invoice on-time rate | 100% (vs ~80% manual) | Week 4 |
| Lead response time | Under 1 hour (vs ~24 hours manual) | Week 4 |
| Founder daily ops time saved | 3+ hours/day | Month 2 |
| Trust graduation | At least 3 action types at L1+ | Month 3 |

---

## 13. Risks & Mitigations

| Risk | Impact | Mitigation |
|---|---|---|
| Agent sends wrong email to prospect | High — reputation damage | L0 perpetual for all outbound email |
| Invoice with wrong amount | High — financial/legal | L0 perpetual for all financial actions |
| LLM costs spiral | Medium — budget blow | Per-agent hard caps, 80% alerts, auto-pause at 100% |
| Agent hallucinates proposal scope | High — scope creep | Template-constrained generation + mandatory founder review |
| Hetzner VPS goes down | Medium — all agents offline | Daily SQLite backups to Google Drive; redeploy script for recovery in <30min |
| LinkedIn API access revoked | Low — marketing paused | Decouple from core ops; marketing agents are Month 3 |
| Prompt injection via inbound email | Medium — agent manipulation | Input sanitization layer before agent processing; never execute commands from inbound content |
| Over-engineering before PMF | High — wasted effort | Launch with 6 agents max, add rest only after 3 paying clients |

---

## 14. Resolved Decisions (Originally Open Questions)

### Q1: Business Email
**Decision: Start with personal Gmail. Add Google Workspace before Month 3 (outreach launch).**

Rationale: Launch agents (Day 1) don't send external emails — they handle internal ops (scheduling, invoicing, onboarding). The Sales Outreach agent (Agent 2) launches at Month 3. Before that, set up Google Workspace ($6/user/month for nihaan@cloudsaathi.com, ankit@cloudsaathi.com). Outreach from a branded domain is non-negotiable for credibility.

Action item: Set up Google Workspace by end of Month 2.

### Q2: LinkedIn API Access
**Decision: Apply for Marketing Developer Platform access immediately. Use manual posting as fallback.**

Research findings:
- Need to apply via LinkedIn Developer Portal for "Marketing Developer Platform Access"
- App must be associated with Cloud Saathi's LinkedIn Company Page
- Company page super admin must approve the app verification request
- Approval takes 2-4 weeks (not guaranteed)
- Scopes needed: `w_member_social` for posting

Since LinkedIn Marketing agent is Month 3, we have runway. Apply now so access is ready by then. Until approved, Agent 1 drafts posts → founder manually copies to LinkedIn.

Action item: Nihaan or Ankit to create LinkedIn Developer App and apply for Marketing Developer Platform access.

### Q3: Scheduling Platform
**Decision: Cal.com self-hosted on the Hetzner VPS.**

Rationale: Free. Open source. Full API access for agent integration. Self-hosting it on the same VPS is trivial (Docker container). It's a portfolio piece — "we run our own scheduling infra, we can do the same for you." Calendly's free tier is limited to 1 event type and has no API.

### Q4: Proposal Templates
**Decision: Create 2 templates from scratch in Phase 2.**

Templates needed:
1. **Fractional DevOps Retainer** — ongoing monthly engagement (scope, hours/month, SLA, pricing tiers)
2. **Infrastructure Audit** — one-time assessment (scope, deliverables, timeline, fixed price)

Both stored as Google Docs templates in a shared Drive folder. Proposal Generator agent (Agent 5) uses these as base and personalizes per lead context.

### Q5: Hetzner Region
**Decision: Singapore (sin1).**

Research findings:
- Hetzner launched Singapore DC in 2024 (colocation, not owned)
- Same CX23 specs available (2 vCPU, 4GB RAM, 40GB NVMe)
- Lowest latency to India (~30-50ms vs ~150ms from EU)
- Cloud Saathi's clients are Indian startups — latency matters for n8n webhooks, API responses, and any future client-facing dashboards

Pricing may be marginally higher than EU (~$5 vs $4) but the latency benefit justifies it.

### Q6: n8n vs Pure Python
**Decision: Hybrid — n8n for triggers + webhooks, Python for agent logic.**

Rationale:
- **n8n handles:** Cron schedules, webhook listeners, Slack event routing, visual debugging, integration connectors
- **Python handles:** Agent execution loop, LLM calls, state management, policy engine, trust levels

Why not pure Python: Nihaan is a PM, not an engineer. n8n's visual interface lets him see what triggered what, debug flows, and adjust schedules without touching code. Pure Python would make him dependent on Ankit for every operational change.

Why not pure n8n: n8n's AI nodes lack the sophistication needed for trust graduation, multi-model routing, and stateful agent orchestration. The agent brain must be code.

---

## Appendix A: Sources

- Anthropic: Building Effective Agents (2025)
- Anthropic: Measuring AI Agent Autonomy in Practice (Feb 2026)
- CrewAI: Lessons From 2 Billion Agentic Workflows (2026)
- Deloitte: Unlocking Exponential Value with AI Agent Orchestration (2026)
- Google Cloud: Agentic AI Architecture Components (2026)
- LangGraph: Interrupts & Time Travel Documentation (2026)
- Microsoft: AI Agent Orchestration Design Patterns (2025)
- Slack: Developing Approval Workflows Blueprint (2025)
- Various: AI Agent Frameworks Compared 2026 (CrewAI vs LangGraph vs AutoGen)
- Various: Cost Analysis of LLM API Pricing (March 2026)
