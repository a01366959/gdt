# GDT - Architecture Blueprint

## Mission

GDT workspace where humans and AI agents collaborate in one place to manage clients, deals, communications, and daily work. Replace fragmented workflows across WhatsApp, Excel, CRM, and email with a single intelligent platform.

1. **AI First** — every workflow is AI-assisted or AI-driven.
2. **Agent Collaboration** — specialized agents coordinate through an orchestrator.
3. **Human-in-the-Loop** — humans approve sensitive or external actions.
4. **Central Inbox** — all communication in one interface.
5. **Client Context Memory** — persistent memory per client/deal.
6. **Modular Architecture** — easy to add or replace agents.

---

## GDT Operations Context

### Business Model

Grupo Diagnóstico Toluca (GDT) provides clinical laboratory diagnostic services, primarily occupational health testing for companies. Revenue mix: **90% B2B**, 10% individual.

Services:
- New employee medical examinations
- Annual employee health screenings
- Regulatory occupational health tests
- On-site testing via mobile lab units
- Corporate diagnostic campaigns

Revenue model: **recurring but informal** — no formal contracts, clients issue purchase orders. Services are naturally recurring (annual screenings, new-hire testing).

### Client Segments

| Segment | Monthly Revenue (MXN) |
|---|---|
| Charales | < $10,000 |
| Truchas | $12,000 – $20,000 |
| Atunes | $20,000 – $50,000 |
| Tiburones | $100,000+ |
| Ballenas | $400,000+ |

### Sales Cycle

| Company Size | Cycle Length |
|---|---|
| Small | 2–3 weeks |
| Medium | 3–6 weeks |
| Large (corporate) | 1–3 months |

Many deals depend on budget cycles or HR planning periods.

### Sales Pipeline Stages

| # | Stage | Trigger / Definition |
|---|---|---|
| 1 | **Prospect Identified** | Company discovered via prospecting, referral, or existing relationship. Created in Zoho. |
| 2 | **First Contact** | Rep calls the company, identifies correct contact (HR, Safety, Medical). |
| 3 | **Discovery** | Rep determines: # employees, test types, on-site vs lab, frequency. |
| 4 | **Quote Preparation** | Rep builds internal expense plan in Excel (travel, lodging, food, logistics, commissions). |
| 5 | **Quote Sent** | Quote entered into Zoho and delivered to client. **Requires Miriam's approval.** |
| 6 | **Follow-up / Negotiation** | Rep follows up via calls, messages, clarifications. Most deals require multiple follow-ups. |
| 7 | **Closed Won** | Client sends Purchase Order. Sampling logistics scheduled, operations prepares service. |
| 8 | **Closed Lost** | Reasons: poor follow-up (95%), client postponed, competitor selected, budget issues. |

**Stale deal threshold:** 5–7 days without activity.

**Critical insight:** 95% of lost deals are attributed to poor follow-up.

Zoho CRM remains the current system of record for GDT sales operations. Reps and Miriam use it daily. GDT must coexist with Zoho until it fully replaces it.

### Team

| Role | Person/Count | Responsibilities |
|---|---|---|
| Sales Reps | 5 | Prospecting, calls, quotes, follow-ups, Zoho updates |
| Sales Manager | Miriam | Supervises activity, assigns follow-ups, approves quotes, forecasts revenue, monitors Zoho |
| Operations / Logistics | Team | Mobile lab deployment, sampling coordination |
| Administration / Billing | Team | Invoicing, purchase order management |

### Communication

- **Primary channel:** Phone calls (best window: 10:00–11:30 AM)
- **Secondary:** In-person visits, email, WhatsApp (likely personal accounts)
- **Language:** Spanish only
- **Typical follow-up flow:** Call → send quote via email → follow up by phone/WhatsApp

### Documents & Quotes

- Quotes built in **Excel** (expense plan: travel, lodging, food, logistics, commissions) → recreated in **Zoho** → sent as **PDF**
- Pricing is **custom per company** (depends on employee count, logistics, mobile unit usage)
- This dual entry (Excel → Zoho) is the biggest time sink for reps

### Zoho CRM — Current State

Zoho is **mandatory** for all sales activity. Already contains existing data.

| Module | Usage |
|---|---|
| Accounts | Companies |
| Contacts | Client contacts (HR, Safety, Medical) |
| Deals | Sales opportunities |
| Quotes | Generated from deals |
| Activities | Calls, commitments, tasks |

