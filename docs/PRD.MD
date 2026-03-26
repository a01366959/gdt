# GDT — Product Requirements Document (PRD)

## Document Control

- Version: `1.2.0`
- Status: `implementation-ready`
- Last Updated: `2026-03-25`
- Owners: Product + Architecture

---

## 1. Product Goal

Build an AI-native operating system for GDT's sales team (5 reps + manager) where humans and specialized agents collaborate in one workspace to manage client follow-ups, deal rooms, tasks, and reporting — replacing fragmented workflows across WhatsApp, Excel, Zoho, and email.

**Primary outcome:** eliminate the 95% deal loss rate caused by poor follow-up discipline.

**Secondary outcome:** fully replace Zoho CRM by August 2026 (rep training September 2026, subscription cancelled October).

**Tertiary outcome:** build mini-ERP for expense planning, billing (Facturella), and collections (auxiliar de pagos, mora tracking).

---

## 2. Target Users

| Role | Count | User(s) | Primary Need |
|---|---|---|---|
| Sales Rep | 5 | GDT sales team | Follow-up reminders, conversation context, task management, call logging |
| Sales Manager | 1 | Miriam | Pipeline visibility, activity monitoring, quote/stage approval, reporting |
| Operations | 2–3 | Logística team | Service scheduling, mobile unit assignment, expense tracking, status updates |
| Director | 1 | Omar | Revenue forecasting, margin analysis, cash flow tracking, factoring analysis |

---

## 3. MVP Scope

### In Scope

| Module | Description |
|---|---|
| Inbox | GDT main chat + specialist conversations + auto-created client channels (desktop + mobile PWA) |
| CRM (read/write) | Zoho sync → Supabase; dual-write deals/tasks back to Zoho until August 2026 |
| Tasks | Supabase-primary with dual-write to Zoho Activities until August |
| Agents (4) | Follow Up, Supervisor, Sales Assistant, Reporting |
| Approvals | Miriam gates external sends, quotes, stage changes; Dirección gates preplan < 30% margin, factoring |
| Call Logging | Mic-listen mode (Web Speech API, $0/rep), post-call AI summary, auto-task creation |
| **Mobile App (PWA)** | **Progressive Web App installable on iPhone/Android home screen, inbox + tasks + approvals + call logging** |
| Dashboards | 4 role-specific (Rep, Manager, Director, Operations) with live KPIs, PDF export, AI chat |
| **ERP (Mini, Phase 2)** | **Expense planning (preplan/plan/real), billing (Facturella), collections (auxiliar de pagos, mora escalation), PO remanente tracking** |
| **Reporting** | **Weekly activity report, monthly pipeline report, three-horizon pipeline model, margin analysis** |
| **Integrations** | **Zoho (until Aug), Facturella (invoicing CFDI), Medicati (medical results delivery)** |

### Out of Scope (Phase 1 MVP)

- Autonomous external outbound (always requires approval)
- Automated medical results delivery (always requires Miriam approval)
- Expense plan templating / reusable templates (Phase 2)
- Full ERP (payroll, HR, inventory) — only sales/expenses/billing/collections in MVP
- WhatsApp Business API integration (personal accounts only, post-MVP)
- Multi-department support beyond sales (Phase 2+)
- Custom BI/forecasting models beyond dashboard (Phase 3+)
- Browser VoIP (Tier A/B/C — post-MVP, currently mic-listen only)
- **Native iOS/Android apps** (PWA only; native post-MVP if needed)
- **Mobile dashboards** (lightweight KPIs on mobile; full dashboards desktop-only)
- **Mobile expense planning** (too complex for phone; desktop-only)

---

## 4. Zoho Replacement Roadmap

**Hard deadline: August 2026 — Zoho migration complete (rep training September, subscription compo cancelled October).**

| Phase | Dates | Zoho Role | GDT Workspace Role |
|---|---|---|---|
| 1 (MVP) | Mar – May 2026 | Source of truth for CRM | Primary UI for agents, tasks, inbox. Reads from Zoho. |
| 2 (Full sync) | Jun – Early Aug 2026 | Background sync only. Reps stop opening Zoho. | Full CRM + ERP managed natively. Writes back to Zoho for safety. |
| 3 (Training) | Late Aug – Sep 2026 | Read-only archive. Final export/validation. | Complete replacement. Rep training on GDT workflows. |
| 4 (Decommission) | Oct 2026 | **Cancelled.** | Sole platform. Zoho data archived in Supabase. |

### Module Replacement Checklist

| Zoho Module | GDT Equivalent | Phase Gate |
|---|---|---|
| Accounts | `clients` (with segments) | Phase 1 |
| Contacts | `contacts` (FK → clients) | Phase 1 |
| Deals | `deals` (8-stage pipeline) | Phase 1 |
| Quotes | Quote builder + approval flow | Phase 2 |
| Activities | Tasks + agent-logged activities + call logs | Phase 1 |

### Transition Gates

**Phase 1 → 2:** All 5 Zoho modules readable · rep adoption > 50% · task dual-write reliable

**Phase 2 → 3:** Rep adoption > 80% · all CRM writes in GDT · Miriam confirms Zoho redundant

**Phase 3 → 4:** All Zoho data exported + verified · 2-week parallel run, zero data loss · Miriam sign-off

---

## 5. Non-Negotiable Constraints

