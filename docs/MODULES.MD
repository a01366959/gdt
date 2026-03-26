# GDT — Module Build Order, Agents, Flows & API Contracts

**Purpose:** Define the implementation sequence for all modules with concrete sections, agent interactions, chat flows, and API endpoints. This is the working spec for Copilot.

**Version:** 2.0.0  
**Last Updated:** 2026-03-25

---

## Architecture Summary

| Component | Technology | Deployment | Purpose |
|---|---|---|---|
| **Frontend (Dashboard)** | Next.js 14 (App Router), React 18, TypeScript | Vercel | Inbox, deals, tasks, dashboards. User-facing UI. |
| **Agents** | LangGraph (TypeScript) + OpenRouter (100+ models) | Railway | Specialized agents: Supervisor, Follow Up, Sales Assistant, Reporting, Intent Router. Proactive + reactive. |
| **Database** | Supabase PostgreSQL + RLS + Realtime + pgvector | Supabase Cloud | System of record. All data scoped by org_id with row-level security. |
| **Components** | ShadCN/ui (preset b2Cin5UWY) | Included in Next.js | 24+ preloaded components. No custom components allowed. |
| **Mobile** | Progressive Web App (PWA) | Same Vercel domain | Installable on iPhone/Android home screen. Service Worker + offline support. |

**Execution model:**
- Frontend (Vercel): Handles UI, receives Realtime updates from database
- Agents (Railway): Run LangGraph orchestration, execute proactive jobs (cron), create tasks/approvals
- Database (Supabase): All reads/writes gated by RLS. Agents + frontend both use same Supabase client

---

## Overview

GDT is built in **5 phases**. Each phase introduces specific modules with detailed agent integration, database changes, and API contracts. Within a phase, independent modules can be built in parallel.

```
Phase 0 — Scaffold & Infrastructure (1 day, no code yet)
  ↓
Phase 1 — Authentication & Core Schema (3–4 days)
  ↓
Phase 2 — Conversations & Inbox (4–5 days)
  ↓
Phase 3 — CRM Core + Quote Management (5–6 days)
  ↓
Phase 4 — Agents & Automation (6–8 days)
  ↓
Phase 5 — Reporting + ERP Modules (7–10 days)
```

---

## UI/Component Standards (All Phases)

**Critical constraint:** All UI components must use **ShadCN/ui exclusively**. No custom component invention.

**Component library:** ShadCN preset `b2Cin5UWY` (installed in TASKS.md Step 4) includes:
- Layout: Card, Separator, Sidebar, NavigationMenu
- Input: Button, Input, Textarea, Select, Checkbox, RadioGroup, Switch, Combo Box
- Display: Badge, Avatar, Alert, Progress, Skeleton, Spinner
- Feedback: Tooltip, Popover, AlertDialog, Dialog, Sheet
- Charts: Recharts (area, bar, line, pie, scatter, radar)
- Data: Table, Tabs, Accordion, Pagination, Breadcrumb
- Navigation: Dropdown Menu

