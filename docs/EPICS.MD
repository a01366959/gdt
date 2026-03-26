# GDT — Epics, User Stories & Tasks

## Document Control

- Version: `1.0.0`
- Status: `active`
- Last Updated: `2026-03-08`
- Owners: Product + Architecture

## How to Use This Document

- **Epics** = major feature areas aligned to MVP Build Sequence
- **User Stories** = "As a [role], I want [capability] so that [outcome]"
- **Tasks** = implementation steps under each story
- **Acceptance Criteria** = how we validate the story is done
- Status: `[ ]` not started · `[~]` in progress · `[x]` done

---

## Epic 1: Foundation & Infrastructure

> Set up monorepo, deployment pipeline, and core packages so all subsequent work has a stable base.

### Story 1.1 — Monorepo Scaffold

> As a developer, I want a working Turborepo + pnpm monorepo so that all packages build and share dependencies correctly.

Acceptance Criteria:
- `pnpm install` succeeds at root
- `turbo build` compiles all packages
- `turbo dev` starts the web app locally

Tasks:
- [x] Initialize Turborepo with pnpm workspace
- [x] Create `apps/web` (Next.js 14, App Router)
- [x] Create `packages/ui` stub
- [x] Create `packages/database` stub
- [x] Create `packages/integrations` stub
- [x] Create `packages/prompts` with all agent prompts
- [x] Create `packages/agent-tools` with typed tool registry
- [x] Create `agents/langgraph` with orchestration stubs

### Story 1.2 — Deployment Pipeline

> As a developer, I want CI/CD so that pushes to main auto-deploy frontend and agent runtime.

Acceptance Criteria:
- Vercel deploys `apps/web` on push
- Railway deploys `apps/agents` on push
- Environment secrets are configured per environment

Tasks:
- [ ] Create Vercel project linked to repo
- [ ] Create Railway project linked to repo
- [ ] Configure GitHub environments (`preview`, `production`)
- [ ] Set environment variables (Supabase, OpenRouter key)

### Story 1.3 — Supabase Project

> As a developer, I want a configured Supabase instance so that the app has auth, database, realtime, and storage ready.

Acceptance Criteria:
- Supabase project created with pgvector enabled
- Connection strings in env vars for Vercel + Railway
- RLS enabled on all tables

Tasks:
- [ ] Create Supabase project
- [ ] Enable pgvector extension
- [ ] Export URL, anon key, service role key to env vars

### Story 1.4 — Design System Bootstrap

> As a developer, I want design tokens and base components in `packages/ui` so that all UI work uses a consistent, Figma-aligned system.

Acceptance Criteria:
- Color, typography, spacing, and shadow tokens defined as CSS custom properties
- Tailwind config extends with token values
- ShadCN + Radix base components available

Tasks:
- [x] Define CSS custom properties (colors, typography, spacing, shadows, radius)
- [ ] Configure Tailwind to use custom properties
- [ ] Set up ShadCN UI with Radix primitives
- [ ] Install Font Awesome Solid icons
- [ ] Import Inter Tight font
- [x] Set `lang="es"` in root layout

---

## Epic 2: CRM Integration (Zoho Sync)

> Connect Zoho CRM as the initial data source so the app has real client, deal, and activity data.

### Story 2.1 — Zoho OAuth Connection

> As a sales manager, I want the app connected to our Zoho CRM so that our existing client data is available without re-entry.

Acceptance Criteria:
- OAuth flow completes and stores refresh token
- Token auto-refreshes on expiry
- Connection status visible in settings

Tasks:
- [ ] Register Zoho OAuth app (client ID/secret)
- [ ] Implement OAuth flow in `packages/integrations/zoho`
- [ ] Store tokens securely in Supabase (encrypted)
- [ ] Add token refresh middleware

### Story 2.2 — Zoho Read Sync

> As a sales rep, I want to see my Zoho accounts, contacts, deals, quotes, and activities in GDT so that I have a single place to work.

Acceptance Criteria:
- Accounts, Contacts, Deals, Quotes, Activities synced to Supabase
- Sync runs on schedule (every 15 min) via Railway cron
- Field mapping matches `zoho-field-mapping.json`
- Incremental sync (only changed records)

Tasks:
- [ ] Create Supabase tables matching Zoho entity mapping
- [ ] Build sync worker in `apps/agents` (Railway)
- [ ] Implement incremental sync with `Modified_Time` filter
- [ ] Map Zoho fields → Supabase columns per `zoho-field-mapping.json`
- [ ] Add sync status logging and error handling
- [ ] Schedule cron on Railway (every 15 min)

### Story 2.3 — Task Dual-Write (Supabase → Zoho)

> As a sales rep, I want tasks created in AI Sales OS to also appear in Zoho so that my manager can see them in both systems during the transition.