**Data quality issue:** Reps update manually throughout the day but often forget to log calls/notes, creating data gaps.

### Reporting Cadence

| Report | Frequency | Metrics |
|---|---|---|
| Activity report | Weekly | Calls, follow-ups, quotes sent |
| Revenue report | Monthly | Pipeline value, conversions, deals per rep |

Most important success factor: **consistent follow-up with clients**.

### Pain Points (Priority Order)

1. **Poor follow-up discipline** — 95% of lost deals. No system enforces persistence.
2. **Dormant clients never reactivated** — no detection for companies that haven't purchased or been contacted recently.
3. **Quote preparation overhead** — duplicate entry from Excel to Zoho.
4. **Forgotten call logging** — reps forget what clients said, miss logging in Zoho.
5. **No stale deal alerts** — follow-up timing depends on rep judgment alone.

---

## Technology Stack

| Layer | Choice |
|---|---|
| Frontend | Next.js 14 (App Router), React, TailwindCSS, ShadCN UI, Radix UI |
| Server State | React Query (TanStack Query) |
| Realtime | Supabase Realtime |
| Backend | Supabase (PostgreSQL, Auth, Realtime, Storage, Edge Functions, RLS) |
| Vector Store | pgvector (inside Supabase) |
| AI / Agents | LangGraph, OpenRouter (multi-model: GPT-4o, Claude-3, Mistral, Llama, etc. per agent) |
| Agent Tools | create_task, update_deal, send_followup, generate_report, schedule_meeting |
| File Storage | Supabase Storage |
| Charts | Recharts or Tremor |
| CRM | Zoho CRM (OAuth + REST API, scheduled sync) |
| Deployment | Vercel (frontend), Railway (agent runtime), Supabase (backend) |
| Monorepo | Turborepo + pnpm |

## Security & Data Isolation

**Row Level Security (RLS)** is enforced at the database layer on all tables. Every row is scoped to an `organization_id`, and PostgreSQL policies deny access unless explicitly permitted.

**Policy types:**
- **Org-scoped**: Users see only rows in their organization
- **Role-scoped**: Different roles (rep, manager, director) have different access levels
- **Owner-scoped**: Users can edit only their own tasks, calls, expense plans
- **Approval-gated**: Manager approval required before sensitive operations (stage changes, high-margin quotes)

**Implementation:** See [RLS_SETUP.md](/docs/RLS_SETUP.md) for complete policy templates, testing procedures, and development checklist.

**JWT claims** (included in Supabase Auth JWT):
```json
{
  "sub": "user-uuid",
  "email": "rep@gdt.com",
  "org_id": "org-uuid",
  "role": "rep|manager|operaciones|director",
  "iat": 1234567890
}
```

## Core Entities

Users, Organizations, Clients, Deals, Conversations, Messages, Threads, Channels, Tasks, Events, AgentActions, Embeddings, Documents.

## Conversation Threading & Channel Model

Every agent has a **main conversation** — an open chat where reps interact with that agent freely.

When an agent detects a specific client context inside a main conversation (by mention, CRM lookup, or deal reference), the system **auto-creates a client channel** (thread) dedicated to that client.

### Flow

1. Rep messages an agent in the main chat (e.g. "¿Qué pasa con Cervecería Toluca?").
2. Agent classifies the message → detects `client_id` via entity extraction or CRM lookup.
3. If no channel exists for that client under this agent, the system creates one.
4. The agent response is posted in the **new client channel**, and the rep is redirected there.
5. All subsequent messages about that client happen in the client channel.
6. The main chat remains a general-purpose interface.

### Data Model

```
conversations
  id          UUID PK
  type        'main' | 'client_channel'
  agent_id    FK → agents
  client_id   FK → clients (NULL for main conversations)
  deal_id     FK → deals (optional, set if channel is deal-specific)
  parent_id   FK → conversations (NULL for main, points to main for channels)
  title       text (auto-generated: "{client_name} — {agent_name}")
  created_at  timestamptz
  updated_at  timestamptz

messages
  id              UUID PK
  conversation_id FK → conversations
  sender_type     'user' | 'agent' | 'system'
  sender_id       UUID
  content         text
  metadata        jsonb (detected_client_id, intent, confidence, tool_calls)
  created_at      timestamptz
```

### Rules