**Rule:** If a component need isn't covered by ShadCN preset, DON'T build custom. Instead:
1. Check ShadCN docs (https://ui.shadcn.com/)
2. If not in preset, ask Copilot to add via `shadcn-ui add <component>` during the phase that needs it
3. Custom components are not permitted

**Styling:** Tailwind CSS only. No inline styles, no CSS modules. Use preset theme colors defined in `globals.css`.

**Colors rule:** DO NOT invent custom hex/rgb colors. All colors come from ShadCN preset theme (e.g., `bg-slate-100`, `text-blue-600`, `border-gray-300`). Check `globals.css` for available palette.

See [copilot-instructions.md#when-building-components](/.github/copilot-instructions.md#when-building-components) for detailed rules + examples.

---

## Mobile Features by Phase

### PWA Strategy

**GDT mobile is a Progressive Web App (PWA)** — a web app installable on iPhone and Android home screens. Not a native app — web installed via "Share > Add to Home Screen."

**Why PWA?**
- Single codebase (same Next.js, no iOS/Android dev needed)
- Instant updates (no app store review delays)
- Offline support (Service Worker caches critical pages)
- Home screen icon (feels like native app)
- Push notifications (iOS 16.4+, Android full support)
- Camera/mic access (call logging works on mobile)

**Target:** iPhone 12+, Android 10+

### Feature Coverage by Phase

**GDT mobile is feature-gated. Not all features are mobile-optimized:**

| Feature | Phase | Desktop | Mobile | Mobile UX | Notes |
|---|---|---|---|---|---|
| **Inbox & Chat** | 2 | FULL | FULL | Full-width messages, realtime | PRIMARY mobile feature |
| **Task List** | 2 | FULL | SIMPLE | Swipe-to-mark-done, cards | PRIMARY mobile feature |
| **Call Logging** | 2 | OPTIONAL | PRIMARY | Mic-listen, one-tap record | Designed for mobile first |
| **Deal Status** | 3 | FULL EDIT | READ-ONLY | Card view, tap to expand | No editing on mobile |
| **Approvals** | 4 | FULL | CARD STACK | Swipe left=reject, right=approve | CRITICAL for Miriam |
| **Dashboards** | 5 | FULL (4 roles) | HIDDEN | Too complex for mobile | Desktop-only |
| **Expense Planning** | 5 | FULL FORM | HIDDEN | Multi-step form unsupported | Desktop-only |
| **Quote Builder** | 3 | FULL EDITOR | HIDDEN | Too complex, small screen | Desktop-only |
| **Reporting** | 5 | FULL CHARTS | MINIMAL | Simple text summary only | Desktop-only for full view |
| **Settings** | 1 | FULL | MINIMAL | Profile + password only | Rare use on mobile |

### PWA Setup (Phase 2)

**1. Service Worker** (`apps/web/public/sw.js`)

Handles offline caching and push notifications:
- Cache-first for static assets (UI bundle, icons)
- Network-first for API calls (fail gracefully offline)
- Push listener for notifications (stale deals, follow-ups, approvals)

**2. Web App Manifest** (`apps/web/public/manifest.json`)

Declares app identity and shortcuts:
```json
{
  "name": "GDT — Sales Operating System",
  "short_name": "GDT",
  "start_url": "/inbox",
  "display": "standalone",
  "theme_color": "#059669",
  "icons": [/* 72px-512px sizes, maskable support */],
  "shortcuts": [
    { "name": "Inbox", "url": "/inbox" },
    { "name": "Tasks", "url": "/tasks" },
    { "name": "Log Call", "url": "/call" }
  ]
}
```

**3. Manifest Registration** (`apps/web/app/layout.tsx`)

```typescript
<head>
  <link rel="manifest" href="/manifest.json" />
  <link rel="apple-touch-icon" href="/icon-192x192.png" />
  <meta name="theme-color" content="#059669" />
  <meta name="apple-mobile-web-app-capable" content="yes" />
</head>
```

**4. Service Worker Registration** (`useEffect` in `app/layout.tsx`)

```typescript
navigator.serviceWorker.register('/sw.js', { scope: '/' })
```

### Mobile Routes & Navigation (Phase 2)

| Route | Purpose | Mobile First? |
|---|---|---|
| `/` | Redirects to `/inbox` on mobile | YES |
| `/inbox` | Main chat interface, full-width | YES |
| `/tasks` | Task list, swipe-to-mark-done | YES |
| `/deals/[id]` | Deal detail (read-only) | NO (Phase 3) |
| `/approvals` | Approval card stack | YES (Phase 4) |
| `/call` | Call logging UI | YES |
| `/profile` | User settings (minimal) | YES |
| `/dashboard`, `/reporting`, `/admin` | Redirected to `/inbox` on mobile | HIDDEN |

**Middleware** (`apps/web/middleware.ts`) detects mobile and enforces routing:
```typescript
if (isMobile && desktopOnlyRoutes.includes(pathname)) {
  return NextResponse.redirect(new URL('/inbox', request.url))
}
```

### Touch-Optimized Components (Phase 2)

**1. Task Swipe Component** — `TaskSwipeCard`
- Swipe right = mark done
- Swipe left = dismiss
- 44px minimum touch target (Apple guidelines)

**2. Approval Card Stack** — `ApprovalCard`
- Display one approval at a time
- Swipe right = approve
- Swipe left = reject
- Smooth card transitions

**3. Message Bubbles** — Full-width, sticky composer at bottom
- Auto-scroll to latest message
- Keyboard-aware layout

**4. Inbox Navigation**
- Sticky header with agent selector
- Back button to inbox list
- Call button (always accessible)

**Tailwind Config** (`tailwind.config.ts`): Add mobile breakpoints and `touch-*` spacing:
```typescript
screens: {
  'mobile-sm': '375px',  // iPhone SE
  'mobile-md': '390px',  // iPhone 12
  'mobile-lg': '428px',  // iPhone Pro Max
},
spacing: {
  'touch': '44px',  // Min touch target (Apple)
}
```

### Offline Support (Phase 2)

**IndexedDB Storage** (`offlineStorage.ts`):
- Cache last 20 messages
- Cache last 5 tasks
- Cache last 3 deals viewed
- Syncs automatically when connection restored

**Offline Indicator** (`<OfflineIndicator />`)
- Yellow banner: "📡 Offline Mode — Some features unavailable"
- Appears only when `navigator.onLine === false`
- Automatically hides when online

**Network Strategy**:
- API calls: Use `fetch(...).catch(() => getFromIndexedDB())`
- Messages: Display locally, queue for sync when online
- Tasks: Display cached, disable create/edit when offline

### Push Notifications (Phase 4)

**Request Permission** — On login:
```typescript
const permission = await Notification.requestPermission()
const subscription = await serviceWorkerRegistration.pushManager.subscribe({ ... })
// Send subscription endpoint to backend
```

**Notification Types**:
| Trigger | User | Platform | Action |
|---|---|---|---|
| Stale deal (4h+) | All reps | iOS/Android | Tap → open deal |
| Follow-up task due | All reps | iOS/Android | Tap → open task |
| Approval pending | Miriam only | iOS/Android | Tap → approval card stack |
| Call summary ready | Assigned rep | iOS/Android | Tap → open inbox |

### Testing Checklist

- [ ] iPhone SE (375px): text readable, no overlay
- [ ] iPhone 12 (390px): standard test device
- [ ] iPhone 12 Pro Max (428px): large screen upscaling
- [ ] Samsung Galaxy (720px): Android platform
- [ ] Lighthouse Mobile score > 85
- [ ] Offline mode: disable network, verify cache loads
- [ ] Touch targets: min 44x44px on all buttons
- [ ] PWA install: "Add to Home Screen" works iOS/Android
- [ ] Service Worker: check DevTools → Application → Service Workers
- [ ] Push notifications: send test, verify arrives
- [ ] Swipe gestures: test task/approval swiping
- [ ] Performance: < 3s load time on 3G

### Deployment

**HTTPS Required** — PWA requires HTTPS everywhere (including local dev):
- Vercel: auto-enables HTTPS ✅
- Local: use `local-ssl-proxy` or ngrok

**Icon Generation** — Generate all sizes from source 512x512 PNG:
```bash
npm install --save-dev sharp
npx node scripts/generate-icons.js public/logo-512.png
```

**PWA Validation in CI** — Add to GitHub Actions:
```yaml
- name: Validate PWA
  run: npx lighthouse https://staging.gdt.app/inbox --chrome-flags="--headless" --view
```

---

## Database Relationship Map

**Complete foreign key relationships across all modules:**

| Table | References | Referenced By | Purpose |
|---|---|---|---|
| **organizations** | auth.users | users, clients, contacts, deals, tasks, conversations, messages, agent_actions, calls, expense_plans, quotes, invoices, payments, purchase_orders, service_events, medical_results, mora_escalation, dashboard_recommendations | Tenant isolation root |
| **users** | organizations, auth.users | tasks, deals, calls, agent_actions (approver), conversations_users, payments (escalation alerts) | Authentication + roles |
| **clients** | organizations | contacts, deals, tasks, conversations, calls, expense_plans, invoices, payments, medical_results | CRM parent entity |
| **contacts** | organizations, clients | calls, medical_results | Client relationships |
| **deals** | organizations, clients, users (assigned_to) | tasks, conversations, calls, expense_plans, quotes, invoices, payments, purchase_orders, medical_results | Sales cycle root |
| **tasks** | organizations, users (assigned_to), clients, deals, conversations | agent_actions | Task management |
| **conversations** | organizations, clients, deals | messages, conversations_users | Thread root |
| **messages** | organizations, conversations, users (sender_id) | — | Thread content |
| **conversations_users** | conversations, users | — | User participation |
| **agent_actions** | organizations, users (approved_by), tasks | approvals | Audit log + approval gate |
| **calls** | organizations, deals, contacts, users (called_by) | — | Call history |
| **expense_plans** | deals, parent_id (self-ref) | expense_plan_line_items | Preplan/plan/real chain |
| **expense_plan_line_items** | expense_plans | — | Line item breakdown |
| **quotes** | organizations, deals, expense_plans | invoices | Quote lifecycle |
| **invoices** | organizations, deals, quotes | payments | Invoicing |
| **payments** | organizations, invoices, deals | mora_escalation | Payment tracking |
| **purchase_orders** | organizations, deals | — | PO tracking |
| **service_events** | organizations, deals, contacts | medical_results | Operations scheduling |
| **medical_results** | organizations, deals, service_events, contacts | — | Results delivery |
| **mora_escalation** | organizations, payments | — | Collections escalation |
| **approvals** | organizations, agent_actions, users (approver_id), users (requested_by) | — | Approval queue |
| **zoho_sync_log** | — | — | Integration audit log |

---

### Key Relationship Patterns:

**1. Organization Isolation (Multi-Tenant)**
- Every table has `org_id` — enforces data isolation via RLS policies
- All queries must filter by `org_id`

**2. User Ownership**
- `deals` → `assigned_to` (FK to users)
- `tasks` → `assigned_to` (FK to users)
- `calls` → `called_by` (FK to users)
- `agent_actions` → `approved_by` (FK to users, nullable)
- `approvals` → `approver_id` + `requested_by` (FK to users)

**3. Sales Cycle Chain**
- `clients` ← parent
  - `contacts` (subordinate, many-to-one)
  - `deals` (subordinate, many-to-one)
    - `tasks` (subordinate, many-to-one)
    - `conversations` (subordinate, many-to-one)
    - `calls` (subordinate, many-to-one)
    - `expense_plans` (subordinate, many-to-one)
      - `expense_plan_line_items` (child items)
    - `quotes` (subordinate, many-to-one)
      - `invoices` (generated from quotes, many-to-one)
        - `payments` (many-to-one)
          - `mora_escalation` (many-to-one, one mora per payment level)
    - `purchase_orders` (subordinate, many-to-one)
    - `service_events` (subordinate, many-to-one)
      - `medical_results` (many-to-one)

**4. Conversation Threading**
- `conversations` ← parent
  - `messages` (many-to-one + many users per conversation via conversations_users)
  - `conversations_users` (many-to-many join: tracks participation)

**5. Approval Gate**
- `agent_actions` → pending/approved/rejected
- `approvals` table → explicit approval card
- `approver_id` (FK to users) → who approved

**6. Zoom In: Three-Tier Expansion from Deal**
```
deal
  ├─ expense_plans (preplan) → self-ref → (plan) → self-ref → (real)
  │   └─ expense_plan_line_items
  ├─ quotes (generated from plan) → invoices → payments → mora_escalation
  ├─ purchase_orders
  ├─ service_events → medical_results
  ├─ tasks
  ├─ calls
  └─ conversations → messages
```

---

## Agent Model Routing Strategy

**All agents use OpenRouter for cost optimization, with per-agent model selection:**

| Agent | Primary Model | Fallback | Cost/1k | Task |
|---|---|---|---|---|
| **Supervisor** | Mistral-7B-Instruct | GPT-4o-mini | $0.0002 | Detect stale deals, flag risks, route tasks |
| **Follow Up** | GPT-4o-mini | Mistral-Medium | $0.0015 | Template generation, follow-up message drafts |
| **Sales Assistant** | GPT-4o | Claude-3.5-Sonnet | $0.003 | Context synthesis, upsell detection, quotes |
| **Intent Router** | Mistral-7B-Instruct | GPT-4o-mini | $0.0002 | Classify message intent, route to agent |
| **Reporting** | GPT-4-Turbo | Claude-3-Opus | $0.01 | Deep analysis, complex aggregations (use sparingly) |
| **Technical** | GPT-4o | Claude-3.5-Sonnet | $0.003 | Call transcript summarization, long-context |

**Key principles:**
1. **Lightweight tasks** (routing, classification) → Mistral-7B ($0.0002/1k) saves ~80% vs GPT-4o-mini
2. **Standard tasks** (templates, summaries) → GPT-4o-mini or Mistral-Medium ($0.0015/1k)
3. **Complex tasks** (synthesis, analysis) → GPT-4o ($0.003/1k, balanced)
4. **Heavy analysis only** → GPT-4-Turbo/Claude-Opus ($0.01/1k, use 1x/week max)
5. **All fallbacks** → Automatic retry if primary times out or errors

**Cost estimate:**
- Supervisor runs 2 times/day, ~200 tokens each → $0.00008/day
- Sales Assistant runs 15x/day, ~500 tokens each → $0.0225/day
- Reporting Agent runs 2x/week, ~2000 tokens → $0.04/week
- **Total estimated:** $3–5/month (vs $20–30/month with direct OpenAI)

**Implementation:**
- Environment variable: `OPENROUTER_API_KEY=sk-or-...` (from https://openrouter.ai/keys)
- Config file: `agents/langgraph/policies/model-routing.json` (per-agent model selection)
- TypeScript: `apps/agents/src/lib/modelGateway.ts` (decides which model to use)

---

## Phase 0: Scaffold & Infrastructure

**Duration:** 1 day  
**Deliverable:** Next.js + ShadCN + folder structure + docs ready  
**Owner:** Manual (you), then Copilot

| Module | Task | Status | Notes |
|---|---|---|---|
| Monorepo setup | Run Turborepo + pnpm init | Manual | See TASKS.md Phase 0.5 for step-by-step |
| Next.js init | Create apps/web with App Router | Manual | Use create-next-app, TypeScript, Tailwind |
| ShadCN install | Install all components upfront | Manual | Button, Input, Card, Form, Table, Sheet, Dialog, etc. |
| Folder structure | Create all standard folders | Manual | Copy from TASKS.md Phase 0.5 |
| Docs import | Copy all docs/ files to repo | Manual | ARCHITECTURE.md, PRD.md, EPICS.md, QUESTIONS.md, MODULES.md (this file) |

**Blocking:** None. This is prerequisite for everything.

---

## Phase 1: Authentication & Core Database Schema

**Duration:** 3–4 days  
**Deliverable:** Users can log in, roles exist, base schema created with RLS  
**Owner:** Copilot

### 1.1 — Authentication (3 days)

**Sections:**
- Login page (`/login`)
- Signup page (`/signup`)
- Admin panel (`/admin/users`, manager-only role required)
- Session management (middleware, JWT cookies, refresh)
- Password recovery

**Database changes:**
```sql
CREATE TABLE organizations (id UUID, name TEXT, created_at TIMESTAMP);
CREATE TABLE users (
  id UUID PRIMARY KEY,
  org_id UUID REFERENCES organizations,
  auth_id UUID REFERENCES auth.users,
  email TEXT,
  name TEXT,
  role TEXT CHECK (role IN ('rep', 'manager', 'operaciones', 'director')),
  status TEXT CHECK (status IN ('active', 'inactive')),
  created_at TIMESTAMP,
  updated_at TIMESTAMP
);
CREATE TABLE roles (org_id UUID, role_name TEXT, permissions JSONB);
```

**Agent interactions:** None (Phase 1 has no agents yet).

**API endpoints:**
- `POST /api/auth/login` — email/password → JWT token
- `POST /api/auth/signup` — create user (requires org code)
- `POST /api/auth/refresh` — refresh JWT
- `GET /api/admin/users` — list users (manager-only)
- `POST /api/admin/users` — create user (manager-only)
- `PUT /api/admin/users/[id]` — update user role (manager-only)
- `DELETE /api/admin/users/[id]` — deactivate user (manager-only)

**RLS policies:**
- Users can only see users in their org
- Only managers can edit user roles
- Only directors can view deletion history

**Policy implementation:** See [RLS_SETUP.md Section 5](/docs/RLS_SETUP.md#step-2-create-foundation-policies-for-phase-1) for exact SQL. Foundation policies are applied in TASKS.md Phase 2.

**Acceptance criteria:**
- Rep can login with email/password
- Manager can create users and assign roles
- Session expires after 7 days
- RLS blocks cross-org access

**Blocking:** Q7 (role system definition) should be clarified before Role Admin UI is built; can stub role checks as TBD.

---

### 1.2 — Core CRM Schema (3 days, parallel with 1.1)

**Sections:**
- Organizations + Users (from 1.1)
- Clients table
- Contacts table
- Deals + Pipeline stages
- Tasks table (base structure, full implementation in Phase 4)

**Database changes:**
```sql
CREATE TYPE segment_enum AS ENUM ('charales', 'truchas', 'atunes', 'tiburones', 'ballenas');
CREATE TYPE deal_stage_enum AS ENUM (
  'prospecto_identificado', 'primer_contacto', 'descubrimiento',
  'cotizacion_en_preparacion', 'cotizacion_enviada', 'seguimiento_negociacion',
  'cerrado_ganado', 'cerrado_perdido'
);
CREATE TYPE deal_classification_enum AS ENUM ('propia', 'asignada', 'heredada');

CREATE TABLE clients (
  id UUID PRIMARY KEY,
  org_id UUID REFERENCES organizations,
  name TEXT NOT NULL,
  segment segment_enum NOT NULL,
  annual_revenue_mxn DECIMAL,
  contact_count INT DEFAULT 0,
  zoho_id TEXT,
  created_at TIMESTAMP, updated_at TIMESTAMP
);

CREATE TABLE contacts (
  id UUID PRIMARY KEY,
  org_id UUID REFERENCES organizations,
  client_id UUID REFERENCES clients,
  name TEXT,
  email TEXT,
  phone TEXT,
  role TEXT CHECK (role IN ('rh', 'seguridad', 'medico', 'compras', 'director')),
  zoho_id TEXT,
  created_at TIMESTAMP
);

CREATE TABLE deals (
  id UUID PRIMARY KEY,
  org_id UUID REFERENCES organizations,
  client_id UUID REFERENCES clients,
  assigned_to UUID REFERENCES users,
  title TEXT,
  value_mxn DECIMAL,
  stage deal_stage_enum DEFAULT 'prospecto_identificado',
  classification deal_classification_enum DEFAULT 'propia',
  zoho_id TEXT,
  created_at TIMESTAMP, updated_at TIMESTAMP
);

CREATE TABLE tasks (
  id UUID PRIMARY KEY,
  org_id UUID REFERENCES organizations,
  title TEXT, description TEXT,
  status TEXT CHECK (status IN ('pending', 'in_progress', 'done', 'dismissed')),
  priority TEXT CHECK (priority IN ('low', 'medium', 'high', 'urgent')),
  assigned_to UUID REFERENCES users,
  client_id UUID REFERENCES clients,
  deal_id UUID REFERENCES deals,
  due_date TIMESTAMP,
  created_by TEXT,
  agent_name TEXT,
  zoho_task_id TEXT,
  zoho_synced_at TIMESTAMP,
  created_at TIMESTAMP, updated_at TIMESTAMP
);
```

**Agent interactions:** None yet (Supervisor agent in Phase 4 will create tasks).

**API endpoints (all gated by org + role RLS):**
- `GET /api/clients` — list clients
- `POST /api/clients` — create client
- `GET /api/clients/[id]` — client detail
- `PUT /api/clients/[id]` — update client
- `GET /api/contacts?client_id=[id]` — contacts for client
- `POST /api/contacts` — create contact
- `GET /api/deals` — list deals
- `POST /api/deals` — create deal
- `GET /api/deals/[id]` — deal detail
- `PUT /api/deals/[id]` — update deal
- `GET /api/tasks` — list tasks
- `POST /api/tasks` — create task
- `PUT /api/tasks/[id]` — update task

**RLS policies:**
- Users can only see clients/deals/tasks in their org
- Reps can only see their own deals unless Miriam shares
- Managers can see all org deals

**Acceptance criteria:**
- Reps can view clients by segment
- Managers can create new clients
- Deals show correct pipeline stage
- Tasks can be created and assigned
- RLS blocks cross-org/unauthorized access

---

## Phase 2: Conversations & Inbox (Main Interface)

**Duration:** 4–5 days  
**Deliverable:** Reps can chat with agents, threads visible, agent taxonomy in UI  
**Owner:** Copilot

### 2.1 — Conversation Infrastructure (3 days)

**Sections:**
- Inbox layout (nav rail + sidebar + main chat area)
- Conversation/thread UI
- Message bubble rendering (user/agent/system types)
- Realtime message updates
- Agent list with icons

**Database changes:**
```sql
CREATE TYPE message_sender_type AS ENUM ('user', 'agent', 'system');

CREATE TABLE conversations (
  id UUID PRIMARY KEY,
  org_id UUID REFERENCES organizations,
  title TEXT,
  type TEXT CHECK (type IN ('main', 'client_channel', 'agent_specialist')),
  agent_name TEXT,
  client_id UUID REFERENCES clients,
  deal_id UUID REFERENCES deals,
  created_at TIMESTAMP
);

CREATE TABLE messages (
  id UUID PRIMARY KEY,
  org_id UUID REFERENCES organizations,
  conversation_id UUID REFERENCES conversations,
  sender_type message_sender_type,
  sender_id UUID REFERENCES users,
  agent_name TEXT,
  content TEXT,
  metadata JSONB,
  created_at TIMESTAMP
);

CREATE TABLE conversations_users (
  conversation_id UUID REFERENCES conversations,
  user_id UUID REFERENCES users,
  is_archived BOOLEAN DEFAULT FALSE,
  last_read TIMESTAMP
);
```

**Agent interactions:**
- GDT Supervisor agent creates/reuses main conversation threads
- Each specialist agent (Follow Up, Sales Assistant, Reporting) has own conversation thread
- Agents post summaries, task creation cards, and recommendations
- Intent Router classifies incoming user messages and selects appropriate agent

**API endpoints:**
- `GET /api/conversations` — list user's conversations
- `POST /api/conversations` — create conversation
- `GET /api/conversations/[id]/messages` — get messages in conversation
- `POST /api/conversations/[id]/messages` — send message (user)
- `POST /api/conversations/[id]/agent-message` — post agent message (internal)
- `PUT /api/conversations/[id]` — mute/archive conversation
- `GET /api/conversations/[id]/realtime` — WebSocket for realtime updates

**Components:**
- `NavRail` — agent icons, dashboard, tasks, settings
- `Sidebar` — main GDT conv, specialist convs with unread badges, client channels, search
- `MessageBubble` — renders user/agent/system messages with formatting
- `MessageComposer` — text input, @mentions, /slash commands, file upload
- `ConversationFrame` — conversation detail view

**Acceptance criteria:**
- Rep opens Inbox, sees GDT main chat
- Rep sends message, Supervisor agent responds within 2 sec
- Rep can see specialist conversations (Follow Up, Sales Assistant, Reporting)
- Client channels auto-appear under specialist when context detected
- Realtime: new messages appear without refresh

**Chat flow example (Follow Up Agent):**
```
Rep: "@client Acme Corp, what happened last week?"
[GDT Supervisor analyzes message, detects Sales Assistant intent]
[GDT routes to Sales Assistant agent conversation]
→ Sales Assistant: "Acme Corp • Revenue $200K • Last activity: March 10 (3 days ago)
   Last communication: Quote sent for $50K project. Probability: 70%.
   Recommended: Call today to confirm timeline. [Create Follow-up Task]"
[If rep clicks "Create Follow-up Task," task posted to follow-up agent thread]
```

---

### 2.2 — Slash Commands (2 days, parallel with 2.1)

**Sections:**
- Command parser
- Command handler functions
- Autocomplete UI

**Available commands (MVP):**
- `/resumen @cliente` → Sales Assistant generates client context card
- `/pipeline` → Supervisor generates pipeline summary
- `/reporte` → Reporting agent generates activity report
- `/tarea titulo…` → Create task manually with due date

**Database changes:** None (uses existing messages, conversations tables).

**Agent interactions:**
- Each command routes to appropriate agent (Sales Assistant, Supervisor, Reporting)
- Agent executes and posts result back to conversation

**API endpoints:**
- `POST /api/commands/resumen` — sales assistant context
- `POST /api/commands/pipeline` — supervisor summary
- `POST /api/commands/reporte` — reporting agent report
- `POST /api/commands/tarea` — create task

**Acceptance criteria:**
- `/resumen` shows client name, revenue, last activity, number of deals
- `/pipeline` shows funnel by stage with values and rep names
- `/reporte` shows weekly activity (calls, tasks, quotes)
- `/tarea Llamar a cliente` creates task due today

---

## Phase 3: CRM Core + Quote Management

**Duration:** 5–6 days  
**Deliverable:** Reps can view/manage clients/deals/quotes; quotation builder works  
**Owner:** Copilot

### 3.1 — Client & Deal Views (2 days)

**Sections:**
- Client list page (`/clients`)
- Client detail page (`/clients/[id]`)
- Deal list page (`/deals`, Kanban + table views)
- Deal detail page (`/deals/[id]`)
- Quick actions (call, create task, create deal)

**Database changes:** None (uses Phase 1 + Phase 2 schema).

**Agent interactions:** None (read-only UI).

**API endpoints:**
- `GET /api/clients?segment=truchas&limit=10` — list with filters
- `GET /api/clients/[id]` — detail (includes contacts, deals, last activity)
- `GET /api/deals?stage=cotizacion_enviada&sort=value_desc` — list with filters
- `GET /api/deals/[id]` — detail (includes tasks, calls, quotes, timeline)

**Routes & Components:**
- `/clients` — DataTable with columns: name, segment, revenue, contact count, next activity
  - Filter by segment
  - Search by name
  - Quick create deal button
- `/clients/[id]` — tabs: Overview | Contacts | Deals | Timeline | Calls
  - Overview: client info, annual revenue, segment badge
  - Contacts: table of contacts with roles
  - Deals: cards showing all deals for client, sorted by stage
  - Timeline: chronological log of all activities
  - Calls: call history with post-call summaries
- `/deals` — Kanban board with 8 stage columns, or table view
  - Each card shows: client name, deal value, 3-day stale indicator
  - Click to open detail
  - Drag to change stage (if authorized)
- `/deals/[id]` — tabs: Overview | Conversations | Tasks | Timeline | Quotes | Expenses
  - Overview: client name, segment, deal value, stage, assigned rep, next task
  - Conversations: all messages related to deal across agents
  - Tasks: tasks linked to deal
  - Timeline: agent actions + user actions + call logs
  - Quotes: preplan → plan → quote progression
  - Expenses: (Phase 5) preplan/plan/real comparison

**Acceptance criteria:**
- Reps see their own deals filtered correctly
- Managers see all org deals
- 3-day stale deals highlighted
- Can click to drill into detail
- Can create new deal from client view

---

### 3.2 — Quote Management / Expense Planning (3 days, parallel with 3.1)

**Sections:**
- Quotation form (preplan stage)
- Quotation builder (plan → final quote)
- Approval queue for low-margin quotes
- PDF generation
- Email send (requires Miriam approval)

**Database changes:**
```sql
CREATE TABLE expense_plans (
  id UUID PRIMARY KEY,
  org_id UUID REFERENCES organizations,
  deal_id UUID REFERENCES deals,
  type TEXT CHECK (type IN ('preplan', 'plan', 'real')),
  parent_id UUID REFERENCES expense_plans,
  title TEXT,
  base_cost_mxn DECIMAL,
  margin_percent DECIMAL,
  total_revenue_mxn DECIMAL,
  approval_status TEXT CHECK (approval_status IN ('pending', 'approved', 'rejected')),
  approved_by UUID REFERENCES users,
  approved_at TIMESTAMP,
  created_by UUID REFERENCES users,
  created_at TIMESTAMP
);

CREATE TABLE expense_plan_line_items (
  id UUID PRIMARY KEY,
  expense_plan_id UUID REFERENCES expense_plans,
  category TEXT,
  description TEXT,
  quantity INT,
  unit_cost_mxn DECIMAL,
  line_total_mxn DECIMAL
);

CREATE TABLE quotes (
  id UUID PRIMARY KEY,
  org_id UUID REFERENCES organizations,
  deal_id UUID REFERENCES deals,
  expense_plan_id UUID REFERENCES expense_plans,
  quote_number TEXT,
  pdf_url TEXT,
  sent_at TIMESTAMP,
  sent_to TEXT,
  viewed_at TIMESTAMP,
  valid_until TIMESTAMP,
  status TEXT CHECK (status IN ('draft', 'sent', 'viewed', 'accepted', 'rejected')),
  created_at TIMESTAMP
);
```

**Agent interactions:**
- **Sales Assistant agent:** When rep enters deal, Sales Assistant generates preplan template with historical margin expectations
- **Miriam (manager) approval gate:** If margin < 30%, quote marked "pending_approval" until Miriam approves
- **Reporting agent:** Can generate quote summary and send via email (requires Miriam approval for external send)

**Chat flow (Sales Assistant + Quote):**
```
Rep in deal: "Help me with a quote for Acme"
→ Sales Assistant posts template: "Based on similar deals with Acme:
   Labor: $15K (3 engineers × 5 days)
   Materials: $8K
   Margin target: 40% (industry standard for Atún segment)
   If margin < 30%, requires Miriam approval.
   [Use Template] or [Enter Custom]"
Rep fills form, submits
If margin >= 30%: Auto-approved, can send to client
If margin < 30%: Card posted to Miriam approval queue
```

**API endpoints:**
- `POST /api/quotes/preplan` — create preplan (sales assistant suggests template)
- `PUT /api/quotes/preplan/[id]` — edit line items, recalc margin
- `POST /api/quotes/plan` — generate plan from preplan
- `POST /api/quotes/[id]/generate-pdf` — PDF for quote
- `POST /api/quotes/[id]/send` — send email (requires approval if margin < 30%)
- `GET /api/quotes/[id]/approval-status` — check if pending approval
- `POST /api/approvals/[id]/approve` — Miriam approves low-margin quote

**Routes & Components:**
- `/deals/[id]/quotes` — Quote builder
  - Form: preplan > plan > final quote
  - Margin calculator (real-time)
  - "Margin too low" alert with Miriam approval button
  - PDF preview
  - Send email button (disabled if approval pending)
- `/approvals` — Approval queue for low-margin quotes
  - Shows quote, deal client, calculated margin, rep name
  - Approve / Reject buttons
  - Comments field
- Modal: "Quote sent to client" → Email tracking

**Acceptance criteria:**
- Rep can create preplan with template suggestion
- Margin < 30% halts send until Miriam approves
- PDF generated with quote number, valid-until date, client name
- Email send tracked with open/click status (Phase 5)

**Blocking:** Q2 (preplan approval authority) should be clarified. Current assumption: Miriam for all low-margin quotes; Dirección approval gate added in Phase 5.

---

## Phase 4: Agents & Automation

**Duration:** 6–8 days  
**Deliverable:** Supervisor agent runs daily, agents create tasks and recommendations  
**Owner:** Copilot + Railroad setup

### 4.1 — Core Agent Infrastructure (3 days)

**Sections:**
- LangGraph orchestration setup
- Supervisor agent
- Intent router
- Agent action logging
- Approval queue

**Database changes:**
```sql
CREATE TABLE agent_actions (
  id UUID PRIMARY KEY,
  org_id UUID REFERENCES organizations,
  agent_name TEXT NOT NULL,
  action_type TEXT,
  input JSONB,
  output JSONB,
  rationale TEXT,
  confidence DECIMAL,
  model_name TEXT,
  model_provider TEXT,
  token_usage INT,
  estimated_cost DECIMAL,
  status TEXT CHECK (status IN ('auto_applied', 'pending_approval', 'rejected', 'executed')),
  approved_by UUID REFERENCES users,
  created_at TIMESTAMP
);
```

**Agent flows:**

#### **Supervisor Agent (Daily 7:00 AM, every 4 hours)**

**Trigger:** Scheduled cron (30-min scan via Railway agent).

**Signals detected:**
- Stale deals (no activity in 5+ days, adjusted by segment SLA)
- High-value deals at risk (value > $50K, stage not progressing)
- Unassigned leads
- Overdue tasks
- Clients with no activity in 90+ days (dormant reactivation)

**Actions:**
1. Posts daily pipeline summary to GDT main conversation
2. Creates follow-up tasks for stale deals (due: today)
3. Flags high-value deals with risk score
4. Creates "Reactivation" deals for dormant clients
5. Alerts reps on 0-activity days

**Chat example:**
```
[GDT Supervisor, 7:00 AM]
"Good morning Miriam! 📊 Pipeline summary:
 • 12 active deals, $450K total
 • 3 deals stale 7+ days (need follow-up):
   - Acme Corp (Atún, $50K, awaiting response since Mar 8)
   - Beta Inc (Charales, $12K, no activity)
   - Gamma Ltd (Tiburón, $180K, proposal sent Mar 5)
 • 1 high-value deal at risk: Delta Corp (Ballena, $300K, stuck in discovery)
 
 ⚠️ Recommended: Call Gamma Ltd today. [Create Follow-up Task]"
```

**API endpoints for Supervisor:**
- `GET /api/deals/stale?org_id=X&days=5` — get stale deals
- `POST /api/tasks` — create follow-up task
- `POST /api/messages/[conversation_id]` — post summary to GDT chat
- `GET /api/clients/dormant?days=90` — get dormant clients
- `POST /api/deals` — create reactivation deal

**Model:** Mistral-7B (via OpenRouter, $0.0002/1k tokens) → fallback GPT-4o-mini (routing/classification at 80% cost savings).

#### **Follow Up Agent (Low-frequency; triggered by Supervisor)**

**Trigger:** When Supervisor posts follow-up task.

**Signal:** Stale deal detected by Supervisor.

**Action:**
1. Generates follow-up message template ("Hi [contact], checking in on [project]…")
2. Suggests next step (call, email, meeting)
3. Drafts email or call script
4. Posts to Follow Up conversation channel

**Chat example:**
```
[Follow Up Agent, GDT main]
"Acme Corp • Stale 7 days
Suggested: Call Javier (Compras) today
Script: 'Hi Javier, I wanted to follow up on the $50K project we discussed on March 8. 
         Have you had a chance to review the proposal? Any questions?'
[Create Task] [Draft Email] [View Deal]"
```

**API endpoints for Follow Up:**
- `GET /api/deals/[id]` — get deal context
- `GET /api/contacts?client_id=X` — get contact names
- `POST /api/messages` — post template

**Model:** GPT-4o-mini (via OpenRouter, $0.0015/1k tokens) → fallback Mistral-Medium (template generation, cost-optimized).

#### **Sales Assistant Agent (Real-time; triggered by user message)**

**Trigger:** User message in inbox (intent = context request, upsell, deal question).

**Signals:**
- "@client [name]" -> user asking for context
- "What happened with…" -> deal status request
- "Is there an upsell?" -> opportunity detection

**Actions:**
1. Generates client context card (revenue, history, contacts, last activity)
2. Suggests next steps (call script, quote template)
3. Detects upsell/renewal opportunities
4. Auto-prepares quote if opportunity matches

**Chat example:**
```
Rep: "@client Acme Corp - what's our status?"
→ Sales Assistant Card:
"Acme Corp • Atún segment • $200K annual
Last deal: $50K (Quote sent Mar 5, no response 3 days)
Contacts: Javier (Compras), Maria (Director)
Recommended: Call Javier today about delivery timeline.
Upsell opportunity: Year-round support contract ($15K/yr, 60% margin). [Draft Quote]"
```

**API endpoints for Sales Assistant:**
- `GET /api/clients/[id]` — client detail
- `GET /api/deals?client_id=X` — recent deals
- `GET /api/contacts?client_id=X` — contact list
- `POST /api/quotes/[id]/suggestions` — upsell templates

**Model:** GPT-4o (via OpenRouter, $0.003/1k tokens) → fallback Claude-3.5-Sonnet (context synthesis, balanced quality/cost).

#### **Intent Router**

**Purpose:** Every message from rep → classify intent → select appropriate agent.

**Intents:**
- `context_request` ("¿Qué ha pasado con Acme?") → Sales Assistant
- `stale_deal_follow_up` ("Creo que debería llamar a Acme") → Follow Up Agent
- `pipeline_status` ("/pipeline") → Supervisor
- `task_creation` ("Crear tarea…") → Supervisor
- `approval_request` ("Aprueba esta quote?") → Human (Miriam) UI
- `call_transcript` (post-call summary trigger) → Technical Agent (Phase 5)

**API endpoints:**
- `POST /api/intent-router/classify` — classify message & route (via OpenRouter Mistral-7B, fallback GPT-4o-mini)

**Model:** Mistral-7B (via OpenRouter, ultra-cheap) → fallback GPT-4o-mini (intent routing at lowest cost).

**Acceptance criteria:**
- Supervisor agent runs daily at 7 AM
- Stale deal tasks created automatically
- Follow Up agent drafts messages on request
- Sales Assistant generates context in < 3 sec
- Intent router classifies 95%+ correctly

---

### 4.2 — Task System & Approvals (2 days, parallel with 4.1)

**Sections:**
- Task creation (agent + manual)
- Task lifecycle (pending → in progress → done)
- Approval queue UI
- Task completion & reward

**Database changes:** Extends Phase 1 tasks table.
```sql
ALTER TABLE tasks ADD COLUMN conversation_id UUID REFERENCES conversations;
ALTER TABLE tasks ADD COLUMN action_id UUID REFERENCES agent_actions;

CREATE TABLE approvals (
  id UUID PRIMARY KEY,
  org_id UUID REFERENCES organizations,
  action_id UUID REFERENCES agent_actions,
  action_type TEXT,
  content JSONB,
  requested_by UUID REFERENCES users,
  approver_id UUID REFERENCES users,
  status TEXT CHECK (status IN ('pending', 'approved', 'rejected')),
  approved_at TIMESTAMP
);
```

**Agent interactions:**
- Supervisor creates tasks for stale deals
- Miriam sees Approvals queue with pending action cards
- Miriam approves/rejects with comments
- Approved actions execute, rejected actions logged

**Chat flow (Approval):**
```
[Supervisor posts to GDT]
"🔔 Acme Corp quote ready (margin 28%, below 30% threshold)
 Requested approval: Send quote to client"

[Miriam sees approval queue card]
Card shows: Deal, quote margin, rep name, "Approve" / "Reject" buttons

Miriam clicks "Approve", reply: "Approved - send it. Good deal with Acme."
→ Quote sent to client
→ Task marked "completed"
→ Timeline log: "Miriam approved quote send at 2:45 PM"
```

**API endpoints:**
- `GET /api/approvals?status=pending` — list pending approvals
- `POST /api/approvals/[id]/approve` — approve with comment
- `POST /api/approvals/[id]/reject` — reject with reason
- `PUT /api/tasks/[id]` — mark task done
- `GET /api/tasks?assigned_to=[user_id]` — user's task list

**Routes & Components:**
- `/approvals` — Approval queue page
  - Cards for pending actions: quote sends, stage changes, meeting scheduling
  - Action card shows: Deal client, amount, rep, reason, margin/risk
  - Approve / Reject buttons with comment field
- `/tasks` — Task list
  - Filters: by owner, status, priority, due date
  - Quick mark done / dismiss buttons
  - Grouped: Today | Overdue | Upcoming | Completed

**Acceptance criteria:**
- Supervisor creates 1–3 tasks per day
- Miriam approves quote sends in < 5 min
- Task completed logged in timeline
- RLS blocks cross-org task access

---

### 4.3 — Call Logging & Post-Call Summary (2 days, parallel with 4.1)

**Sections:**
- Mic-listen mode (Web Speech API)
- Real-time transcript display
- Post-call AI summary
- Task auto-creation from call

**Database changes:**
```sql
CREATE TABLE calls (
  id UUID PRIMARY KEY,
  org_id UUID REFERENCES organizations,
  deal_id UUID REFERENCES deals,
  contact_id UUID REFERENCES contacts,
  called_by UUID REFERENCES users,
  transcript TEXT,
  summary TEXT,
  tasks_created INT,
  created_at TIMESTAMP
);
```

**Agent interactions:**
- **Technical Agent:** Listens to call transcript (not chat-based), generates summary
- **Supervisor:** Reads summary, creates follow-up task if needed

**Flow:**
```
Rep clicks "Call" in deal view
→ Mic input starts, transcript displays real-time
→ Rep talks to client
→ Rep clicks "End Call"
→ Technical Agent analyzes transcript (async)
→ Summary posted to deal conversation: "Client agreed to implement March 20. Issue: budget approval needed from director. Action: Send PO template. [Create Task]"
→ Task created automatically with due date = next day
```

**API endpoints:**
- `POST /api/calls/start` — begin call, return session ID
- `POST /api/calls/[id]/transcript` — stream transcript chunks
- `POST /api/calls/[id]/end` — end call, trigger summary worker
- `GET /api/calls/[id]/summary` — poll for summary (when ready)
- `POST /api/calls/[id]/create-task` — auto-create task from summary

**Components:**
- `CallButton` — starts mic, shows active call indicator
- `LiveTranscript` — real-time transcript display during call
- `PostCallSummary` — shows summary, task suggestions, "Create Task" button

**Acceptance criteria:**
- Mic input captures speech
- Transcript displays real-time (< 2 sec latency)
- Summary generated within 30 sec of call end
- Tasks created with client name + action in title

---

## Phase 5: Reporting + ERP Modules

**Duration:** 7–10 days (longest, many parallel modules)  
**Deliverable:** 4 dashboards, Zoho sync, invoicing, collections flows operational  
**Owner:** Copilot + integration team

### 5.1 — Reporting & Dashboards (3 days)

**Sections:**
- Rep dashboard (`/dashboard/rep`)
- Manager dashboard (`/dashboard/manager`)
- Director dashboard (`/dashboard/director`, requires Q3 answer)
- Operations dashboard (`/dashboard/operaciones`)
- Reporting agent automation
- AI chat on dashboards (Reporting Agent answering questions)
- PDF export

**Database changes:**
```sql
CREATE TABLE dashboard_recommendations (
  id UUID PRIMARY KEY,
  org_id UUID REFERENCES organizations,
  user_id UUID REFERENCES users,
  recommendation_text TEXT,
  action_type TEXT,
  priority TEXT,
  created_at TIMESTAMP
);
```

#### **Rep Dashboard (`/dashboard/rep`)**

**Widgets:**
- My Deals (pipeline: stage breakdown, total value)
- My Tasks (due today, overdue, upcoming)
- My Calls (this week, next call due)
- My Quotes (sent, pending response, accepted)
- Activity (calls logged, follow-ups completed, deals won)
- AI Chat ("Ask about my pipeline")

**Chat example:**
```
Rep: "Why is Acme Corp stuck?"
→ Reporting Agent: "Acme Corp (Atún, $50K quote) has been in 'Cotización Enviada' for 3 days.
                    Last contact: Mar 5 quote, no follow-up call yet.
                    Recommendation: Call Javier today. [Create Task]"
```

**API endpoints:**
- `GET /api/dashboard/rep/deals` — my deals by stage
- `GET /api/dashboard/rep/tasks` — my tasks
- `GET /api/dashboard/rep/calls` — my calls
- `GET /api/dashboard/rep/ai-chat` — chat with Reporting Agent
- `POST /api/dashboard/rep/export-pdf` — PDF of dashboard

**Components (Recharts + ShadCN):**
- PieChart: Deal breakdown by stage
- BarChart: Tasks by status/priority
- ProgressBar: Pipeline progress this quarter
- ListCard: Next tasks, next calls, recent activity

---

#### **Manager Dashboard (`/dashboard/manager`)**

**Widgets:**
- Team Pipeline (all reps, summary by stage, total value)
- Rep Activity (calls, follow-ups, quotes sent, win rate)
- Approvals Pending (summary, quick approval buttons)
- At-Risk Deals (stale, high-value stuck deals)
- KPIs (calls/rep/day, quote conversion, cycle time)
- AI Chat ("How many deals with Acme?")

**Chat example:**
```
Miriam: "What's my approval load?"
→ Reporting Agent: "You have 3 pending approvals:
                    • Acme Corp quote ($50K, 28% margin) - awaiting approval
                    • Beta Inc stage change to Won ($100K) - awaiting approval
                    • Gamma Ltd email send - awaiting approval
                    Estimated response time: margin < 30% requires 05 min review."
```

**API endpoints:**
- `GET /api/dashboard/manager/team-deals` — all reps' deals
- `GET /api/dashboard/manager/rep-activity` — KPIs per rep
- `GET /api/dashboard/manager/approvals` — pending approvals
- `GET /api/dashboard/manager/at-risk-deals` — stale/stuck deals
- `GET /api/dashboard/manager/ai-chat` — chat with Reporting Agent
- `POST /api/dashboard/manager/export-pdf` — PDF

**Components:**
- Stacked BarChart: Pipeline by rep
- LineChart: Calls/day trend
- Table: Rep leaderboard (calls, quotes, conversions)
- Alert cards: Pending approvals

---

#### **Director Dashboard (`/dashboard/director`)**

**Widgets:**
- Pipeline by Horizon (Prospección | Forecast | Business Case)
- Revenue Forecast (3-month outlook, confidence bands)
- Margin Analysis (by segment, by rep, by deal classification)
- Collections Status (invoiced, paid, overdue, aging)
- Factoring Portfolio (if applicable, Phase 2)
- AI Chat ("What's forecast for Q3?")

**Chart example:**
```
Director: "What's our Q3 forecast?"
→ Reporting Agent: "Q3 2026 Forecast (≥90% confidence): $1.2M
                    • Forecast deals: $950K confirmed, $250K pending
                    • Business Case: $500K pipeline (future upside)
                    • Collections: $200K invoiced, $180K paid (90% rate)
                    • Margin avg: 32% (target 35%). Risk: 2 deals with 25% margin."
```

**API endpoints:**
- `GET /api/dashboard/director/pipeline-by-horizon` — three horizons
- `GET /api/dashboard/director/forecast` — revenue forecast
- `GET /api/dashboard/director/margin-analysis` — margins by segment/rep
- `GET /api/dashboard/director/collections` — invoiced/paid/overdue
- `GET /api/dashboard/director/ai-chat` — chat
- `POST /api/dashboard/director/export-pdf` — PDF

**Widgets (Recharts):**
- StackedAreaChart: Forecast by horizon (stacked)
- BarChart: Margin % by segment
- PieChart: Collections status (paid, overdue, aging)
- LineChart: Revenue trend (actual vs forecast vs target)

**Blocking:** Q3 (arrastre calculation formula) must be clarified to calculate accurate forecasts.

---

#### **Operations Dashboard (`/dashboard/operaciones`)**

**Widgets:**
- Service Calendar (mobile unit availability, personnel)
- Pending Approvals (medical results, scheduling)
- Expense Tracking (preplan/plan/real de gastos)
- PO Status (pending, approved, delivered, overdue)
- AI Chat ("Which services are pending?")

**API endpoints:**
- `GET /api/dashboard/operaciones/calendar` — upcoming services
- `GET /api/dashboard/operaciones/medical-results` — pending approvals
- `GET /api/dashboard/operaciones/expense-plans` — expense tracking
- `GET /api/dashboard/operaciones/purchase-orders` — PO status
- `GET /api/dashboard/operaciones/ai-chat` — chat

(**Details in Phase 5.3 below.**)

---

#### **Reporting Agent (Automated Reports)**

**Triggers:**
- Weekly (Friday 5 PM): Activity report
- Monthly (1st, 8 AM): Pipeline & revenue report
- On-demand (via `/reporte` slash command)

**CSV/PDF exports:**
- Weekly: Calls/rep, quotes sent/converted, follow-ups, deals won/lost, KPIs
- Monthly: Pipeline funnel, conversion rates, cycle time, margin analysis, rep rankings

**API endpoints:**
- `GET /api/reporting/weekly` — weekly report
- `GET /api/reporting/monthly` — monthly report
- `POST /api/reporting/export` — export to CSV/PDF

---

### 5.2 — Integrations (3 days, parallel with 5.1)

#### **Zoho CRM Sync**

**Phases:**
1. Phase 1 MVP (Mar–May): Read-only sync from Zoho
2. Phase 2 (Jun–Aug): Dual-write (Supabase → Zoho for tasks/deals)
3. Phase 3 (Aug–Sep): Read-only switch (Zoho archive only)
4. Phase 4 (Oct): Decommission

**Database changes:**
```sql
ALTER TABLE clients ADD COLUMN zoho_id TEXT;
ALTER TABLE contacts ADD COLUMN zoho_id TEXT;
ALTER TABLE deals ADD COLUMN zoho_id TEXT;
ALTER TABLE tasks ADD COLUMN zoho_task_id TEXT;
ALTER TABLE tasks ADD COLUMN zoho_synced_at TIMESTAMP;

CREATE TABLE zoho_sync_log (
  id UUID PRIMARY KEY,
  entity TEXT,
  zoho_id TEXT,
  action TEXT CHECK (action IN ('sync_in', 'sync_out', 'conflict', 'error')),
  status TEXT,
  error_message TEXT,
  created_at TIMESTAMP
);
```

**Sync flow (Pipeline):**
```
[Railway cron every 15 min]
→ Fetch Zoho Accounts, Contacts, Deals, Quotes, Activities
→ GET /api/zoho/sync/accounts (incremental: modified_time > last_sync)
→ Match to Supabase clients by zoho_id
→ Upsert (fields: name, revenue, etc.)
→ Log sync to zoho_sync_log
→ If conflict (Supabase updated, Zoho updated): Log conflict, alert Miriam
```

**API endpoints:**
- `POST /api/zoho/oauth/callback` — OAuth callback after login
- `GET /api/zoho/sync/accounts` — fetch and sync accounts
- `GET /api/zoho/sync/deals` — fetch and sync deals
- `GET /api/zoho/sync/activities` — fetch and sync activities
- `POST /api/zoho/write/task` — write task back to Zoho
- `POST /api/zoho/write/deal` — write deal back to Zoho
- `GET /api/zoho/sync-status` — sync status & last timestamp

**Blocking:** None (read-sync can start Phase 5). Write-sync needs Q4 (Facturella) and Q5 (Medicati) answers to avoid conflicts.

---

#### **Facturella Integration (CFDI Invoicing)**

**Database changes:**
```sql
CREATE TABLE invoices (
  id UUID PRIMARY KEY,
  org_id UUID REFERENCES organizations,
  deal_id UUID REFERENCES deals,
  quote_id UUID REFERENCES quotes,
  invoice_number TEXT UNIQUE,
  amount_mxn DECIMAL,
  tax_mxn DECIMAL,
  total_mxn DECIMAL,
  customer_rfc TEXT,
  cfdi_xml TEXT,
  cfdi_url TEXT,
  facturella_id TEXT,
  status TEXT CHECK (status IN ('draft', 'sent', 'signed', 'cancelled')),
  created_at TIMESTAMP
);
```

**Flow:**
```
1. Rep/Manager clicks "Generate Invoice" on deal
2. System creates invoice from quote metadata
3. Calls Facturella API to sign CFDI
4. Stores PDF + XML
5. Sends to client email (requires approval)
6. Status updates when client views/pays
```

**Agent interactions:** None (Miriam approves send).

**API endpoints:**
- `POST /api/invoices/generate` — create draft invoice
- `POST /api/invoices/[id]/sign` — call Facturella to sign CFDI
- `POST /api/invoices/[id]/send` — email invoice (requires approval)
- `GET /api/invoices/[id]/status` — fetch current status from Facturella
- `POST /api/invoices/[id]/cancel` — cancel invoice

**Blocking:** Q4 (Facturella API validation) — Need API credentials and sandbox test.

---

#### **Medicati Integration (Medical Results)**

**Database changes:**
```sql
CREATE TABLE medical_results (
  id UUID PRIMARY KEY,
  org_id UUID REFERENCES organizations,
  deal_id UUID REFERENCES deals,
  service_event_id UUID REFERENCES service_events,
  medicati_request_id TEXT,
  patient_name TEXT,
  result_type TEXT,
  result_files JSONB,
  status TEXT CHECK (status IN ('pending', 'received', 'approved', 'delivered')),
  created_at TIMESTAMP
);
```

**Flow:**
```
1. Operations requests medical work (healthcare deal)
2. Medicati performs tests, delivers results
3. Webhook: Medicati → /api/webhooks/medicati
4. System stores results, creates approval task for Operations
5. Operations approves, client receives results
```

**Agent interactions:** Supervisor monitors pending approvals, alerts Operations.

**API endpoints:**
- `POST /api/webhooks/medicati` — receive results from Medicati
- `GET /api/medical-results/[id]` — get result detail
- `PUT /api/medical-results/[id]/approve` — approve delivery
- `POST /api/medical-results/[id]/send` — send to client

**Blocking:** Q5 (Medicati webhook format & signature validation) — Need webhook spec documentation.

---

### 5.3 — ERP Modules (4 days, parallel with 5.1 + 5.2)

#### **5.3a — Expense Planning (Preplan → Plan → Real)**

**Sections:**
- Expense planning form
- Approval workflow (Miriam for margin < 30%)
- Real gastos tracking
- Variance analysis

**Database:** (Already in Phase 3.2)
- `expense_plans` table (preplan | plan | real)
- `expense_plan_line_items`

**Flow:**
```
1. Rep creates preplan (Sales Assistant suggests template)
2. Margin calculated: (Total Revenue - Base Cost) / Total Revenue × 100
3. If margin < 30%: Approval card sent to Miriam
4. Miriam approves/rejects
5. Plan created (frozen copy of preplan)
6. Real gastos tracked as project progresses
7. Variance report: Plan vs Real (to Director)
```

**Agent interactions:**
- **Sales Assistant:** Suggests preplan templates (labor hours, materials, travel)
- **Reporting Agent:** Generates variance reports (Plan vs Real)

**API endpoints:**
- `POST /api/expense-plans/preplan` — create preplan
- `PUT /api/expense-plans/[id]/line-items` — edit line items
- `POST /api/expense-plans/[id]/plan` — freeze to plan
- `POST /api/expense-plans/[id]/real` — add real gastos
- `GET /api/expense-plans/[id]/variance` — Plan vs Real comparison
- `GET /api/expense-plans?status=pending_approval` — approval queue

**Routes:**
- `/deals/[id]/expenses` — Expense planning tab
  - Form: Preplan section with line items
  - Margin calc (real-time)
  - "Awaiting Miriam approval" badge if margin < 30%
  - Plan + Real sections (once approved)
  - Variance comparison (final)

**Acceptance criteria:**
- Rep can create preplan with 3–5 line items
- Margin < 30% auto-routes to Miriam
- Miriam approves in < 5 min
- Plan locked after approval
- Real gastos tracked weekly

---

#### **5.3b — Invoicing (Facturella CFDI)**

**Sections:** (See 5.2 above)
- Generate invoice from quote
- Sign CFDI via Facturella API
- Send to client (requires approval)
- Track payment status

**Database:** `invoices` table (see 5.2)

**Flow:**
```
1. Quote → Plan finalized → Miriam marks "Ready to Invoice"
2. System generates invoice (quote data + tax calculations)
3. Facturella signs CFDI (async, ~5 sec)
4. Invoice card posted to Miriam approval queue
5. Miriam approves send
6. Email to client with PDF + download link
7. Webhook: Client views, pays, invoice status updates
```

**Agent interactions:**
- **Reporting Agent:** Weekly invoicing summary (invoiced, paid, overdue)

**API endpoints:** (See 5.2)

**Routes:**
- `/dashboard/manager/invoices` — Invoice management
  - Table: invoice number, client, amount, date, status, paid date
  - Filter: by status (draft, sent, paid, overdue)
  - Quick action: "Send Invoice" (if draft)
- `/deals/[id]/expenses` → Invoke tab → "Generate Invoice" button

**Acceptance criteria:**
- Invoice generated in < 2 sec
- CFDI signed via Facturella
- Email sent with download link
- Payment tracking updated via webhook

**Blocking:** Q4 (Facturella API credentials + sandbox setup).

---

#### **5.3c — Collections: Auxiliar de Pagos, Mora Escalation, Factoring**

**Sections:**
- Payment auxiliary (tracking what's been invoiced/paid)
- Mora (overdue) escalation workflow
- Factoring analysis (if applicable)
- Aging report

**Database changes:**
```sql
CREATE TABLE payments (
  id UUID PRIMARY KEY,
  org_id UUID REFERENCES organizations,
  invoice_id UUID REFERENCES invoices,
  deal_id UUID REFERENCES deals,
  payment_date TIMESTAMP,
  amount_mxn DECIMAL,
  status TEXT CHECK (status IN ('pending', 'partial', 'paid', 'overdue', 'mora')),
  days_overdue INT GENERATED ALWAYS AS (CURRENT_DATE - (due_date)::DATE) STORED,
  mora_escalation_level INT DEFAULT 0,
  factoring_flagged BOOLEAN DEFAULT FALSE,
  factoring_analysis JSONB,
  created_at TIMESTAMP
);

CREATE TABLE mora_escalation (
  id UUID PRIMARY KEY,
  org_id UUID REFERENCES organizations,
  payment_id UUID REFERENCES payments,
  level INT (1 = reminder, 2 = second notice, 3 = legal hold),
  sent_at TIMESTAMP,
  action_taken TEXT
);
```

**Flow (Mora Escalation):**
```
[Cron: Daily 8 AM, "payment-mora-check"]
→ Query invoices with due_date BEFORE today
→ Calculate days_overdue for each
→ Level 1 (1-5 days): Email reminder to customer
→ Level 2 (6-15 days): Email + SMS reminder
→ Level 3 (16+ days): Alert to Dirección (Omar), legal hold flag, factoring analysis
→ Posts summary to Collections conversation
```

**Flow (Factoring Analysis):**
```
If invoice 16+ days overdue && amount > $50K:
→ Director (Omar) sees factoring alert
→ Factoring summary: "Invoice $100K, 20 days overdue. Factoring cost ~3% = $3K.
                     Recommendation: Offer to client or hold."
→ Omar approves/rejects factoring
→ If approved: Submit to factoring partner, track proceeds
```

**Agent interactions:**
- **Supervisor:** Daily mora escalation alerts
- **Reporting Agent:** Weekly aging report (current, 30, 60, 90+ days)

**API endpoints:**
- `GET /api/payments/auxiliary` — auxiliar de pagos (all invoiced deals)
- `GET /api/payments?status=mora` — overdue payments
- `POST /api/mora/escalate` — trigger mora escalation (cron)
- `POST /api/mora/[id]/level-1` — send reminder
- `POST /api/mora/[id]/level-3` — legal hold + alert Director
- `POST /api/factoring/analyze` — factoring analysis
- `POST /api/factoring/[id]/approve` — approve factoring
- `GET /api/reporting/aging-report` — aging by days due

**Routes:**
- `/dashboard/director/collections` — Collections dashboard
  - Auxiliar table: Invoice, client, amount, due date, payment date, status, days overdue
  - Quick filter: Current | 30+ days | 60+ days | 90+ days
  - Mora escalation alerts (Level 3 in red)
  - Factoring recommendations
- `/approvals` → Approvals queue (factoring card for Omar)

**Cron jobs:**
- `payment-mora-check` (daily 8 AM via Railway)
- `mora-escalation-level-1` (if overdue 1-5 days)
- `mora-escalation-level-2` (if overdue 6-15 days)
- `mora-escalation-level-3` (if overdue 16+ days, alert Director)

**Acceptance criteria:**
- Mora alerts sent daily
- Level 3 alerts reach Director
- Factoring analysis generated for 16+ day invoices
- Aging report accurate

---

#### **5.3d — Purchase Orders & Remanente Tracking**

**Sections:**
- PO tracking (pending → approved → delivered)
- Remanente calculation (remaining commitment value)
- 3-month expiry rule

**Database:** `purchase_orders` table (see Phase 3.2)
```sql
CREATE TABLE purchase_orders (
  id UUID PRIMARY KEY,
  org_id UUID REFERENCES organizations,
  deal_id UUID REFERENCES deals,
  po_number TEXT,
  amount_mxn DECIMAL,
  received_date TIMESTAMP,
  expiry_date TIMESTAMP GENERATED ALWAYS AS (received_date + INTERVAL '3 months'),
  remanente_mxn DECIMAL GENERATED ALWAYS AS (amount_mxn - (SELECT SUM(invoice_total) FROM invoices WHERE po_id = purchase_orders.id)),
  expired BOOLEAN GENERATED ALWAYS AS (expiry_date < CURRENT_DATE),
  status TEXT CHECK (status IN ('pending', 'approved', 'partial', 'complete', 'expired')),
  created_at TIMESTAMP
);
```

**Flow:**
```
1. Client sends PO (email → Operations → stored in system)
2. Operations uploads PO image/PDF to deal
3. System extracts PO number, amount, validates
4. Remanente tracked: Total PO - Invoiced to date
5. Cron weekly: Alert if expiry_date within 14 days
6. If expired: Flag, alert Sales rep to renew PO
```

**Agent interactions:**
- **Supervisor:** Weekly PO remanente alerts (expiring in 14 days)
- **Operations:** Track PO status, upload new POs

**API endpoints:**
- `POST /api/purchase-orders/upload` — upload PO image
- `GET /api/purchase-orders/[id]` — PO detail (remanente, expiry)
- `GET /api/purchase-orders?deal_id=X` — all POs for deal
- `GET /api/purchase-orders/expiring` — POs expiring in 14 days
- `POST /api/purchase-orders/[id]/renew` — request new PO from client

**Routes:**
- `/deals/[id]/expenses` → PO tab
  - PO list: PO number, amount, remanente, expiry date, status
  - Upload button
  - Alert: "Expiring in 10 days" (yellow) or "EXPIRED" (red)
- `/dashboard/operaciones/purchase-orders` — PO management

**Cron jobs:**
- `po-remanente-alert` (weekly Friday 9 AM)

**Acceptance criteria:**
- PO remanente calculated correctly
- Expiry alerts sent 14 days before
- Expired POs flagged in red
- Operations can upload new POs

---

## Summary: Build Sequence & Parallel Work

### Critical Path:
1. Phase 0: 1 day (scaffold)
2. Phase 1: 3–4 days (auth + schema)
3. Phase 2: 4–5 days (conversations)
4. Phase 3: 5–6 days (CRM + quotes)
5. Phase 4: 6–8 days (agents)
6. Phase 5: 7–10 days (reporting + ERP)

**Total: ~26–40 days (5–8 weeks)**

### Parallel opportunities within phases:
- Phase 1: Auth (1.1) + Schema (1.2) can be built in parallel
- Phase 2: Inbox layout (2.1) + Slash commands (2.2) can be built in parallel
- Phase 3: Client/Deal views (3.1) + Quote builder (3.2) can be built in parallel
- Phase 4: Core agents (4.1) + Task system (4.2) + Call logging (4.3) can be built in parallel
- Phase 5: Dashboards (5.1) + Integrations (5.2) + ERP modules (5.3a–d) can be built in parallel

### Blockers by Phase:
- **Phase 1:** Q7 (roles definition) — can stub and update later
- **Phase 3:** Q1 (deal classification quota), Q2 (preplan approval authority)
- **Phase 5:** Q3 (arrastre formula), Q4 (Facturella API), Q5 (Medicati webhook)

**Next action:** Collect answers to Q1, Q2, Q7 before Phase 3 starts. Collect Q3, Q4, Q5 answers before Phase 5 starts.
- Q5 (Medicati spec) blocks results delivery

---

## Build Sequence Gantt (Estimated)

```
Week 1 (Mar 25–29)
  Phase 0     ──────────  [scaffold + setup]
  Phase 1     ──────────────────  [auth + schema]

Week 2 (Apr 1–5)
  Phase 1     ────  [role admin]
  Phase 2     ──────────  [inbox, nav, messages]
  Phase 3     ──────────  [clients, deals, list views]

Week 3 (Apr 8–12)
  Phase 2     ──  [completion]
  Phase 3     ──────────  [deal detail, quotation form]
  Phase 4     ──────────  [LangGraph, Supervisor]

Week 4 (Apr 15–19)
  Phase 3     ──  [quotation builder, completion]
  Phase 4     ──────────  [Follow Up, approval UI, intent router]
  Phase 5     ──────────  [Schema, Rep dashboard started]

Week 5 (Apr 22–26)
  Phase 4     ──  [Call logging, completion]
  Phase 5     ──────────────  [Dashboards, Reporting Agent]

Week 6+ (May onwards)
  Phase 5     ──────────────────  [Integrations: Zoho, Facturella, Medicati]
```

---

## Dependency Matrix

```
Phase 0 (Scaffold)
  └── Phase 1 (Auth)
        └── Phase 2 (Inbox)
        └── Phase 3 (CRM) — [depends on Q1, Q2, Q7]
              └── Phase 4 (Agents) — [depends on Phase 3]
                    └── Phase 5 (Reporting) — [depends on Q3, Q4, Q5]
```

**Critical path:** Phase 0 → Phase 1 → Phase 3 Quotation Builder (blocked by Q1, Q2)

---

## What Blocks Each Phase

| Phase | Blocker | Status | Unblock By |
|---|---|---|---|
| 0 | None | Ready | --- |
| 1 | None | Ready | --- |
| 2 | None | Ready | --- |
| 3 | Q1 (deal classification) | Open | Miriam + Omar |
| 3 | Q2 (preplan margin approval) | Open | Miriam |
| 3 | Q7 (role system definition) | Open | Miriam + Omar |
| 4 | Phase 3 complete | Blocked | Complete Phase 3 first |
| 5 | Q3 (arrastre calculation) | Open | Pablo + Omar |
| 5 | Q4 (Facturella API validation) | Open | Pablo |
| 5 | Q5 (Medicati webhook spec) | Open | Miriam → Diego |

---

## Implementation Guidelines

### Within a Phase

**Parallel work is OK:**
- If Module A and Module B have no dependencies on each other, build them simultaneously
- Example (Phase 3): Client list and Deal list can be built in parallel because they use independent schema

**Merge conflicts will happen:**
- When building in parallel, use `feature/` branches
- PRs are reviewed, tested in preview, then merged to `develop` in sequence
- Resolve conflicts explicitly (don't rely on auto-merge)

### Between Phases

**Always complete Phase N before starting Phase N+1:**
- Phase 0 → Phase 1 (must have auth before building features)
- Phase 2 → Phase 3 (nice to build in parallel, but Phase 3 assumes Phase 2 messaging works first)

**Use feature flags for soft phase boundaries:**
- If Phase 5 reporting is nearly done but Phase 4 agents aren't, flag dashboards as "beta" in UI
- Use `if (process.env.FEATURE_REPORTING === 'true')` to hide incomplete sections

---

## How to Track Progress

1. Each module in the table above corresponds to 1–2 GitHub issues
2. Issues are grouped by phase (use GitHub Projects for board view)
3. When a module is done, move issue to "Done" and reference the PR
4. When a blocker is resolved (Q1 answered), update this doc, then unblock the module

Example:
```
Issue: [GDT-42] Quotation Builder Form
Status: Blocked → needs Q1 + Q2
Blocker: Q1 (deal classification quota impact)
Unblock: When Miriam provides decision on Q1
```