Acceptance Criteria:
- Tasks created in Supabase are pushed to Zoho Activities within 5 min
- `zoho_task_id` stored for deduplication
- Task updates sync bidirectionally
- Conflict resolution: last-write-wins by `updated_at`

Tasks:
- [ ] Build task sync worker (Supabase → Zoho)
- [ ] Build reverse sync (Zoho → Supabase) for tasks created in Zoho
- [ ] Add `zoho_task_id` and `zoho_synced_at` to tasks table
- [ ] Handle conflict resolution (last-write-wins)
- [ ] Test dual-write with real Zoho data

---

## Epic 3: Database Schema & Core Data Model

> Define and migrate all core tables so that agents, UI, and sync have a stable data layer.

### Story 3.1 — Core Entity Tables

> As a developer, I want typed database tables for all core entities so that the app has a reliable data layer.

Acceptance Criteria:
- Tables: `organizations`, `users`, `clients`, `contacts`, `deals`, `tasks`, `events`, `documents`
- All tables have RLS policies
- TypeScript types generated from schema

Tasks:
- [ ] Write migration: `organizations` table
- [ ] Write migration: `users` table (with Supabase Auth link)
- [ ] Write migration: `clients` table (with `segment` enum: Charal, Trucha, Atún, Tiburón, Ballena)
- [ ] Write migration: `contacts` table (FK → clients)
- [ ] Write migration: `deals` table (8 pipeline stages, FK → clients)
- [ ] Write migration: `tasks` table (with `zoho_task_id`, FK → users, clients, deals)
- [ ] Write migration: `events` table (calendar events)
- [ ] Write migration: `documents` table (quotes, PDFs)
- [ ] Generate TypeScript types from schema
- [ ] Apply RLS policies (org-scoped access)

### Story 3.2 — Conversation & Channel Tables

> As a developer, I want conversation and message tables so that agent chats and client threads are persisted.

Acceptance Criteria:
- `conversations` table with `type` ('main' | 'client_channel')
- `messages` table with sender_type, metadata (jsonb)
- Parent-child relationship for threading
- Realtime subscriptions work on messages table

Tasks:
- [ ] Write migration: `conversations` table (type, agent_id, client_id, deal_id, parent_id)
- [ ] Write migration: `messages` table (conversation_id, sender_type, content, metadata)
- [ ] Enable Realtime on `messages` table
- [ ] Add RLS: users can only see conversations in their org

### Story 3.3 — Agent Action Log

> As a sales manager, I want a log of all agent actions so that I can audit what agents did and why.

Acceptance Criteria:
- `agent_actions` table stores every action with rationale, confidence, model used, cost
- Queryable by agent, client, deal, date range

Tasks:
- [ ] Write migration: `agent_actions` table
- [ ] Include fields: agent_name, action_type, input, output, rationale, confidence, model_used, token_usage, estimated_cost, status, approved_by
- [ ] Add index on (agent_name, created_at) for fast queries

### Story 3.4 — Embeddings & Memory

> As a developer, I want a vector store so that agents can retrieve relevant context from past conversations and client history.

Acceptance Criteria:
- `embeddings` table with pgvector column
- Embedding generation pipeline (messages → vectors)
- Similarity search function available to agents

Tasks:
- [ ] Write migration: `embeddings` table (content, embedding vector(1536), source_type, source_id)
- [ ] Build embedding generation worker (message → OpenRouter embedding (text-embedding-3-small) → store)
- [ ] Create `match_embeddings` SQL function for similarity search
- [x] Define memory interfaces (WorkingMemory, ShortTermMemory, LongTermMemory)
- [x] Implement MemoryManager class

---

## Epic 4: Inbox & Conversation Threading

> Build the primary UI for reps to interact with agents and manage client conversations.

### Story 4.1 — Inbox Shell

> As a sales rep, I want an inbox view where I can see all my agent conversations and client threads so that I have one place for all work communication.

Acceptance Criteria:
- Left sidebar: list of agents with main conversations
- Client channels appear as sub-items under each agent
- Unread indicators
- Mobile responsive

Tasks:
- [ ] Build inbox layout (sidebar + conversation panel)
- [ ] Fetch conversations from Supabase (main + client_channel)
- [ ] Render conversation list grouped by agent
- [ ] Show client channels as nested items under agent
- [ ] Add unread count badges
- [ ] Implement mobile responsive layout

### Story 4.2 — Conversation View

> As a sales rep, I want to read and send messages in a conversation so that I can interact with agents naturally.

Acceptance Criteria:
- Message list with user/agent/system message styling
- Text input with send button
- Real-time message updates via Supabase Realtime
- Messages show metadata (tools used, confidence) on hover/expand

Tasks:
- [ ] Build message list component (user, agent, system variants)
- [ ] Build message input component
- [ ] Subscribe to Realtime for new messages
- [ ] Send message → POST to agent invoke endpoint
- [ ] Show agent "thinking" indicator during response
- [ ] Display proactive agent actions inline (quotes, tasks created)