- Each agent has exactly **one main conversation** per user.
- Client channels are children of the main conversation (`parent_id`).
- A client channel is scoped to one `client_id` (and optionally one `deal_id`).
- Channels persist — revisiting a client reuses the existing channel.
- Agents carry context from the main chat into the channel on creation.
- Channel list appears as a sidebar/thread panel under the agent's section.

## Task Dual-Write Strategy (Supabase + Zoho)

### Current State

Zoho CRM is the mandatory system of record for GDT sales operations. Reps and Miriam use it daily. AI Sales OS must coexist with Zoho until it can fully replace it.

### Strategy: Supabase-Primary, Zoho-Synced

1. **Tasks are created in Supabase first** — this is the source of truth for AI Sales OS.
2. **A sync worker writes tasks to Zoho Activities** — so reps who still check Zoho see them.
3. **Zoho → Supabase sync** runs on a schedule to pull tasks/activities created directly in Zoho.
4. **Conflict resolution:** Last-write-wins with `updated_at` comparison. Supabase record includes `zoho_task_id` for deduplication.

### Data Model Extension

```
tasks
  id              UUID PK
  title           text
  description     text
  status          'pending' | 'in_progress' | 'done' | 'dismissed'
  priority        'low' | 'medium' | 'high' | 'urgent'
  due_date        timestamptz
  assigned_to     FK → users
  client_id       FK → clients (optional)
  deal_id         FK → deals (optional)
  conversation_id FK → conversations (optional, links task to thread)
  created_by      'agent' | 'user'
  agent_name      text (which agent created it, if agent-created)
  zoho_task_id    text (NULL until synced to Zoho)
  zoho_synced_at  timestamptz
  created_at      timestamptz
  updated_at      timestamptz
```

### Sync Behavior

| Event | Supabase | Zoho |
|---|---|---|
| Agent creates task | ✅ Immediate | ⏳ Sync worker pushes within 5 min |
| User creates task in app | ✅ Immediate | ⏳ Sync worker pushes within 5 min |
| User creates task in Zoho | ⏳ Pulled on next sync cycle | ✅ Already there |
| Task updated in app | ✅ Immediate | ⏳ Sync worker pushes update |
| Task completed in Zoho | ⏳ Pulled on next sync cycle | ✅ Already there |

### Zoho Replacement Path

**Hard deadline: October 2026 — Zoho subscription cancelled.**

- Phase 1 (March–May): Dual-write. Both systems have tasks. Reps can use either.
- Phase 2 (June–August): AI Sales OS becomes primary UI. Zoho is background sync only. Reps stop opening Zoho.
- Phase 3 (September): Zoho sync disabled. Final data export and validation. AI Sales OS is sole platform.
- Phase 4 (October): Zoho subscription cancelled. All data archived in Supabase.

The `zoho_task_id` and `zoho_synced_at` fields make this transition seamless — when sync is turned off, the data stays intact.

## Development Principles

1. Do not rewrite architecture without clear reason.
2. Keep modules decoupled and composable.
3. Use typed interfaces.
4. Document every new module.
5. Keep prompts in `packages/prompts`.
6. Route model access through typed gateway interfaces.
7. Update `docs/PRD.md` for every material architecture/scope/workflow change.
8. Use Turborepo + pnpm for monorepo management.

## Repository Layout

```text
apps/
  web/                 # Next.js 14 (App Router)
  agents/              # Agent runtime (Next.js API routes or separate worker)
packages/
  ui/                  # ShadCN + Radix components, Tailwind tokens
  database/            # DB schema, typed queries, migrations
  integrations/        # Zoho, email, WhatsApp adapters
  prompts/             # Agent system prompts and policies
  agent-tools/         # Typed tool implementations (create_task, update_deal, etc.)
agents/
  langgraph/           # Multi-agent graph orchestration (n8n/Flowise-style)
```

Monorepo: Turborepo + pnpm

## Runtime Architecture

**Frontend (Dashboard):** `apps/web` runs on **Vercel** — Next.js 14 with ShadCN UI. Users interact with inbox, deals, tasks, dashboard.

**Agents:** `agents/langgraph/` runs on **Railway** — LangGraph (TypeScript) orchestrates specialized agents. Agents execute proactively (cron jobs for follow-ups, stale deal scans) and reactively (user messages trigger agent routing).

**Database:** Supabase PostgreSQL + Realtime. All data scoped by `organization_id` with RLS policies enforced.

### Agent Orchestration Stack