1. **Miriam approves** all quotes before sending to clients
2. **Miriam approves** all external outbound communication (messages, emails, results)
3. **Miriam approves** deal stage changes to Closed Won / Closed Lost
4. **Miriam approves** external meeting scheduling (calendar invites with external attendees)
5. **Miriam or Dirección (by amount)** approves preplan de gastos if margin < 30%
6. **Dirección (Omar) approves** factoring analysis and 60+ day credit terms
7. **Dirección (Omar) approves** price modifications and negotiations
8. All agent output in **Spanish (es-MX)** — no English in customer-facing messages
9. Typed TypeScript interfaces across app, agents, DB schema, API contracts
10. All prompts isolated in `packages/prompts` — no prompt logic in components or routes
11. No raw SQL in agent prompts — agents call typed service interfaces only
12. **Supabase RLS enforced on all tables** (org-scoped + role-scoped where applicable) — see [RLS_SETUP.md](/docs/RLS_SETUP.md) for implementation
13. **NDA compliance:** No cross-client AI training, complete data isolation per client
14. **Three environments minimum:** develop (Supabase staging), production (Supabase prod), plus feature branches
15. **Mobile PWA:** HTTPS only, service worker caching, 44px min touch targets, responsive viewport
16. **ShadCN/ui only** — all UI components from ShadCN preset `b2Cin5UWY`. No custom component invention. See [copilot-instructions.md](/.github/copilot-instructions.md#when-building-components)
17. **No invention** — use existing libraries/patterns, never create custom solutions when standard alternatives exist
18. **Colors from ShadCN preset only** — all colors defined in `globals.css` via ShadCN theme. No custom hex/rgb colors, no custom color schemes

---

## 6. Feature Specifications

### 6.1 — Authentication & Authorization

**Provider:** Supabase Auth (email + password for MVP; Google SSO post-MVP).

| Concept | Detail |
|---|---|
| Users | 8–10 total (5 reps + Miriam + 2–3 Operaciones + Omar). Seeded on project creation. Self-registration disabled. |
| Roles | `rep`, `manager`, `operaciones`, `director`. Stored in `users.role`. |
| Admin panel | Miriam can create/edit/deactivate users and assign roles. Route: `/admin/users` (manager-only). |
| Org isolation | All tables scoped by `organization_id`. RLS enforces per-role access. |
| Session | Supabase JWT, httpOnly cookie via Next.js middleware. Token refresh via `@supabase/ssr`. |
| Route protection | Next.js middleware protects all routes except `/login`. Role-based middleware redirects to correct dashboard. |

**Screens:**
- `/login` — email + password form. Redirect to dashboard on success (role-based redirect).
- `/admin/users` — user management (manager only)
- No signup page. Users created by Miriam via admin panel or seed script.

**Dashboard routing (post-login redirection):**
- `rep` → `/dashboard/rep`
- `manager` → `/dashboard/manager`
- `operaciones` → `/dashboard/operaciones`
- `director` → `/dashboard/director`

### 6.2 — Inbox & Conversation Threading

The inbox is the primary workspace. It replaces WhatsApp + Zoho as the single place reps work.

**Layout:**

```
┌──────┬────────────────┬──────────────────────┬──────────────────┐
│ Nav  │   Sidebar      │    Conversation       │  Thread / Detail │
│ Rail │   (agents +    │    (messages)          │  Panel           │
│      │   channels)    │                        │                  │
└──────┴────────────────┴──────────────────────┴──────────────────┘
```

- **Nav rail**: Agent icons, quick nav to Dashboard, Tasks, Settings. Call mode selector.
- **Sidebar**: GDT main chat first, then specialized agents with nested client channels. Unread badges. Search.
- **Conversation**: Message list (user/agent/system types), rich message composer, slash commands.
- **Thread panel**: Client channel detail, deal info, contact card.

**Conversation model:**
- Each user has exactly 1 main `GDT` conversation that acts as the primary entry point.
- GDT classifies each incoming message, selects the best specialist, and reuses or creates a specialist thread for execution.
- Each specialist agent has exactly 1 main conversation per user plus reusable client channels.
- When a client is detected in the GDT main chat, a client channel is auto-created or resolved under the selected specialist and the handoff is summarized back in the main chat.
- Messages have types: `user`, `agent`, `system`.
- Agent messages can contain inline actions (tasks created, quotes prepared, approval cards).

**Slash commands (MVP):**
- `/resumen [cliente]` — client context summary
- `/pipeline` — current pipeline overview
- `/reporte` — trigger activity report
- `/tarea [título]` — create task manually

**Composer:**
- Text input with send button
- `@mention` for client/contact names (triggers entity extraction)
- Slash command autocomplete
- Supports rich agent action cards inline

### 6.3 — CRM: Clients, Contacts & Deals

**Client segments** (based on annual revenue MXN):

| Segment | Revenue Range | SLA (days) | Follow-up Priority |
|---|---|---|---|
| Charales | < $10,000 | 7 | Low |
| Truchas | $12,000 – $20,000 | 5 | Medium |
| Atunes | $20,000 – $50,000 | 5 | Medium |
| Tiburones | $100,000 – $400,000 | 3 | High |
| Ballenas | $400,000+ | 2 | Critical |

**Pipeline stages** (8):

```
Prospecto Identificado → Primer Contacto → Descubrimiento →
Cotización en Preparación → Cotización Enviada → Seguimiento/Negociación →
Cerrado Ganado → Cerrado Perdido
```

**Deal classifications** (new):

| Classification | Definition | Quota Credit | Evaluation |
|---|---|---|---|
| `propia` | Rep sourced independently | 100% | Primary dimension |
| `asignada` | Assigned by Miriam | TBD (Q1)* | Secondary dimension |
| `heredada` | Inherited from previous rep | TBD (Q1)* | Track separately |

**Three-horizon pipeline model** (new):

| Horizon | Definition | Use Case |
|---|---|---|
| **Prospección** (Past) | Historical sales already completed | Rep activity KPIs, follow-up discipline |
| **Forecast** (Present) | ≥90% probability deals billing this period | Revenue forecast, calendar commitments |
| **Business Case** (Future) | All qualified opportunities in-flight | Pipeline growth tracking, future planning |

*See QUESTIONS.md Q1 for classification quota impact — impacts quota calculation engine.

**Deal room view** (route: `/deals/[id]`):
- Header: client name, segment badge, classification badge (propia/asignada/heredada), deal value, stage, assigned rep, SLA indicator, next task
- Tabs: Overview, Conversations, Tasks, Timeline, Quotes, Expenses
- Overview: quick client context, associated contacts, service details, opportunity notes
- Conversations tab: all threads related to this deal across agents
- Tasks tab: tasks linked to this deal (follow-ups, approvals pending)
- Timeline tab: chronological agent actions + user actions + call logs
- Quotes tab: preplan → plan → quote progression with approval status
- Expenses tab: (Phase 2) preplan/plan/real de gastos comparison, margin variance

**Client detail view** (route: `/clients/[id]`):
- Contact list, deal history, segment info, last activity, call history
- Quick actions: call, create task, create deal

**Contact roles:** RH, Seguridad e Higiene, Médico, Compras, Director

### 6.4 — Task System

**Source of truth:** Supabase. Zoho is synced for backward compatibility.

**Task properties:**
- `title`, `description`, `status` (pending | in_progress | done | dismissed)
- `priority` (low | medium | high | urgent)
- `due_date`, `assigned_to` (user), `client_id`, `deal_id`, `conversation_id`
- `created_by` (agent | user), `agent_name`
- `zoho_task_id`, `zoho_synced_at` (for dual-write)

**Dual-write behavior:**
| Event | Supabase | Zoho |
|---|---|---|
| Agent creates task | Immediate | Sync within 5 min |
| User creates task in app | Immediate | Sync within 5 min |
| Task created in Zoho | Pulled on next sync | Already there |
| Task updated in app | Immediate | Sync pushes update |
| Task completed in Zoho | Pulled on next sync | Already there |

**Conflict resolution:** last-write-wins by `updated_at`.

**Task list view** (route: `/tasks`):
- Filters: by assignee, status, priority, due date, client, deal
- Grouped by: today, overdue, upcoming, completed
- Quick mark done / dismiss from list

### 6.5 — Agent System

Four specialized agents, orchestrated by a supervisor pattern via LangGraph.

**Execution model:** Proactive, execute-first. Agents detect signals → do the full work → present completed artifacts. No "¿Quieres que haga X?" messages. The human gate is on the output, not the intent.

**Action classification:**
| Class | Examples | Gate |
|---|---|---|
| Auto-applied | Tasks, internal notes, summaries, risk flags, call logs, reports | Persist + notify |
| Approval-required | External sends, quotes, stage changes (Won/Lost), meeting scheduling, price mods | Store as pending → approval UI |
| Blocked | Outside agent scope or violating org policy | Discard + log |

#### 6.5.1 — Follow Up Agent

- **Trigger:** Cron every 4 hours scans active deals.
- **Signal:** Deal with no activity in 5+ days (adjusted by segment SLA).
- **Action:** Creates follow-up task with due date = today. Drafts follow-up message. Posts proactively in agent conversation.
- **Model:** GPT-4o-mini (lightweight automation).

#### 6.5.2 — Supervisor Agent

- **Trigger:** Daily cron at 7:00 AM CST. Weekly cron for dormant clients.
- **Signals:** Stale deals, high-value deals at risk, unassigned leads, overdue tasks, clients with no activity in 90+ days.
- **Action:** Posts daily pipeline summary. Creates reactivation deals for dormant clients. Flags risks. Monitors rep activity (alerts on 0-activity days).
- **Model:** GPT-4o-mini for routing/classification, escalate to GPT-4o on low confidence.

#### 6.5.3 — Sales Assistant Agent

- **Trigger:** User messages ("¿Qué ha pasado con [cliente]?"), deal context requests, upsell signals.
- **Action:** Generates client context summaries as structured cards. Suggests 1–3 next actions. Detects upsell/renewal opportunities and auto-prepares quotes.
- **Model:** GPT-4o for drafting, GPT-4o-mini for summaries.

#### 6.5.4 — Reporting Agent

- **Trigger:** Weekly cron (Friday 5 PM), monthly cron (1st of month), on-demand via `/reporte`.
- **Action:** Generates weekly activity report (calls, follow-ups, quotes, deals per rep). Generates monthly pipeline report (value, conversion, cycle time). Posts to conversation.
- **Model:** GPT-4.1 for deep analysis, GPT-4o-mini for quick summaries.

**Intent routing:**

```
User message in GDT main chat → IntentClassifier (GPT-4o-mini)
  → { intent, confidence, rationale, selectedAgent }
  → if confidence < 0.7: escalate to Supervisor
  → Reuse or create specialist thread
  → Dispatch to specialized agent with model policy
```

Route table:
| Intent | Agent |
|---|---|
| `technical` | Technical Agent |
| `sales` | Sales Assistant |
| `follow_up` | Follow Up Agent |
| `reporting` | Reporting Agent |
| `supervisor` | Supervisor Agent |
| `unknown` | Supervisor (clarify + re-route) |

### 6.6 — Approval Workflows

**Approval queue** (accessible to `manager` role, badge count in nav).

**Approval item:**
- What the agent did (action summary)
- Why (rationale, confidence score)
- The artifact (quote content, message draft, stage change details)
- Approve / Reject / Edit buttons
- Approved → immediate execution. Rejected → feedback to agent.

**Approval-required actions:**
- Sending quotes to clients
- Any external outbound communication
- Deal stage → Closed Won or Closed Lost
- Meeting scheduling with external attendees
- Price negotiation or modification

### 6.7 — Call Logging (Mic-Listen Mode)

**MVP tier: Mic-Listen ($0/rep)**

Rep places their phone on speakerphone near their laptop mic. The browser captures audio via Web Speech API and generates a live Spanish transcript. No recording — transcript only.

**Flow:**
1. Rep clicks phone icon next to a contact → CallPanel opens as full-screen overlay
2. CallPanel shows: contact info, deal context, call history, talking points
3. Rep dials client on their phone (manual or `tel:` link)
4. Web Speech API captures and transcribes in real-time (es-MX)
5. Rep types notes during the call
6. Rep clicks "Colgar" → transcript + notes sent to `/api/call-summary`
7. LLM generates structured summary, follow-up tasks, and suggested WhatsApp message
8. Rep can send follow-up via WhatsApp Web deep link (pre-filled)
9. Call log saved to `call_logs` table

**CallPanel modes** (future tiers have adapted UI):
| Mode | Source | Cost |
|---|---|---|
| `mic-listen` | Web Speech API | $0 |
| `hybrid-quo` | Quo webhooks → Realtime | ~$23/rep |
| `hybrid-device` | Web Audio → Whisper | $80-150 once |
| `voip` | Quo real-time | ~$23/rep |

### 6.8 — Reporting Dashboard

**Route:** `/dashboard`

**Charts:**
- Pipeline funnel (deals by stage)
- Activity trend (calls, follow-ups over time — Recharts line chart)
- Revenue by segment (Recharts bar chart)
- Rep activity comparison (calls, tasks, deals per rep)

**Filters:** by rep, date range, segment.

**Data source:** Supabase queries. Near-real-time via Realtime subscription on aggregation triggers.

---

## 7. Database Schema

All tables in Supabase PostgreSQL with RLS. `organization_id` scopes every row.

### 7.1 — Organizations & Users

```sql
CREATE TABLE organizations (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name          TEXT NOT NULL,
  slug          TEXT UNIQUE NOT NULL,
  created_at    TIMESTAMPTZ DEFAULT now()
);

CREATE TABLE users (
  id              UUID PRIMARY KEY REFERENCES auth.users(id),
  organization_id UUID NOT NULL REFERENCES organizations(id),
  email           TEXT NOT NULL,
  full_name       TEXT NOT NULL,
  role            TEXT NOT NULL CHECK (role IN ('rep', 'manager', 'operaciones', 'director')),
  avatar_url      TEXT,
  phone           TEXT,
  is_active       BOOLEAN DEFAULT true,
  created_at      TIMESTAMPTZ DEFAULT now(),
  updated_at      TIMESTAMPTZ DEFAULT now()
);
```

### 7.2 — CRM (Clients, Contacts, Deals)

```sql
CREATE TYPE client_segment AS ENUM ('charales', 'truchas', 'atunes', 'tiburones', 'ballenas');
CREATE TYPE deal_stage AS ENUM (
  'prospecto_identificado', 'primer_contacto', 'descubrimiento',
  'cotizacion_preparacion', 'cotizacion_enviada', 'seguimiento_negociacion',
  'cerrado_ganado', 'cerrado_perdido'
);
CREATE TYPE deal_classification AS ENUM ('propia', 'asignada', 'heredada');

CREATE TABLE clients (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  organization_id UUID NOT NULL REFERENCES organizations(id),
  name            TEXT NOT NULL,
  segment         client_segment NOT NULL,
  industry        TEXT,
  phone           TEXT,
  website         TEXT,
  city            TEXT,
  state           TEXT,
  employee_count  INT,
  service_type    TEXT,
  notes           TEXT,
  zoho_account_id TEXT,
  zoho_created_at TIMESTAMPTZ,
  zoho_updated_at TIMESTAMPTZ,
  last_activity_at TIMESTAMPTZ,
  created_at      TIMESTAMPTZ DEFAULT now(),
  updated_at      TIMESTAMPTZ DEFAULT now()
);

CREATE TABLE contacts (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  organization_id UUID NOT NULL REFERENCES organizations(id),
  client_id       UUID NOT NULL REFERENCES clients(id),
  first_name      TEXT NOT NULL,
  last_name       TEXT NOT NULL,
  email           TEXT,
  phone           TEXT,
  mobile          TEXT,
  department      TEXT,
  title           TEXT,
  contact_role    TEXT CHECK (contact_role IN ('rh', 'seguridad_higiene', 'medico', 'compras', 'director')),
  zoho_contact_id TEXT,
  zoho_created_at TIMESTAMPTZ,
  zoho_updated_at TIMESTAMPTZ,
  created_at      TIMESTAMPTZ DEFAULT now(),
  updated_at      TIMESTAMPTZ DEFAULT now()
);

CREATE TABLE deals (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  organization_id UUID NOT NULL REFERENCES organizations(id),
  client_id       UUID NOT NULL REFERENCES clients(id),
  assigned_to     UUID REFERENCES users(id),
  title           TEXT NOT NULL,
  stage           deal_stage NOT NULL DEFAULT 'prospecto_identificado',
  classification  deal_classification NOT NULL DEFAULT 'propia',
  value           NUMERIC(12,2),
  currency        TEXT DEFAULT 'MXN',
  expected_close  DATE,
  probability     INT CHECK (probability BETWEEN 0 AND 100),
  lost_reason     TEXT,
  notes           TEXT,
  zoho_deal_id    TEXT,
  zoho_created_at TIMESTAMPTZ,
  zoho_updated_at TIMESTAMPTZ,
  last_activity_at TIMESTAMPTZ,
  created_at      TIMESTAMPTZ DEFAULT now(),
  updated_at      TIMESTAMPTZ DEFAULT now()
);
```

### 7.3 — Tasks

```sql
CREATE TABLE tasks (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  organization_id UUID NOT NULL REFERENCES organizations(id),
  title           TEXT NOT NULL,
  description     TEXT,
  status          TEXT NOT NULL DEFAULT 'pending' CHECK (status IN ('pending', 'in_progress', 'done', 'dismissed')),
  priority        TEXT NOT NULL DEFAULT 'medium' CHECK (priority IN ('low', 'medium', 'high', 'urgent')),
  due_date        TIMESTAMPTZ,
  assigned_to     UUID REFERENCES users(id),
  client_id       UUID REFERENCES clients(id),
  deal_id         UUID REFERENCES deals(id),
  conversation_id UUID REFERENCES conversations(id),
  created_by      TEXT NOT NULL CHECK (created_by IN ('agent', 'user')),
  agent_name      TEXT,
  zoho_task_id    TEXT,
  zoho_synced_at  TIMESTAMPTZ,
  created_at      TIMESTAMPTZ DEFAULT now(),
  updated_at      TIMESTAMPTZ DEFAULT now()
);
```

### 7.4 — Conversations & Messages

```sql
CREATE TABLE conversations (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  organization_id UUID NOT NULL REFERENCES organizations(id),
  type            TEXT NOT NULL CHECK (type IN ('main', 'client_channel')),
  agent_id        TEXT NOT NULL,
  user_id         UUID REFERENCES users(id),
  client_id       UUID REFERENCES clients(id),
  deal_id         UUID REFERENCES deals(id),
  parent_id       UUID REFERENCES conversations(id),
  title           TEXT,
  last_message_at TIMESTAMPTZ,
  created_at      TIMESTAMPTZ DEFAULT now()
);

CREATE TABLE messages (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  conversation_id UUID NOT NULL REFERENCES conversations(id),
  sender_type     TEXT NOT NULL CHECK (sender_type IN ('user', 'agent', 'system')),
  sender_id       UUID REFERENCES users(id),
  agent_name      TEXT,
  content         TEXT NOT NULL,
  metadata        JSONB DEFAULT '{}',
  parent_id       UUID REFERENCES messages(id),
  created_at      TIMESTAMPTZ DEFAULT now()
);

-- Enable Realtime on messages
ALTER PUBLICATION supabase_realtime ADD TABLE messages;
```

### 7.5 — Agent Actions

```sql
CREATE TABLE agent_actions (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  organization_id UUID NOT NULL REFERENCES organizations(id),
  agent_name      TEXT NOT NULL,
  action_type     TEXT NOT NULL,
  status          TEXT NOT NULL DEFAULT 'completed' CHECK (status IN ('completed', 'pending_approval', 'approved', 'rejected', 'blocked')),
  input           JSONB,
  output          JSONB,
  rationale       TEXT,
  confidence      NUMERIC(3,2),
  model_used      TEXT,
  token_usage     JSONB,
  estimated_cost  NUMERIC(8,6),
  approved_by     UUID REFERENCES users(id),
  approved_at     TIMESTAMPTZ,
  client_id       UUID REFERENCES clients(id),
  deal_id         UUID REFERENCES deals(id),
  conversation_id UUID REFERENCES conversations(id),
  created_at      TIMESTAMPTZ DEFAULT now()
);

CREATE INDEX idx_agent_actions_agent_date ON agent_actions(agent_name, created_at);
CREATE INDEX idx_agent_actions_status ON agent_actions(status) WHERE status = 'pending_approval';
```

### 7.6 — Call Logs

```sql
CREATE TABLE call_logs (
  id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  organization_id   UUID NOT NULL REFERENCES organizations(id),
  user_id           UUID NOT NULL REFERENCES users(id),
  contact_id        UUID REFERENCES contacts(id),
  deal_id           UUID REFERENCES deals(id),
  call_mode         TEXT NOT NULL CHECK (call_mode IN ('mic-listen', 'voip', 'hybrid-quo', 'hybrid-device')),
  direction         TEXT DEFAULT 'outbound' CHECK (direction IN ('outbound', 'inbound')),
  started_at        TIMESTAMPTZ NOT NULL,
  ended_at          TIMESTAMPTZ,
  duration_seconds  INT,
  recording_url     TEXT,
  transcript        JSONB,
  ai_summary        TEXT,
  suggested_tasks   JSONB,
  follow_up_message TEXT,
  transcript_source TEXT CHECK (transcript_source IN ('web_speech_api', 'quo_ai', 'whisper')),
  quo_call_id       TEXT,
  audio_device      TEXT,
  cost_usd          NUMERIC(8,4) DEFAULT 0,
  synced_to_zoho    BOOLEAN DEFAULT false,
  created_at        TIMESTAMPTZ DEFAULT now()
);
```

### 7.7 — Embeddings (RAG Memory)

```sql
CREATE EXTENSION IF NOT EXISTS vector;

CREATE TABLE embeddings (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  organization_id UUID NOT NULL REFERENCES organizations(id),
  content     TEXT NOT NULL,
  embedding   vector(1536),
  source_type TEXT NOT NULL CHECK (source_type IN ('message', 'call_log', 'task', 'deal_note')),
  source_id   UUID NOT NULL,
  created_at  TIMESTAMPTZ DEFAULT now()
);

CREATE INDEX idx_embeddings_vector ON embeddings USING ivfflat (embedding vector_cosine_ops) WITH (lists = 100);

CREATE OR REPLACE FUNCTION match_embeddings(
  query_embedding vector(1536),
  match_threshold FLOAT,
  match_count INT,
  org_id UUID
) RETURNS TABLE (id UUID, content TEXT, source_type TEXT, source_id UUID, similarity FLOAT)
LANGUAGE plpgsql AS $$
BEGIN
  RETURN QUERY
  SELECT e.id, e.content, e.source_type, e.source_id,
    1 - (e.embedding <=> query_embedding) AS similarity
  FROM embeddings e
  WHERE e.organization_id = org_id
    AND 1 - (e.embedding <=> query_embedding) > match_threshold
  ORDER BY e.embedding <=> query_embedding
  LIMIT match_count;
END;
$$;
```

### 7.8 — Documents & Quotes

```sql
CREATE TABLE documents (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  organization_id UUID NOT NULL REFERENCES organizations(id),
  type            TEXT NOT NULL CHECK (type IN ('quote', 'contract', 'report', 'other')),
  title           TEXT NOT NULL,
  client_id       UUID REFERENCES clients(id),
  deal_id         UUID REFERENCES deals(id),
  status          TEXT DEFAULT 'draft' CHECK (status IN ('draft', 'pending_approval', 'approved', 'sent', 'accepted', 'rejected')),
  content         JSONB,
  file_url        TEXT,
  total_amount    NUMERIC(12,2),
  currency        TEXT DEFAULT 'MXN',
  approved_by     UUID REFERENCES users(id),
  approved_at     TIMESTAMPTZ,
  sent_at         TIMESTAMPTZ,
  zoho_quote_id   TEXT,
  created_at      TIMESTAMPTZ DEFAULT now(),
  updated_at      TIMESTAMPTZ DEFAULT now()
);
```

### 7.8b — ERP: Expense Planning & Quotes (New)

```sql
CREATE TYPE expense_plan_type AS ENUM ('preplan', 'plan', 'real');

CREATE TABLE expense_plans (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  organization_id UUID NOT NULL REFERENCES organizations(id),
  deal_id         UUID NOT NULL REFERENCES deals(id),
  type            expense_plan_type NOT NULL,
  parent_id       UUID REFERENCES expense_plans(id),  -- preplan → plan → real chain
  
  -- Service details
  service_days    INT,
  employee_count_confirmed INT,
  service_type    TEXT,
  
  -- Line items
  travel_cost     NUMERIC(10,2),
  lodging_cost    NUMERIC(10,2),
  food_cost       NUMERIC(10,2),
  external_logistics_cost NUMERIC(10,2),
  internal_labor_cost  NUMERIC(10,2),
  commission_pct  NUMERIC(5,2),
  
  -- Calculations
  total_cost      NUMERIC(12,2),
  estimated_revenue NUMERIC(12,2),
  gross_margin_pct NUMERIC(5,2),
  
  -- Metadata
  created_by      UUID REFERENCES users(id),
  approved_by     UUID REFERENCES users(id),
  approved_at     TIMESTAMPTZ,
  status          TEXT DEFAULT 'draft' CHECK (status IN ('draft', 'pending_approval', 'approved', 'executed')),
  variance_notes  TEXT,
  
  created_at      TIMESTAMPTZ DEFAULT now(),
  updated_at      TIMESTAMPTZ DEFAULT now()
);

CREATE INDEX idx_expense_plans_deal ON expense_plans(deal_id, type);
```

### 7.8c — ERP: Billing & Collections (New)

```sql
CREATE TABLE invoices (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  organization_id UUID NOT NULL REFERENCES organizations(id),
  deal_id         UUID REFERENCES deals(id),
  client_id       UUID NOT NULL REFERENCES clients(id),
  
  -- CFDI metadata (Facturella)
  folio           TEXT,
  cfdi_uuid       TEXT UNIQUE,
  rfc_cliente     TEXT,
  rfc_emisor      TEXT,
  
  -- Invoice details
  amount          NUMERIC(12,2) NOT NULL,
  currency        TEXT DEFAULT 'MXN',
  concept         TEXT NOT NULL,
  issued_at       TIMESTAMPTZ NOT NULL,
  status          TEXT DEFAULT 'pendiente' CHECK (status IN ('pendiente', 'pagado', 'cancelada', 'sustituida')),
  
  -- Payment terms
  credit_days     INT DEFAULT 30,
  due_date        TIMESTAMPTZ,
  
  -- Collections
  factoring_flag  BOOLEAN DEFAULT false,
  factoring_cost  NUMERIC(8,6),
  
  -- Files
  pdf_url         TEXT,
  xml_url         TEXT,
  
  -- Sync
  zoho_invoice_id TEXT,
  facturella_status TEXT,
  
  created_at      TIMESTAMPTZ DEFAULT now(),
  updated_at      TIMESTAMPTZ DEFAULT now()
);

CREATE TABLE payments (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  organization_id UUID NOT NULL REFERENCES organizations(id),
  invoice_id      UUID NOT NULL REFERENCES invoices(id),
  
  folio           TEXT,
  empresa         TEXT,
  ejecutivo       TEXT,
  
  factura_date    TIMESTAMPTZ,
  credit_days     INT,
  vencimiento_date TIMESTAMPTZ,
  
  estado          TEXT DEFAULT 'pendiente' CHECK (estado IN ('pendiente', 'pagado', 'mora', 'en_disputa')),
  days_overdue    INT GENERATED ALWAYS AS (
    EXTRACT(DAY FROM NOW() - vencimiento_date)
  ) STORED,
  
  collection_task_id UUID REFERENCES tasks(id),
  
  created_at      TIMESTAMPTZ DEFAULT now(),
  updated_at      TIMESTAMPTZ DEFAULT now()
);

CREATE TABLE purchase_orders (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  organization_id UUID NOT NULL REFERENCES organizations(id),
  client_id       UUID NOT NULL REFERENCES clients(id),
  
  po_number       TEXT NOT NULL,
  total_amount    NUMERIC(12,2),
  consumed_amount NUMERIC(12,2) DEFAULT 0,
  remanente       NUMERIC(12,2) GENERATED ALWAYS AS (
    total_amount - COALESCE(consumed_amount, 0)
  ) STORED,
  
  issued_at       TIMESTAMPTZ,
  expiry_date     TIMESTAMPTZ,
  expired         BOOLEAN GENERATED ALWAYS AS (
    expiry_date < NOW()
  ) STORED,
  
  status          TEXT CHECK (status IN ('active', 'expired', 'closed')),
  
  created_at      TIMESTAMPTZ DEFAULT now(),
  updated_at      TIMESTAMPTZ DEFAULT now()
);

CREATE INDEX idx_purchase_orders_expiry ON purchase_orders(expiry_date) WHERE NOT expired;
```

### 7.8d — Operational: Service Scheduling & Medical Results (New)

```sql
CREATE TABLE service_events (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  organization_id UUID NOT NULL REFERENCES organizations(id),
  deal_id         UUID NOT NULL REFERENCES deals(id),
  client_id       UUID NOT NULL REFERENCES clients(id),
  
  -- Scheduling
  scheduled_date  TIMESTAMPTZ,
  scheduled_end   TIMESTAMPTZ,
  location        TEXT,
  confirmed_employee_count INT,
  
  -- Resources
  mobile_unit_id  UUID,
  assigned_personnel TEXT ARRAY,
  
  -- Status tracking
  status          TEXT DEFAULT 'scheduled' CHECK (status IN ('scheduled', 'en_proceso', 'completado', 'cancelled')),
  started_at      TIMESTAMPTZ,
  completed_at    TIMESTAMPTZ,
  
  -- Notes
  operational_notes TEXT,
  
  approved_by     UUID REFERENCES users(id),
  
  created_at      TIMESTAMPTZ DEFAULT now(),
  updated_at      TIMESTAMPTZ DEFAULT now()
);

CREATE TABLE medical_results (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  organization_id UUID NOT NULL REFERENCES organizations(id),
  client_id       UUID NOT NULL REFERENCES clients(id),
  service_event_id UUID REFERENCES service_events(id),
  
  -- Results
  medicati_request_id TEXT,
  results_url     TEXT,
  result_files    JSONB,  -- { lab_id, file_urls, completion_date }
  
  -- Delivery
  status          TEXT DEFAULT 'pending' CHECK (status IN ('pending', 'pending_approval', 'approved', 'delivered', 'rejected')),
  approved_by     UUID REFERENCES users(id),
  approved_at     TIMESTAMPTZ,
  delivered_at    TIMESTAMPTZ,
  
  -- Sync
  zoho_activity_id TEXT,
  
  created_at      TIMESTAMPTZ DEFAULT now(),
  updated_at      TIMESTAMPTZ DEFAULT now()
);
```

### 7.9 — Calendar Events

```sql
CREATE TABLE events (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  organization_id UUID NOT NULL REFERENCES organizations(id),
  title           TEXT NOT NULL,
  description     TEXT,
  start_at        TIMESTAMPTZ NOT NULL,
  end_at          TIMESTAMPTZ NOT NULL,
  location        TEXT,
  event_type      TEXT CHECK (event_type IN ('call', 'meeting', 'follow_up', 'internal')),
  user_id         UUID REFERENCES users(id),
  client_id       UUID REFERENCES clients(id),
  deal_id         UUID REFERENCES deals(id),
  is_external     BOOLEAN DEFAULT false,
  zoho_event_id   TEXT,
  created_at      TIMESTAMPTZ DEFAULT now(),
  updated_at      TIMESTAMPTZ DEFAULT now()
);
```

---

## 8. API Endpoints

### 8.1 — Agent Runtime (`apps/agents` on Railway)

| Method | Path | Description | Auth |
|---|---|---|---|
| POST | `/api/agent/invoke` | Send message to agent, get response | Bearer (Supabase JWT) |
| POST | `/api/agent/webhook/quo` | Receive Quo call webhooks | Quo signature |
| GET | `/api/agent/health` | Health check | None |

**`POST /api/agent/invoke`**

```typescript
// Request
{
  message: string;
  conversationId: string;
  entryPoint?: 'main_chat' | 'agent_thread';
  agentName?: string;       // optional — auto-detected via intent
  metadata?: {
    clientId?: string;
    dealId?: string;
  };
}

// Response
{
  message: string;
  actions: Array<{
    type: 'task_created' | 'deal_updated' | 'quote_prepared' | 'report_generated';
    data: Record<string, unknown>;
    status: 'auto-applied' | 'pending_approval';
  }>;
  metadata: {
    agentName: string;
    routedVia: 'gdt-main' | 'direct';
    handoffMode: 'redirect_to_agent_thread' | 'stay_in_current_thread';
    targetThreadMode: 'reuse_or_create' | 'current_thread';
    intent: string;
    confidence: number;
    modelUsed: string;
    tokenUsage: { prompt: number; completion: number };
    estimatedCost: number;
    latencyMs: number;
  };
}
```

### 8.2 — Web App API Routes (`apps/web` on Vercel)

| Method | Path | Description | Auth |
|---|---|---|---|
| POST | `/api/call-summary` | Generate AI call summary from transcript | Bearer |
| GET | `/api/clients` | List clients (paginated, filterable) | Bearer |
| GET | `/api/clients/[id]` | Get client detail with contacts + deals | Bearer |
| GET | `/api/deals` | List deals (paginated, filterable) | Bearer |
| GET | `/api/deals/[id]` | Get deal detail with tasks + timeline + expenses | Bearer |
| GET | `/api/tasks` | List tasks (filterable by assignee, status) | Bearer |
| POST | `/api/tasks` | Create task | Bearer |
| PATCH | `/api/tasks/[id]` | Update task status/details | Bearer |
| GET | `/api/conversations` | List conversations for current user | Bearer |
| GET | `/api/conversations/[id]/messages` | Get messages for conversation | Bearer |
| POST | `/api/conversations/[id]/messages` | Send message (triggers agent invoke) | Bearer |
| GET | `/api/approvals` | List pending approvals (manager only) | Bearer |
| POST | `/api/approvals/[id]/approve` | Approve an agent action | Bearer |
| POST | `/api/approvals/[id]/reject` | Reject an agent action | Bearer |
| **GET** | **`/api/dashboard/rep`** | **Rep dashboard: my deals, tasks, calls, KPIs** | **Bearer** |
| **GET** | **`/api/dashboard/manager`** | **Manager dashboard: pipeline, rep activity, approvals** | **Bearer** |
| **GET** | **`/api/dashboard/director`** | **Director dashboard: revenue, margins, cash flow, forecasts** | **Bearer** |
| **GET** | **`/api/dashboard/operaciones`** | **Operations dashboard: service events, mobile units (Phase 2)** | **Bearer** |
| **POST** | **`/api/dashboard/[role]/chat`** | **Chat with Reporting Agent for current dashboard** | **Bearer** |
| **GET** | **`/api/dashboard/[role]/export-pdf`** | **Export current dashboard to PDF** | **Bearer** |
| GET | `/api/quotes/preplan` | Get preplan expense data for deal | Bearer |
| POST | `/api/quotes/preplan` | Create preplan de gastos | Bearer |
| GET | `/api/quotes/plan` | Get plan expense data (post-preplan) | Bearer |
| POST | `/api/quotes/plan` | Create plan de gastos (requires preplan) | Bearer |
| POST | `/api/quotes/generate` | Generate and send quote from approved plan | Bearer |
| GET | `/api/invoices` | List invoices (manager + operations only) | Bearer |
| POST | `/api/invoices` | Generate invoice (Facturella) via Facturella API | Bearer |
| GET | `/api/payments/auxiliary` | List auxiliar de pagos with mora status | Bearer |
| GET | `/api/purchase-orders` | List POs with remanente tracking | Bearer |

### 8.3 — Cron Jobs (Railway Scheduled Tasks)

| Schedule | Job | Description | Phase |
|---|---|---|---|
| Every 15 min | `zoho-sync` | Incremental Zoho → Supabase sync (deals, activities, contacts) | Phase 1 |
| Every 5 min | `task-sync` | Supabase tasks → Zoho Activities push (dual-write) | Phase 1 |
| Every 4 hours | `stale-deal-scan` | Follow Up Agent: flag deals with no activity > SLA | Phase 1 |
| Daily 7:00 AM CST | `daily-pipeline-scan` | Supervisor: pipeline summary, risk flags, overdue tasks | Phase 1 |
| Weekly Monday | `dormant-client-scan` | Supervisor: clients with no activity > 90 days | Phase 1 |
| Friday 5:00 PM CST | `weekly-report` | Reporting Agent: weekly activity report | Phase 1 |
| 1st of month | `monthly-report` | Reporting Agent: monthly pipeline + revenue report | Phase 1 |
| **Daily 8:00 AM CST** | **`payment-mora-check`** | **Supervisor: check invoices entering mora, create collection tasks** | **Phase 2** |
| **Weekly Friday** | **`po-remanente-alert`** | **Flag POs expiring within 30 days** | **Phase 2** |
| **Every 15 min** | **`facturella-sync`** | **Sync invoice status updates from Facturella webhooks** | **Phase 2** |
| **On webhook** | **`medicati-results`** | **Receive medical results, store, create approval task** | **Phase 2** |

---

## 9. Technology Stack

| Layer | Technology |
|---|---|
| **Frontend (Dashboard)** | **Next.js 14 (App Router), React 18, TypeScript** |
| Styling | Tailwind CSS, CSS custom properties (HSL tokens) |
| **Components** | **ShadCN UI exclusively** (preset `b2Cin5UWY` — no custom components) |
| Icons | Font Awesome 7 (Solid) |
| Font | Inter Tight |
| State / Fetch | React Query (TanStack Query v5) |
| Charts | Recharts |
| Auth | Supabase Auth (`@supabase/ssr`) |
| **Database** | **Supabase PostgreSQL + pgvector + Row Level Security (RLS)** |
| Realtime | Supabase Realtime (websockets for live inbox/tasks) |
| Storage | Supabase Storage (call recordings, documents) |
| **Agents** | **LangGraph (TypeScript) + OpenRouter (100+ models: GPT-4o, Claude-3, Mistral, Llama)** |
| **Agent Routing** | **Per-agent model selection** (Supervisor: Mistral-7B $0.0002/1k, Sales Assistant: GPT-4o $0.003/1k, Follow Up: GPT-4o-mini $0.0015/1k, Reporting: GPT-4 Turbo $0.01/1k) → 75% savings vs direct OpenAI |
| **Mobile** | **PWA (Progressive Web App)** — installable on iOS/Android home screen, offline support via Service Worker + IndexedDB |
| Monorepo | Turborepo + pnpm |
| Frontend deploy | Vercel (Next.js dashboard) |
| Agent deploy | Railway (LangGraph services + cron jobs for proactive agents) |
| Transcription | Web Speech API (MVP, free), Whisper (post-MVP) |

---

## 10. Repository Structure

```
apps/
  web/                    # Next.js 14 — deployed on Vercel
    app/
      (auth)/login/       # Login page
      (app)/              # Authenticated routes (inbox, deals, tasks, dashboard)
      api/                # API routes (call-summary, CRUD)
    components/
      ui/                 # ShadCN base components
      call/               # CallPanel, CallControls, etc.
      inbox/              # MessageList, Composer, ThreadPanel
      crm/                # ClientCard, DealRoom, ContactList
      dashboard/          # Charts, filters, KPI cards
      approvals/          # ApprovalQueue, ApprovalCard
    lib/
      supabase/           # Supabase client (browser + server)
      utils.ts
    types/
      speech.d.ts         # Web Speech API type declarations
  agents/                 # Agent runtime — deployed on Railway
    src/
      invoke.ts           # POST /api/agent/invoke handler
      cron/               # Scheduled jobs (zoho-sync, stale-deal, reports)
      gateway/            # ModelGateway (typed LLM interface)
packages/
  database/               # Schema, migrations, typed queries
    migrations/           # SQL migration files (numbered)
    types.ts              # Generated TypeScript types from schema
    queries/              # Typed query functions
  integrations/
    zoho/                 # OAuth, sync worker, field mapping
    quo/                  # Call API (post-MVP)
  prompts/                # Agent system prompts (.md files)
  agent-tools/            # Typed tool implementations
    types.ts              # Tool registry types
  ui/                     # Shared design tokens + components
agents/
  langgraph/              # LangGraph orchestration
    src/                  # IntentClassifier, WorkflowEngine, MemoryManager
    policies/             # intent-routing.json, model-routing.json
    workflows/            # Workflow graph definitions
docs/
  ARCHITECTURE.md
  PRD.md
  EPICS.md
  TASKS.md
```

---

## 11. Environment Variables

```bash
# Supabase
NEXT_PUBLIC_SUPABASE_URL=
NEXT_PUBLIC_SUPABASE_ANON_KEY=
SUPABASE_SERVICE_ROLE_KEY=          # Server-side only (Railway + API routes)

# OpenAI
OPENAI_API_KEY=
OPENAI_BASE_URL=https://api.openai.com/v1   # or compatible endpoint
OPENAI_MODEL=gpt-4o-mini                     # default model

# Zoho CRM
ZOHO_CLIENT_ID=
ZOHO_CLIENT_SECRET=
ZOHO_REFRESH_TOKEN=
ZOHO_API_BASE_URL=https://www.zohoapis.com

# Railway (agent runtime)
RAILWAY_AGENT_URL=                  # Used by web app to invoke agents

# Optional
QUO_API_KEY=                        # Post-MVP calling
QUO_WEBHOOK_SECRET=
```

---

## 12. Implementation Phases

### Phase 1A — Foundation (Week 1-2)

1. Supabase project + run all migrations
2. Auth: login page, middleware, session management
3. Seed data: organization, 6 users, sample clients/contacts/deals from Zoho
4. Supabase client setup (`@supabase/ssr` for Next.js)
5. Basic layout shell (nav rail, sidebar, conversation area)

### Phase 1B — Data Layer (Week 2-3)

1. Zoho OAuth flow in `packages/integrations/zoho`
2. Zoho → Supabase read sync worker (15-min cron on Railway)
3. CRUD API routes for clients, contacts, deals, tasks
4. React Query hooks for all data fetching
5. Real data rendering in inbox sidebar (agents + client channels)

### Phase 1C — Conversations & Agents (Week 3-5)

1. Conversation + message tables wired to inbox UI
2. Real message sending: composer → API route → Supabase → Realtime subscription
3. Agent invoke endpoint on Railway (receives message, runs LangGraph)
4. ModelGateway implementation (task-based model selection, cost logging)
5. Intent classifier integrated with invoke endpoint
6. Follow Up Agent: stale deal scanner + task creation
7. Supervisor Agent: daily pipeline scan + risk flags
8. Sales Assistant Agent: client summary + next actions
9. Reporting Agent: weekly/monthly reports
10. Proactive agent messages (auto-post to conversations)

### Phase 1D — Tasks & Approvals (Week 5-6)

1. Task list view with filters
2. Task creation from UI + from agents
3. Task dual-write sync (Supabase → Zoho)
4. Approval queue UI (manager only)
5. Approval flow: approve → execute, reject → feedback

### Phase 1E — Call Logging & Dashboard (Week 6-7)

1. CallPanel wired to real data (contacts, deals from Supabase)
2. Call logs saved to `call_logs` table on hangup
3. Post-call summary generates real tasks + updates `last_activity_at`
4. Reporting dashboard with pipeline, activity, revenue charts
5. Dashboard data API routes

### Phase 1F — Polish & Launch (Week 7-8)

1. RLS audit on all tables
2. Error handling + loading states across all views
3. Mobile responsive pass
4. Zoho sync validation with real data
5. UAT with Miriam and 1-2 reps

---

## 13. Performance Requirements

| Metric | Target |
|---|---|
| Page load (LCP) | < 2s |
| Agent response (simple) | < 3s |
| Agent response (complex) | < 10s |
| Realtime message delivery | < 500ms |
| Zoho sync cycle | < 60s per run |
| Dashboard data load | < 2s |
| Supabase query (indexed) | < 100ms |

---

## 14. Security Requirements

1. **RLS on all tables** — every query scoped by `organization_id` from JWT
2. **No raw SQL in agent prompts** — agents use typed service interfaces only
3. **API key rotation** — secrets in Vercel/Railway env vars, never in code
4. **Input validation** — Zod schemas on all API route inputs
5. **CORS** — Vercel only allows requests from app domain
6. **Rate limiting** — agent invoke endpoint rate-limited per user (10 req/min)
7. **Audit trail** — all agent actions logged in `agent_actions` with cost + model
8. **Encrypted storage** — Supabase Storage for recordings (encrypted at rest)
9. **No credential logging** — API keys, tokens never appear in logs or responses

---

## 15. Risks and Tradeoffs

| Risk | Mitigation |
|---|---|
| Zoho API rate limits during sync | Incremental sync with `Modified_Time` filter. Backoff on 429. |
| Agent hallucination on client data | RAG retrieval from verified Supabase data, not free generation. Confidence thresholds. |
| Web Speech API accuracy (es-MX) | Mic-listen is for logging context, not legal transcripts. Notes supplement transcript. |
| LLM cost overrun | Per-request cost cap, cheapest model by default, promote only on low confidence. |
| Zoho Oct 2026 deadline pressure | Phase gates enforce incremental delivery. Each phase is independently useful. |
| Single LLM provider dependency | OpenAI has highest uptime. ModelGateway abstraction allows future provider swap. |

---

## 16. Success Metrics (Phase 1)

| Metric | Current | Target |
|---|---|---|
| Deal follow-up rate | ~5% (manual) | > 80% (agent-prompted) |
| Avg deal response time | 5-7 days | < 2 days |
| Calls logged in CRM | ~10% | > 90% (auto-logged) |
| Weekly report generation | 3-4 hours manual | < 1 min (automated) |
| Rep daily active usage | 0 (system doesn't exist) | > 80% of reps daily |

---

## Change Log

### 2026-03-17 — v1.1.0: GDT Branding + Main Chat Handoff
- Change: Rebranded the product surface from AI Sales OS to GDT in the main docs and demo-facing UI. Updated the inbox spec so GDT is the primary chat entry point that classifies requests, reuses or creates specialist threads, and summarizes the handoff back to the user.
- Why: The demo needs clearer branding and a more intuitive first-run experience where users start in one chat and are redirected into the correct specialist flow without ambiguity.
- User Impact: Reps now start from a single obvious GDT conversation and then land in the right specialist thread with context already prepared.
- MVP Impact: Improves demo clarity and lowers UX confusion without expanding functional scope; introduces a small routing-contract change for main-chat handoff metadata.
- Owner: Product + Architecture

### 2026-03-09 — v1.0.0: Implementation-Ready PRD
- Change: Complete rewrite from governance-focused v0.2.0 to implementation-ready v1.0.0. Added: full database schema (10 tables), API endpoint specifications (20+ endpoints), feature specs per module (auth, inbox, CRM, tasks, agents, approvals, calling, dashboard), implementation phases (6 phases over 8 weeks), environment variables, performance requirements, security requirements, success metrics.
- Why: Moving from UI prototyping to real implementation requires concrete schemas, contracts, and phase plans — not just governance rules this was before.
- User Impact: Engineers can start building directly from this document. Every table, endpoint, and feature is specified with enough detail to implement without ambiguity.
- MVP Impact: High clarity gain. Unblocks all implementation workstreams in parallel.
- Owner: Product + Architecture

### 2026-03-08 — v0.2.0: Initial Governance + Operational Lock-In
- Change: Original PRD covering product goal, target users, MVP scope, Zoho roadmap, constraints, model policy, and operational context integration.
- Owner: Architecture

---

## PRD Update Protocol (Mandatory)

For every material change, append one change-log entry with:
1. Change summary
2. Rationale
3. User impact
4. MVP impact (scope, timeline, risk)
5. Owner and date