### Story 4.3 — Auto-Channel Creation

> As a sales rep, I want client-specific threads to be created automatically when I mention a client, so that conversations stay organized without manual effort.

Acceptance Criteria:
- Mentioning a client name in main chat triggers entity extraction
- If no channel exists for that client + agent, one is created
- Agent response appears in the new channel
- Rep is redirected to the channel
- Channel persists and is reused on future mentions

Tasks:
- [ ] Implement entity extraction in agent pipeline (client name → `client_id`)
- [ ] Add channel creation logic: check if channel exists → create if not
- [ ] Redirect agent response to client channel
- [ ] Update sidebar to show new channel immediately (Realtime)
- [ ] Carry main chat context into channel on first creation

### Story 4.4 — Deal Room View

> As a sales rep, I want a deal-specific view that shows all conversations, tasks, and timeline for a deal so that I have full context when following up.

Acceptance Criteria:
- Deal room shows: deal info (stage, value, client), conversation threads, tasks, timeline
- Accessible from inbox or deal list
- Shows all agent interactions related to this deal

Tasks:
- [ ] Build deal room layout (header + tabs: conversations, tasks, timeline)
- [ ] Fetch deal data from Supabase (with client, contacts, tasks)
- [ ] List conversations filtered by deal_id
- [ ] Show task list filtered by deal_id
- [ ] Build timeline component (chronological agent actions + user actions)

---

## Epic 5: Agent Runtime & Orchestration

> Build the LangGraph-based agent runtime so that agents can process messages, take actions, and coordinate.

### Story 5.1 — Agent Invoke Endpoint

> As the frontend, I want an API to send messages to agents and receive responses so that the inbox can interact with the agent runtime.

Acceptance Criteria:
- `POST /api/agent/invoke` accepts `{ message, conversationId, agentName }`
- Returns structured response `{ message, actions[], metadata }`
- Writes response to Supabase (messages + agent_actions)
- Frontend receives response via Realtime subscription

Tasks:
- [ ] Build invoke endpoint in `apps/agents` (Railway)
- [ ] Implement input validation and auth check
- [ ] Connect to LangGraph workflow engine
- [ ] Write response to Supabase messages table
- [ ] Log agent action to `agent_actions` table
- [ ] Return structured response

### Story 5.2 — Intent Detection & Routing

> As the system, I want to classify user messages by intent and route to the right agent so that each agent handles what it's best at.

Acceptance Criteria:
- Intent classifier identifies: technical, sales, follow_up, reporting, supervisor, unknown
- Confidence score included in routing
- Low-confidence messages escalate to Supervisor
- Routing uses policy from `intent-routing.json`

Tasks:
- [x] Define intent routing policy (`agents/langgraph/policies/intent-routing.json`)
- [x] Build intent classifier (`agents/langgraph/src/intentClassifier.ts`)
- [x] Build workflow engine (`agents/langgraph/src/workflowEngine.ts`)
- [ ] Integrate with invoke endpoint
- [ ] Add confidence threshold for escalation to Supervisor

### Story 5.3 — Model Gateway (OpenRouter Multi-Model)

> As the system, I want a typed model gateway that routes to multiple providers via OpenRouter, with per-agent model selection for cost optimization.

Acceptance Criteria:
- `ModelGateway.run(task, input, policy)` returns `{ output, modelUsed, modelProvider, latencyMs, tokenUsage, estimatedCost }`
- Per-agent model selection (Supervisor → Mistral-7B, Sales Assistant → GPT-4o, etc.)
- Automatic failover to fallback model on timeout/error
- Cost tracking per agent + per model
- Single `OPENROUTER_API_KEY` for unified billing

Model assignments (via OpenRouter):
- **Lightweight (routing/classification):** Mistral-7B ($0.0002/1k) [Supervisor, Intent Router]
- **Standard (templates/summaries):** GPT-4o-mini or Mistral-Medium ($0.0015/1k) [Follow Up, Sales Assistant]
- **Premium (synthesis/analysis):** GPT-4o ($0.003/1k) [Sales Assistant, Technical Agent]
- **Heavy analysis:** GPT-4-Turbo or Claude-3-Opus ($0.01/1k, sparingly) [Reporting]

Tasks:
- [x] Define model routing policy (`agents/langgraph/policies/model-routing.json`)
- [ ] Build ModelGateway class in `apps/agents` with OpenRouter client
- [ ] Implement per-agent model selection with fallbacks
- [ ] Add cost tracking and model usage logging to `agent_actions`
- [ ] Add automatic retry logic with fallback model
- [ ] Create `.env` template with `OPENROUTER_API_KEY`

### Story 5.4 — Proactive Agent Execution

> As a sales rep, I want agents to automatically do the work and present results for my approval, instead of asking me if I want them to act.