1. **LangGraph (TypeScript)** — Agent graph execution. Nodes represent specialized agents and decision points. Edges define flow.
2. **OpenRouter API** — Unified LLM access (100+ models). Per-agent model routing via `model-routing.json` for cost optimization.
3. **Typed Service Functions** — Agents interact with data via typed interfaces (`create_task`, `update_deal`, etc.), not raw SQL.
4. **Cron Scheduler** (Railway Cron) — Triggers proactive agents (Follow Up Agent every 4 hours, Supervisor Agent daily at 7 AM).
5. **Realtime Subscriptions** — Agents write to Supabase; frontend receives updates via Realtime websockets.

### Deployment Topology

```text
┌─────────────┐     REST/webhook     ┌──────────────┐
│  apps/web   │ ──────────────────▶  │ apps/agents  │
│  (Vercel)   │                      │  (Railway)   │
└──────┬──────┘                      └──────┬───────┘
       │                                    │
       │  Supabase client                   │  Supabase client
       │  + Realtime sub                    │  + OpenAI API
       │                                    │
       └──────────┐    ┌────────────────────┘
                  ▼    ▼
           ┌──────────────┐
           │   Supabase   │
           │  (Postgres,  │
           │  Auth, RT,   │
           │  Storage)    │
           └──────────────┘
```

- **Frontend → Agents:** REST calls (e.g. `POST /api/agent/invoke`) or Supabase Edge Function triggers.
- **Agents → Frontend:** Writes to Supabase; frontend receives via Realtime subscriptions.
- **Cron jobs** (Zoho sync, SLA checks, pipeline scans) run as scheduled tasks on Railway.

## Intent Detection and Agent Dispatch

Target flow:
1. `Start`
2. `DetectUserIntention` node classifies request intent.
3. Route to specialized agent node by intent.
4. Execute agent with task-specific model policy.
5. Return structured output + cost/latency telemetry.

Route table (MVP):
- `technical`: Technical Agent
- `sales`: Sales Assistant Agent
- `follow_up`: Follow Up Agent
- `reporting`: Reporting Agent
- `supervisor`: Supervisor Agent
- `unknown`: Supervisor Agent (clarify + re-route)

Node contract:
- `DetectUserIntention.input = { message, threadContext, dealContext }`
- `DetectUserIntention.output = { intent, confidence, rationale, selectedAgent }`
- `Dispatch.output = { agentName, modelPolicy, requiresApproval }`

Implemented runtime artifacts:
- `agents/langgraph/workflows/intent-router.workflow.json`
- `agents/langgraph/src/intentClassifier.ts`
- `agents/langgraph/src/workflowEngine.ts`

## Model Routing Strategy

### Decision & Rationale

**Choice:** Use OpenRouter as the LLM provider for MVP.

**Why:**
- **Multi-model access:** 100+ models from OpenAI, Anthropic, xAI, Mistral, Meta, etc.
- **Per-agent flexibility:** Route different agents through optimal models for their task type.
- **Cost savings:** 60–80% cheaper vs direct OpenAI for lightweight tasks (routing, classification).
- **Unified billing & API:** Single key, single invoice, automatic fallback on model failure.
- **Cost breakdown (100% usage):** ~$5–8/month vs $25–30/month direct OpenAI (75% savings).

### Agent Model Assignments

| Agent | Primary Model | Fallback | Cost/1k | Task | Latency |
|---|---|---|---|---|---|
| **Supervisor** | Mistral-7B-Instruct | GPT-4o-mini | $0.0002 | Routing, flagging | 1–2s |
| **Follow Up** | GPT-4o-mini | Mistral-Medium | $0.0015 | Template generation | 2–3s |
| **Sales Assistant** | GPT-4o | Claude-3.5-Sonnet | $0.003 | Context synthesis | 3–5s |
| **Intent Router** | Mistral-7B-Instruct | GPT-4o-mini | $0.0002 | Message classification | <1s |
| **Reporting** | GPT-4-Turbo | Claude-3-Opus | $0.01 | Deep analysis (rare) | 5–10s |
| **Technical** | GPT-4o | Claude-3.5-Sonnet | $0.003 | Call summaries | 3–5s |

### Cost Breakdown (Monthly Estimate)

**Assumptions:** 100 daily calls, mix of agents, 100 tokens average per call.

| Agent | Daily Calls | Tokens/Call | Est. Monthly |
|---|---|---|---|
| Supervisor | 2 | 200 | $0.24 |
| Follow Up | 15 | 500 | $0.68 |
| Sales Assistant | 20 | 400 | $0.72 |
| Intent Router | 50 | 100 | $0.03 |
| Reporting | 2x/week | 2000 | $0.16 |
| Technical | 10 | 800 | $0.72 |
| Embeddings | Daily | 500 | $0.08 |
| **TOTAL** | — | — | **$5.63/month** |