Acceptance Criteria:
- Agents detect signals (stale deal, upsell opportunity, missing log) and execute immediately
- Completed artifacts (draft quotes, follow-up tasks, reactivation plans) are presented inline
- Approval-required outputs show approve/reject buttons
- Auto-executable outputs are applied immediately with a notification
- No "¿Quieres que haga X?" messages — agents DO, then present

Tasks:
- [ ] Define proactive execution policy (auto-execute vs. approval-required)
- [ ] Build action execution pipeline: detect → execute → present → gate
- [ ] Create approval UI component (approve/reject/edit buttons on agent outputs)
- [ ] Update all agent prompts to use proactive tone (execute-first language)
- [ ] Wire auto-executable actions to immediate persistence
- [ ] Wire approval-required actions to pending state until approved

---

## Epic 6: Follow Up Agent

> Detect stale deals and create follow-up reminders automatically.

### Story 6.1 — Stale Deal Detection

> As a sales rep, I want the system to automatically detect deals that haven't been touched in 5-7 days and create follow-up tasks for me.

Acceptance Criteria:
- Cron scans all active deals every 4 hours
- Deals with no activity in 5+ days are flagged
- Follow-up task auto-created with due date = today
- Task includes deal context (client, stage, last activity)
- Rep is notified in inbox (proactive message from Follow Up Agent)

Tasks:
- [ ] Build stale deal scanner (cron on Railway, every 4 hours)
- [ ] Query deals by `last_activity_at` > 5 days
- [ ] Auto-create follow-up task in Supabase (+ Zoho sync)
- [ ] Post proactive message in Follow Up Agent conversation: "Creé tarea de seguimiento para [deal]. Han pasado [N] días sin actividad."
- [ ] Respect SLA thresholds per segment (Ballenas stricter than Charales)

### Story 6.2 — Follow-Up Draft Messages

> As a sales rep, I want the agent to draft a follow-up message for me so I can send it quickly without composing from scratch.

Acceptance Criteria:
- When a follow-up task is created, agent also drafts a message
- Draft includes client context and suggested next action
- Draft is presented inline, ready to copy/send
- Sending externally requires approval

Tasks:
- [ ] Generate follow-up draft using client history + deal context
- [ ] Present draft in conversation as an actionable card
- [ ] "Aprobar y enviar" button triggers approval flow (Miriam gate)
- [ ] "Editar" button lets rep modify before sending

---

## Epic 7: Supervisor Agent

> Daily pipeline oversight, risk flagging, and workload balancing.

### Story 7.1 — Daily Pipeline Scan

> As a sales manager, I want a daily automated scan of the pipeline that flags risks, so I don't have to manually review every deal.

Acceptance Criteria:
- Runs daily at 7:00 AM (before team starts)
- Posts summary in Supervisor Agent main chat
- Flags: stale deals, high-value deals at risk, unassigned leads, overdue tasks
- Summary is actionable (links to deals, pre-created tasks)

Tasks:
- [ ] Build daily pipeline scan cron (Railway, 7:00 AM CST)
- [ ] Query all active deals, group by risk level
- [ ] Auto-create tasks for flagged items
- [ ] Post proactive summary to Supervisor conversation
- [ ] Include metrics: total pipeline value, deals at risk, overdue tasks count

### Story 7.2 — Dormant Client Detection

> As a sales manager, I want to know which clients haven't purchased or been contacted in 3+ months so we can reactivate them.

Acceptance Criteria:
- Scans clients weekly for no activity in 90+ days
- Creates reactivation opportunity (new deal in stage 1)
- Drafts reactivation outreach for the assigned rep
- Posts in Supervisor + Follow Up Agent channels

Tasks:
- [ ] Build dormant client scanner (weekly cron)
- [ ] Query clients by last purchase/contact date > 90 days
- [ ] Auto-create reactivation deal in Supabase
- [ ] Draft reactivation email/call script
- [ ] Assign to original rep (or round-robin if unassigned)
- [ ] Post proactive notification with the plan ready to execute

### Story 7.3 — Rep Activity Monitoring

> As a sales manager, I want to see which reps are falling behind on follow-ups so I can intervene early.

Acceptance Criteria:
- Dashboard shows per-rep: calls today, follow-ups completed, overdue tasks, deals touched
- Alerts when a rep has 0 activity for a full day
- Weekly comparison (this week vs last week)

Tasks:
- [ ] Build rep activity query (calls, tasks, deal touches per day)
- [ ] Create activity monitoring dashboard component
- [ ] Add alert logic: 0 activity for full business day → flag in Supervisor chat
- [ ] Weekly delta comparison

---

## Epic 8: Sales Assistant Agent

> Help reps with conversation context, summaries, and next-action suggestions.

### Story 8.1 — Conversation Summaries

> As a sales rep, I want the agent to summarize my last interactions with a client so I have context before calling them.

Acceptance Criteria:
- Rep asks "¿Qué ha pasado con [cliente]?" → agent pulls full context
- Summary includes: last contact, deal stage, recent tasks, open items
- Presented as a structured card (not a wall of text)
- All output in Spanish

Tasks:
- [ ] Build client context aggregator (conversations + tasks + deal + last activity)
- [ ] Create summary prompt (Spanish, structured output)
- [ ] Render summary as a card component in conversation
- [ ] Cache recent summaries to avoid re-computation

### Story 8.2 — Next-Action Suggestions

> As a sales rep, I want the agent to suggest what I should do next with a deal so that I always have a clear next step.

Acceptance Criteria:
- After summarizing, agent suggests 1-3 next actions
- Actions are pre-formed (e.g., "Llamar a contacto", "Enviar cotización", "Agendar reunión")
- Clicking a suggestion creates the task or opens the action
- Suggestions are context-aware (stage, segment, history)

Tasks:
- [ ] Build next-action suggestion logic (rule-based + LLM)
- [ ] Create actionable suggestion cards (clickable → create task or navigate)
- [ ] Make suggestions context-aware (pipeline stage, SLA, last activity)

### Story 8.3 — Proactive Upsell & Renewal Detection

> As a sales rep, I want the agent to automatically detect upsell and renewal opportunities and prepare quotes for me.

Acceptance Criteria:
- Agent detects: past services due for renewal, additional services client could use
- Quote is auto-prepared (based on client size, past pricing, service catalog)
- Quote presented in conversation for approval before sending
- No "¿Quieres que prepare la cotización?" — it's already prepared

Tasks:
- [ ] Build renewal detection logic (last service date + typical renewal cycle)
- [ ] Build upsell detection (services used vs services available)
- [ ] Auto-generate quote draft based on client profile
- [ ] Present quote card with approve/edit/reject actions
- [ ] On "Aprobar": route to Miriam for final approval, then send

---

## Epic 9: Reporting Agent & Dashboard

> Generate reports and show pipeline metrics without manual data collection.

### Story 9.1 — Weekly Activity Report

> As a sales manager, I want an automated weekly activity report so I don't have to compile it manually from Zoho.

Acceptance Criteria:
- Generated every Friday at 5:00 PM
- Covers: calls made, follow-ups completed, quotes sent, deals advanced, deals lost
- Per-rep breakdown
- Delivered in Reporting Agent conversation + downloadable PDF
- All in Spanish

Tasks:
- [ ] Build weekly report generator (cron, Friday 5 PM CST)
- [ ] Aggregate activity data per rep per week
- [ ] Generate structured report (text + tables)
- [ ] Post to Reporting Agent main chat
- [ ] Generate PDF version (optional, Phase 2)

### Story 9.2 — Monthly Pipeline Report

> As a sales manager, I want a monthly pipeline report showing revenue forecast, conversion rates, and deal flow.

Acceptance Criteria:
- Generated on the 1st of each month
- Covers: pipeline value, stage distribution, conversion rate, avg cycle length, top deals
- Comparison with previous month
- Delivered in Reporting Agent conversation

Tasks:
- [ ] Build monthly report generator (cron, 1st of month)
- [ ] Aggregate pipeline data (value by stage, conversion rates, cycle time)
- [ ] Generate month-over-month comparison
- [ ] Post to Reporting Agent main chat

### Story 9.3 — Reporting Dashboard UI

> As a sales manager, I want a visual dashboard showing pipeline health at a glance.

Acceptance Criteria:
- Charts: pipeline funnel, deals by stage, activity trend, revenue forecast
- Filters: by rep, by date range, by segment
- Real-time data (or near-real-time with Supabase)
- Uses Recharts or Tremor

Tasks:
- [ ] Build dashboard page layout
- [ ] Pipeline funnel chart (deals by stage)
- [ ] Activity trend chart (calls, follow-ups over time)
- [ ] Revenue chart (monthly, by segment)
- [ ] Filter controls (rep, date range, segment)
- [ ] Connect to Supabase queries

---

## Epic 10: Approval Workflows

> Gate sensitive actions behind human approval while keeping the workflow seamless.

### Story 10.1 — Approval Queue

> As a sales manager, I want a queue of pending approvals so I can quickly review and approve agent-generated outputs.

Acceptance Criteria:
- Approval queue shows all pending items (quotes, outbound messages, stage changes)
- Each item shows: what the agent did, why, the artifact (quote/message/change)
- One-click approve/reject
- Approved items are executed immediately
- Rejected items are flagged with reason

Tasks:
- [ ] Build approval queue UI (list of pending agent_actions with status = 'pending_approval')
- [ ] Show artifact preview (quote content, message draft, stage change details)
- [ ] Implement approve action → execute + update status
- [ ] Implement reject action → mark rejected + notify agent
- [ ] Add approval notification (badge count, push notification if possible)

### Story 10.2 — Quote Approval Flow

> As a sales manager, I want to review and approve quotes before they're sent to clients so I maintain pricing control.