**For comparison:** Direct OpenAI (GPT-4o-mini only) = $25–30/month.

### TypeScript Contract

```typescript
interface ModelPolicy {
  agentName: string
  task: 'routing' | 'classification' | 'generation' | 'synthesis' | 'analysis'
  models: Array<{
    name: string
    provider: 'openai' | 'anthropic' | 'mistralai'
    maxLatencyMs: number
    costPer1kTokens: number
    temperature: number
    fallback?: boolean
  }>
}

interface RunResult {
  output: string
  modelUsed: string
  modelProvider: string
  latencyMs: number
  tokenUsage: { input: number; output: number }
  estimatedCost: number
}
```

### Implementation

**1. Environment Setup**

`.env.local` (development):
```env
OPENROUTER_API_KEY=sk-or-v1-... (from https://openrouter.ai/keys)
```

`.env.production` (Railway):
```env
OPENROUTER_API_KEY=sk-or-v1-... (set in Railway dashboard)
```

**2. Model Routing Configuration**

Create `agents/langgraph/policies/model-routing.json` with per-agent model policies (see [MODEL_STRATEGY.md](/docs/MODEL_STRATEGY.md#model-routing-configuration-file) for complete file).

**3. ModelGateway Class**

Implement `apps/agents/src/lib/modelGateway.ts` (see [MODEL_STRATEGY.md](/docs/MODEL_STRATEGY.md#modelgateway-typescript-class) for TypeScript implementation). This class handles:
- Sequential model attempts (primary → fallback)
- Cost calculation and tracking
- Latency measurement
- Automatic retry on failure

**4. Agent Integration**

Each agent node calls `ModelGateway.run(policy, input)`:

```typescript
const gateway = new ModelGateway(process.env.OPENROUTER_API_KEY!)
const result = await gateway.run(supervisorModelPolicy, message, systemPrompt)
```

Log the result to `agent_actions` table for cost tracking and observability.

**5. Monitoring & Cost Tracking**

Query `agent_actions` table to track costs:
```sql
SELECT agent_name, SUM(estimated_cost) as total_cost
FROM agent_actions
WHERE created_at >= NOW() - INTERVAL '1 month'
GROUP BY agent_name
ORDER BY total_cost DESC;
```

### Budget Policy

- **Default:** Use cheapest model satisfying quality threshold.
- **Escalate:** Promote to higher-cost model only on low confidence, high-value deal, or failed attempt.
- **Guardrails:**
  - Never call LLM providers directly from agent code; always route through ModelGateway.
  - Log provider/model in `agent_actions.metadata` for observability.
  - Enforce cost cap per request; return partial-safe output if exceeded.

### Available Models via OpenRouter

**Lightweight (Routing, Classification):**
- `mistral-7b-instruct` ($0.0002/1k)
- `gpt-4o-mini` ($0.0015/1k)
- `llama-2-7b` ($0.0001/1k)

**Standard (Templates, Summaries):**
- `gpt-4o-mini` ($0.0015/1k)
- `mistral-medium` ($0.00565/1k)
- `claude-3-haiku` ($0.0008/1k)

**Premium (Synthesis, Analysis):**
- `gpt-4o` ($0.003/1k)
- `claude-3-5-sonnet` ($0.003/1k)
- `gpt-4-turbo` ($0.01/1k)

**Enterprise (Deep Analysis):**
- `gpt-4-turbo` ($0.01/1k)
- `claude-3-opus` ($0.015/1k)

Full list: https://openrouter.ai/models (sorted by cost/quality)

## Agent Orchestration Contract

Each agent must expose:
- `name`
- `systemPrompt`
- `tools`
- `memoryScope`
- `inputSchema`
- `outputSchema`

Supervisor responsibilities:
- Prioritize pending events
- Delegate work to specialized agents
- Resolve conflicts between agent suggestions
- Emit `AgentAction` records with rationale and confidence

## Data Flow

1. Event enters system (new CRM update, inbound message, stale deal signal).
2. Supervisor classifies event and selects one or more agents.
3. Agent retrieves context from:
- operational DB rows
- RAG memory (pgvector)
- thread/deal history
4. Agent **executes the action immediately** (drafts quote, creates task, generates report, drafts message).
5. Policy layer classifies the completed output:
   - **auto-applied**: persisted and published immediately
   - **approval-required**: artifact stored as pending, presented to approver with approve/reject/edit controls
   - **blocked**: discarded with logged reason
6. Approved outputs are persisted and published to realtime channels.

## Proactive Execution Model

Agents follow an **execute-first, approve-output** paradigm:

- Agents **never** ask permission to work. No "¿Quieres que haga X?" messages.
- Agents detect a signal → do the full work → present the completed artifact.
- The human gate is on the **output**, not the **intent**.

### Behavior Examples

| Signal | Old (Reactive) | New (Proactive) |
|---|---|---|
| Upsell detected | "Detecté oportunidad. ¿Preparo cotización?" | Prepares quote → "Cotización lista para Cervecería Toluca — $42,000 MXN. ¿Apruebo para enviar?" |
| Deal stale 6 days | "El deal lleva 6 días sin actividad" | Creates task + drafts follow-up → "Creé tarea de seguimiento y borrador de llamada para mañana" |
| Dormant client 90d | "Este cliente no ha comprado en 3 meses" | Creates reactivation deal + drafts outreach → "Plan de reactivación listo. ¿Lo activo?" |
| Missing call log | "¿Quieres que registre la llamada?" | Logs it → "Registré la llamada con Juan (Cervecería) — seguimiento viernes" |
| Report due | "¿Genero el reporte semanal?" | Generates it → "Reporte semanal listo. 43 llamadas, 8 cotizaciones, 2 cierres" |

### Execution Policy

```
Agent detects signal
  → Execute action (draft, create, compute)
  → Classify output:
      auto-applied?     → persist + notify
      approval-required? → store as pending + present with controls
      blocked?          → discard + log
  → Post result to conversation (proactive message)
```

- **Auto-applied** actions: tasks, internal notes, summaries, risk flags, call logs, reports
- **Approval-required** actions: external sends, quotes, stage changes (Won/Lost), meeting scheduling, price modifications, medical results
- **Blocked**: anything outside agent's scope or violating org policy

Key principle: reps and Miriam review **artifacts**, not **proposals**. The agent has already done the work.

## Human-in-the-Loop Policy

Always require **Miriam's approval** for (agent prepares the artifact first, then presents for approval):
- Sending quotes to clients (quote already prepared)
- Any external outbound communication (message already drafted)
- Deal stage changes to Closed Won or Closed Lost (change already staged)
- Meeting scheduling with external attendees (invite already drafted)
- Negotiating or modifying pricing (new price already computed)
- Sending medical results (delivery already prepared)

Allow **auto-execution and immediate application** for:
- Internal summaries and conversation notes
- Follow-up reminder tasks assigned to reps
- Dormant client detection alerts + reactivation deal creation
- Internal daily/weekly activity reports
- Deal risk flagging
- Task creation for reps (rep can dismiss)
- Call/activity logging from conversation context
- Upsell/renewal quote preparation (approval required only for sending)

## Design Source of Truth (Figma -> Code)

1. Figma is the canonical source for colors, typography, spacing, radius, and component variants.
2. Sync tokens into `packages/ui` before page implementation.
3. Build UI from tokenized primitives; avoid hard-coded one-off values.
4. If MCP Figma context is available, implementation must reference it for component behavior/states.
5. Any deviation from Figma requires a documented decision in PRD change log.

---

## Hybrid Calling Architecture

### Problem

GDT reps make calls from their personal phones. Call quality on cellular is superior to browser VoIP, and reps are already comfortable with their phones. But the platform loses visibility — no transcription, no AI context during calls, no automatic logging.

### Solution: Hybrid Call Mode

The desktop provides the AI brain (context, transcript, post-call summary). The phone provides the voice pipe. A bridge connects them.

### Architecture — Three Tiers

```text
┌─────────────────────────────────────────────────────────────┐
│                    DESKTOP (apps/web)                        │
│                                                             │
│  ┌─────────────┐  ┌──────────────┐  ┌────────────────────┐ │
│  │ CallPanel    │  │ LiveTranscript│  │ PostCallSummary   │ │
│  │ (context,    │  │ (real-time   │  │ (AI summary,      │ │
│  │  deal info,  │  │  transcript  │  │  tasks, SMS)      │ │
│  │  history)    │  │  stream)     │  │                    │ │
│  └──────┬───────┘  └──────┬───────┘  └──────┬─────────────┘ │
│         │                 │                  │               │
│         └─────────┬───────┘──────────────────┘               │
│                   │                                          │
│         ┌─────────▼──────────┐                               │
│         │  Supabase Realtime  │ ◄── webhooks from Quo/worker │
│         └────────────────────┘                               │
└─────────────────────────────────────────────────────────────┘
                              │
              ┌───────────────┼───────────────┐
              │               │               │
    ┌─────────▼────┐  ┌──────▼──────┐  ┌─────▼──────────┐
    │  TIER A:     │  │  TIER B:    │  │  TIER C:       │
    │  Quo Bridge  │  │  BT Device  │  │  Companion App │
    │              │  │  Bridge     │  │                │
    │ POST /v1/    │  │ getUserMedia│  │ WebSocket      │
    │ calls →      │  │ → Whisper   │  │ audio stream   │
    │ rings rep's  │  │ → transcript│  │ → Whisper      │
    │ phone →      │  │             │  │ → transcript   │
    │ bridges to   │  │ BT Speaker  │  │                │
    │ client       │  │ ↕ Phone     │  │ Android app    │
    │              │  │ ↕ USB→PC    │  │ captures call  │
    │ Quo records  │  │             │  │ audio          │
    │ + transcribes│  │ Web Audio   │  │                │
    │ server-side  │  │ captures    │  │ iOS: limited   │
    └──────────────┘  └─────────────┘  └────────────────┘
```

### Tier A — Quo Click-to-Call (Recommended MVP)

**Flow:**
1. Rep clicks "Call" on contact card → desktop sends `POST /v1/calls` to Quo API
2. Quo calls the rep's phone first → rep answers their phone
3. Quo bridges the call to the client's number → standard phone call begins
4. Audio flows through Quo servers → Quo records both sides + runs AI transcription
5. Quo sends webhooks (`call.started`, `call.transcription`, `call.ended`) → Railway → Supabase Realtime
6. Desktop receives live transcript + call state via Realtime subscription → CallPanel updates
7. On hangup → Quo sends full transcript + AI summary → desktop runs post-call flow

**Key integration points:**
```
packages/integrations/quo/
  quoBridge.ts        — POST /v1/calls, GET /v1/calls/{id}, GET /v1/calls/{id}/transcript
  quoWebhook.ts       — Express handler for Quo webhook events on Railway
  quoMessages.ts      — POST /v1/messages (SMS follow-up after call)
```

**Cost:** ~$23/rep/month (Quo base) + per-minute PSTN.
**Pros:** No hardware, server-side recording, Quo handles transcription.
**Cons:** Per-minute cost on top of Quo subscription.

### Tier B — Bluetooth Speakerphone Bridge (Offline Alternative)

**Flow:**
1. Rep clicks "Call" → desktop opens `tel:` URI → phone's native dialer opens
2. Rep calls client from their phone using BT speakerphone for audio
3. BT speakerphone is paired to phone (BT) AND computer (USB)
4. Desktop captures audio from USB audio device via `navigator.mediaDevices.getUserMedia()`
5. Audio streamed in chunks via WebSocket to Railway → Whisper API (Spanish) → transcript segments
6. Transcript segments pushed to Supabase Realtime → CallPanel shows live transcript
7. Rep clicks "End Call" on desktop → triggers post-call summary flow

**Compatible devices:**
| Device | Price (USD) | Connection | Form Factor |
|---|---|---|---|
| Jabra Speak 750 | ~$120 | BT + USB | Conference speakerphone |
| Poly Sync 20 | ~$80 | BT + USB-C | Portable speakerphone |
| Jabra Engage 55 | ~$150 | DECT + USB | Headset with mixer |
| Plantronics MDA220 | ~$90 | 3.5mm + USB | Audio switch/mixer |

**Key integration points:**
```
packages/integrations/audio/
  audioCapture.ts     — Web Audio API wrapper, getUserMedia, device enumeration
  audioStream.ts      — WebSocket chunked audio streaming (opus/webm)
  deviceDetector.ts   — Auto-detect BT/USB devices, audio level monitoring
```

**Cost:** ~$80-150 one-time per rep, ~$0.006/min for Whisper API.
**Pros:** No monthly subscription, works offline, no VoIP dependency.
**Cons:** Requires hardware purchase, audio quality depends on device placement.

### Tier C — Companion Mobile App (Future)

**Flow:**
1. Rep clicks "Call" → push notification to companion app on rep's phone
2. Companion app auto-dials client via native dialer
3. App captures call audio via Android `AudioRecord` API
4. Audio streamed to desktop via local WiFi WebSocket
5. Desktop → Whisper → transcript → CallPanel

**Cost:** $0 (software only).
**Pros:** No hardware, free.
**Cons:** iOS heavily restricts call audio capture, Android needs special permissions, companion app maintenance.

### CallPanel Mode Selection

The `CallPanel` component accepts a `callMode` prop that adapts the UI:

```typescript
type CallMode = "mic-listen" | "voip" | "hybrid-quo" | "hybrid-device";
```

| Mode | Phone rings | Transcript source | Recording | Status indicator | Cost |
|---|---|---|---|---|---|
| `mic-listen` | No (rep dials manually) | Web Speech API (browser) | None (transcript only) | "Micrófono activo" | $0 |
| `voip` | No (browser audio) | Quo Realtime | Quo server | "Quo conectado" | ~$23/rep |
| `hybrid-quo` | Yes (rep's phone) | Quo webhook → Realtime | Quo server | "Llamada en tu teléfono" | ~$23/rep |
| `hybrid-device` | Yes (rep's phone via tel:) | Web Audio → Whisper | Local capture | "Dispositivo: Jabra Speak 750" | $80-150 once |

### Data Model Extension

```sql
-- Call log table (stores all call records regardless of tier)
call_logs
  id                UUID PK
  user_id           FK → users
  contact_id        FK → contacts
  deal_id           FK → deals (optional)
  call_mode         'mic-listen' | 'voip' | 'hybrid-quo' | 'hybrid-device'
  direction         'outbound' | 'inbound'
  started_at        timestamptz
  ended_at          timestamptz
  duration_seconds  int
  recording_url     text (Supabase Storage path or Quo URL, NULL for mic-listen)
  transcript        jsonb (array of {speaker, text, timestamp})
  ai_summary        text
  suggested_tasks   jsonb
  quo_call_id       text (NULL for mic-listen and device mode)
  audio_device      text (NULL for Quo/mic modes — e.g. "Jabra Speak 750")
  transcript_source text ('web_speech_api' | 'quo_ai' | 'whisper')
  cost_usd          numeric DEFAULT 0
  synced_to_zoho    boolean DEFAULT false
  created_at        timestamptz
```

Current implementation note:
- This repository stores the flow implementation contract from your provided design (intention node -> specialized agents).
- Direct Figma API export is not available from this environment without authenticated MCP/Figma API access.
- Once MCP access is provided, map node variants/states into `packages/ui/tokens/*` and `packages/ui/components/*` without changing the routing contract.

Implementation artifacts:
- `packages/ui/tokens/*` for design tokens
- `packages/ui/components/*` for shared building blocks
- `docs/PRD.md` change log entry for any design-level scope change

## PRD Change Governance

Rule:
- Every architecture, workflow, policy, or scope change must update `docs/PRD.md` in the same change set.

Minimum PRD update payload:
- What changed
- Why it changed
- User impact
- MVP impact (scope/time/risk)
- Owner and date

## MVP Build Sequence

1. Zoho read integration — sync Accounts, Contacts, Deals, Quotes, Activities
2. Database schema mapping Zoho entities to local tables + conversations/channels/tasks schema
3. Inbox + deal room UI shell with Zoho-synced data + conversation threading
4. Task system with dual-write to Supabase + Zoho sync worker
5. Follow Up Agent — stale deal detection (5-7 days), automated reminder tasks for reps
6. Supervisor Agent — daily pipeline scan, risk flagging, dormant client detection
7. Sales Assistant Agent — conversation summaries, suggested next actions, follow-up draft suggestions (Spanish)
8. Reporting Agent — weekly activity report, monthly pipeline report
9. Task generation + approval workflow (Miriam approves quotes/outbound)
10. Basic reporting dashboard (calls, follow-ups, quotes, conversions)
11. Phase 2 planning: Zoho write-back for deals/quotes, progressive Zoho replacement

Priority rationale: Follow-up discipline is the #1 pain point (95% of lost deals). Zoho sync must come first because all existing data lives there.

## Non-Goals for MVP

- Autonomous outbound messaging (always requires approval)
- Automated quote generation from expense plans (future)
- WhatsApp integration (future — personal accounts, not Business API yet)
- Multi-department support beyond sales
- Custom BI or forecasting models
- Automated medical results delivery