Acceptance Criteria:
- Agent prepares quote → lands in approval queue
- Manager sees: client, services, employee count, total amount, rep
- Approve → quote is marked as sent, synced to Zoho, delivered to client
- Reject → agent receives feedback, adjusts if possible
- Edit → manager can modify before approving

Tasks:
- [ ] Build quote preview card (services, pricing, client info)
- [ ] Wire approve → update deal stage to "Quote Sent" + Zoho sync
- [ ] Wire reject → post feedback to agent conversation
- [ ] Wire edit → open inline editor, then approve modified version

### Story 10.3 — Outbound Message Approval

> As a sales manager, I want to approve external messages before they're sent so I control our client communication.

Acceptance Criteria:
- Any agent-drafted external message (email, WhatsApp text) goes through approval
- Manager sees draft + recipient + context
- Approve → message is queued for sending
- No message is sent externally without explicit approval

Tasks:
- [ ] Route all external message drafts to approval queue
- [ ] Show message preview with recipient and context
- [ ] Approve → trigger send (email API or copy-to-clipboard for WhatsApp)
- [ ] Track sent messages in agent_actions log

---

## Epic 11: Polish & Launch Readiness

> Security, performance, testing, and production deployment.

### Story 11.1 — Security Hardening

> As a system, I need RLS policies, input validation, and API key management to be production-safe.

Acceptance Criteria:
- All tables have RLS policies (org-scoped)
- API endpoints validate input and auth
- API keys rotated and stored securely
- No exposed credentials in code or logs

Tasks:
- [ ] Audit and apply RLS policies on all tables
- [ ] Add input validation on all API endpoints
- [ ] Rotate and secure all API keys
- [ ] Security review checklist pass

### Story 11.2 — User Acceptance Testing

> As the product owner, I want to validate all workflows with real GDT data before going live.

Acceptance Criteria:
- All epics pass functional testing
- GDT team uses the system for 1 week with real data
- Feedback captured and critical issues resolved
- Miriam and reps confirm it replaces their Zoho workflow for daily tasks

Tasks:
- [ ] Deploy to staging with production Supabase
- [ ] Sync real Zoho data
- [ ] Run through all user stories with test data
- [ ] Conduct UAT sessions with Miriam and 1-2 reps
- [ ] Fix critical issues from UAT feedback
- [ ] Sign-off for production deploy

### Story 11.3 — Production Deploy

> As a developer, I want the system running in production so GDT can start using it.

Acceptance Criteria:
- Production Supabase project configured
- Vercel production deployment with custom domain
- Railway production deployment
- Monitoring and alerting configured

Tasks:
- [ ] Create production Supabase project
- [ ] Point Vercel + Railway to production env vars
- [ ] Deploy to production
- [ ] Set up monitoring (error tracking, uptime)
- [ ] Monitor first week of usage

---

## Epic 12: In-Platform Calling

> Let reps make and receive PSTN calls directly from the browser, with full client context visible during the call and automatic post-call logging.

**Phase A (MVP — $0/rep):** Call Mode — context panel + `tel:` link, call happens on rep's own phone.  
**Phase B (Post-MVP — ~$14/rep/month):** Browser VoIP via SIP.js + SIP trunk, full in-browser calling with mute/hold/hangup controls.

### Story 12.1 — Call Mode (Phase A)

> As a sales rep, I want to click a contact's phone number and immediately see a full-screen context panel with talking points, deal info, and live note-taking — so I have everything I need during the call without searching.

Acceptance Criteria:
- Clicking a phone number on any contact/deal card opens Call Mode panel
- `tel:` link triggers the device's native dialer simultaneously
- Call Mode shows: contact profile, deal stage & value, segment, history, AI talking points, action buttons
- Live notes area for typing during the call
- "End Call" button triggers post-call summary flow
- Works on desktop (browser) and mobile (triggers phone app)

Tasks:
- [ ] Build `CallButton` component (phone icon on contact cards, deal rooms, inbox)
- [ ] Build `CallPanel` full-screen overlay layout (contact info + deal context + talking points + notes)
- [ ] Pre-generate talking points on deal/contact update (cache in Supabase, not on-demand)
- [ ] Build `CallNotes` live text area with auto-save
- [ ] Build quick-action buttons: schedule follow-up, update deal stage, create task
- [ ] Trigger `tel:` URI on click for native dialer fallback
- [ ] Build mobile-responsive layout for Call Mode

### Story 12.2 — Post-Call AI Summary & Auto-Logging

> As a sales rep, I want the system to summarize my call notes and automatically log the call activity so I never forget to update the CRM.

Acceptance Criteria:
- On "End Call": AI generates structured summary from rep's notes (Spanish)
- Call activity auto-created in Supabase tasks/activities (duration, summary, next steps)
- Activity synced to Zoho (dual-write)
- If deal was stale (5+ days), stale flag is auto-cleared
- Follow-up task auto-suggested based on call outcome
- Rep can edit summary before confirming

Tasks:
- [ ] Build `PostCallSummary` component (editable AI summary + suggested tasks)
- [ ] Build post-call processing worker: notes → GPT-4o-mini → structured summary
- [ ] Auto-create call activity in Supabase (duration, summary, contact_id, deal_id)
- [ ] Sync call activity to Zoho Activities via dual-write
- [ ] Auto-clear stale deal flag on call completion
- [ ] Auto-suggest follow-up task with due date
- [ ] Build confirm/edit flow before final save

### Story 12.3 — Hybrid Call Mode: Phone + Desktop Bridge (Phase B)

> As a sales rep, I want to click "Call" on my desktop and have the call happen on my phone, while my desktop captures the audio for real-time transcription, AI context, and automatic logging — so I get the best call quality from my phone with full AI assistance from the platform.

**Four tiers of implementation:**

| Tier | Approach | Cost | Recording Method |
|---|---|---|---|
| **0 — Mic-Listen** | Phone on speakerphone near laptop mic, Web Speech API transcribes | $0/rep | No recording (transcript only, free) |
| **A — Quo Bridge** | Quo API click-to-call → rings rep's phone → bridges to client | ~$23/rep/mo | Quo records + transcribes server-side |
| **B — BT Speakerphone** | BT speakerphone paired to phone + USB to computer | ~$80-150 one-time | Web Audio API captures from USB audio device |
| **C — Companion App** | Mobile app captures audio + streams to desktop via WebSocket | $0 | App forwards audio stream locally |

Acceptance Criteria:
- Clicking "Call" on desktop triggers the call on the rep's phone (not browser VoIP)
- Real-time transcript appears on the desktop during the call
- Call duration tracked, call state synced between phone and desktop
- Post-call summary flow runs automatically on hangup detection
- Desktop shows full context panel (talking points, deal info, history) during the call
- Recording captured and stored regardless of which tier is used
- Works with existing `CallPanel` UI (same layout, adapted status indicators)

**Tier A — Quo Click-to-Call (recommended MVP):**

Flow: Desktop → `POST /v1/calls` with `from: rep_phone, to: client_phone` → Quo calls rep first → rep answers → Quo bridges to client → Quo records + transcribes → webhooks stream transcript + events to desktop.

Tasks:
- [ ] Build `quoBridge.ts` in `packages/integrations/quo/` — wraps Quo phone-call API (create call, get status, get transcript)
- [ ] Implement click-to-call: `POST /v1/calls` with rep's phone number as originator
- [ ] Subscribe to Quo webhooks: `call.started`, `call.answered`, `call.ended`, `call.transcription`
- [ ] Build webhook handler on Railway (receives events, pushes to Supabase Realtime)
- [ ] Stream live transcript from Quo webhook → Supabase Realtime → `CallPanel`
- [ ] Auto-detect hangup from Quo `call.ended` event → trigger post-call summary
- [ ] Retrieve recording URL + full transcript via Quo API after call ends
- [ ] Map Quo call metadata to Supabase call log (duration, participants, recording_url, transcript)
- [ ] Add `callMode: "hybrid-quo"` UI variant to `CallPanel` (shows "Llamada en tu teléfono" status)

**Tier B — BT Speakerphone Bridge (offline alternative):**

Flow: Desktop → `tel:` URI → phone dials → rep uses BT speakerphone paired to phone — speakerphone also connected to computer via USB → Web Audio API captures both sides of conversation → audio sent to Whisper for transcription.

Compatible devices:
- Jabra Speak 750 (~$120 USD) — BT + USB, conference speakerphone
- Poly Sync 20 (~$80 USD) — BT + USB-C, portable
- Jabra Engage 55 (~$150 USD) — headset, DECT + USB, mixer mode for phone + computer
- Plantronics MDA220 (~$90 USD) — audio switch/mixer, desk phone + computer

Tasks:
- [ ] Build `audioCapture.ts` in `packages/integrations/audio/` — Web Audio API wrapper using `navigator.mediaDevices.getUserMedia()`
- [ ] Auto-detect BT/USB audio devices via `navigator.mediaDevices.enumerateDevices()`
- [ ] Build device selector UI in CallPanel settings (pick which audio input to capture)
- [ ] Stream captured audio via WebSocket to Railway transcription worker
- [ ] Build transcription pipeline: audio chunks → Whisper API (Spanish, `es`) → transcript segments
- [ ] Store recording as WAV/WebM in Supabase Storage (encrypted, org-scoped RLS)
- [ ] Detect call end from audio silence (10s threshold) or manual "End Call" button
- [ ] Add `callMode: "hybrid-device"` UI variant to `CallPanel` (shows device connection status, audio level meters)
- [ ] Show audio input level indicator during capture (confirms device is working)

**Tier C — Companion Mobile App (future):**

Flow: Rep clicks "Call" on desktop → push notification to companion app → app auto-dials client via native dialer → app captures call audio via Android `AudioRecord` API → streams audio to desktop via WebSocket → desktop transcribes.

Tasks:
- [ ] Design companion app architecture (React Native or native Android/iOS)
- [ ] Build WebSocket audio streaming protocol (opus codec, low latency)
- [ ] Handle iOS call audio restrictions (CallKit, limited to voicemail-style capture)
- [ ] Handle Android `AudioRecord` permissions for call capture
- [ ] Build push notification trigger: desktop → Supabase Realtime → companion app
- [ ] Auto-detect call start/end from phone state changes
- [ ] Pair desktop ↔ mobile via QR code during setup

### Story 12.4 — Call Recording & Transcription

> As a sales manager, I want calls optionally recorded and transcribed so that agents can extract full context without relying on rep notes.

Acceptance Criteria:
- Recording toggled per call or by org policy
- Recordings stored in Supabase Storage (encrypted at rest)
- Transcription via Quo AI (Tier A) or Whisper (Tier B/C) — Spanish
- Transcript fed to Sales Assistant Agent for richer summaries
- Consent beep/announcement played at call start (legal compliance MX)
- Recordings retained per org retention policy (default 90 days)

Tasks:
- [ ] Add recording toggle to `CallControls` (with visual indicator)
- [ ] Tier A: Retrieve recording from Quo API (`GET /v1/calls/{id}/recording`)
- [ ] Tier B: Stream audio to server-side recording worker on Railway → Whisper API
- [ ] Store recordings in Supabase Storage (encrypted, org-scoped RLS)
- [ ] Feed transcript to post-call summary for enhanced AI context
- [ ] Add consent announcement audio at call start (Mexican privacy law compliance)
- [ ] Build recording playback UI in deal room timeline
- [ ] Implement retention policy cleanup cron

### Story 12.5 — Browser VoIP Calling (Phase C — Optional)

> As a sales rep, I want to optionally make calls entirely from my browser with mute, hold, and hangup controls — for when I'm at my desk and prefer not to use my phone.

Acceptance Criteria:
- Rep clicks "Call" with VoIP mode → audio connects through browser to client's phone (PSTN)
- Call controls in UI: mute, hold, hangup, speaker toggle
- Incoming call notification with accept/reject
- Audio quality comparable to native phone
- Mexican DID as caller ID (clients see a local 722 number)
- Call duration tracked automatically
- Integrates with Call Panel from Story 12.1 (same context UI)

Tasks:
- [ ] Evaluate SIP trunk provider (VoIP.ms vs Telnyx — target ≤$0.015/min MX)
- [ ] Provision Mexican DID numbers (one per rep or shared)
- [ ] Build `sipClient.ts` in `packages/integrations/voip/` — SIP.js browser wrapper
- [ ] Build `sipGateway.ts` — WebSocket↔SIP bridge on Railway
- [ ] Build incoming call notification + accept/reject UI
- [ ] Wire call events to Supabase call log
- [ ] Add call quality monitoring (jitter, packet loss indicators)
- [ ] Configure SRTP for encrypted audio

---

## Priority Summary

| Priority | Epic | Why |
|---|---|---|
| P0 | Epic 1: Foundation | Everything depends on this |
| P0 | Epic 2: Zoho Sync | All existing data lives in Zoho |
| P0 | Epic 3: Database Schema | Agents and UI need stable data layer |
| P1 | Epic 4: Inbox & Threading | Primary user interface |
| P1 | Epic 5: Agent Runtime | Agents can't run without this |
| P1 | Epic 6: Follow Up Agent | #1 pain point (95% lost deals from poor follow-up) |
| P2 | Epic 7: Supervisor Agent | Manager visibility and oversight |
| P2 | Epic 8: Sales Assistant | Rep productivity and context |
| P2 | Epic 9: Reporting | Reporting cadence automation |
| P2 | Epic 10: Approval Workflows | Required for external actions |
| P3 | Epic 11: Polish & Launch | Final validation and deploy |
| P2 | Epic 12F: Mic-Listen Mode | $0/rep, phone on speakerphone + browser Web Speech API transcription |
| P3 | Epic 12A: Call Mode | Context panel + auto-logging ($0/rep, solves call logging pain) |
| P3 | Epic 12B: Hybrid Call — Quo Bridge | Phone-based calling + Quo server-side recording/transcription (~$23/rep) |
| P4 | Epic 12C: Hybrid Call — Device Bridge | BT speakerphone bridge + local audio capture (~$100 one-time/rep) |
| P4 | Epic 12D: Browser VoIP | Full in-browser calling (~$14/rep, optional alternative to hybrid) |
| P5 | Epic 12E: Companion Mobile App | Software-only phone bridge, Android-first ($0) |

## Validation Protocol

After completing each story:
1. Mark all tasks `[x]`
2. Verify acceptance criteria
3. Update this document
4. If scope changed, append a change-log entry to `docs/PRD.md`
