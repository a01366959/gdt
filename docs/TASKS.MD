# GDT — Your Task Checklist

> These are **only** the tasks that require your manual action (accounts, credentials, config, testing, deploys). Everything else (code, schemas, scaffolds, tests, tooling) is handled in-repo.
>
> Mark tasks `[x]` when completed.

---

## Phase 0 — Accounts & Infrastructure

**Environment Strategy:** You need **3 separate environments** (develop/staging/production) across all platforms. This ensures test data never contaminates production.

| Platform | Environment | Purpose | Branch |
|---|---|---|---|
| Supabase | staging-project | Dev/QA, test data, Zoho sandbox creds | develop branch deploys here |
| Supabase | production-project | Live data, production Zoho creds | main branch deploys here |
| Vercel | Preview | Auto-deployed from feature/* + develop | develop deploys here |
| Vercel | Production | Main app, customer-facing | main branch deploys here |
| Railway | Staging | Agent runtime for develop | develop branch deploys here |
| Railway | Production | Agent runtime for main | main branch deploys here |
| GitHub | preview environment | Staging secrets + Supabase staging + Railway staging URLs |
| GitHub | production environment | Production secrets + Supabase production + Railway production URLs |

**Phase 0 Tasks:**

- [ ] **Create Supabase projects (BOTH):**
  - [ ] `gdt-staging` project (with pgvector enabled)
  - [ ] `gdt-production` project (with pgvector enabled)
  - [ ] In each: `CREATE EXTENSION vector;` in SQL editor
  - [ ] Save connection strings for both
  
- [ ] **Create Vercel projects:**
  - [ ] Link repo to Vercel account
  - [ ] Vercel auto-creates Preview environment from `develop` branch
  - [ ] Vercel auto-creates Production environment from `main` branch
  
- [ ] **Create Railway projects (BOTH):**
  - [ ] `gdt-agents-staging` (for agents runtime on develop)
  - [ ] `gdt-agents-production` (for agents runtime on main)
  - [ ] Both linked to same repo, but configured for different branches
  
- [ ] **Generate API Keys:**
  - [ ] OpenRouter API key (https://openrouter.ai, grants access to 100+ models)
  - [ ] Save to both staging + production Railway env vars as `OPENROUTER_API_KEY`
  - [ ] Alternative: Keep OpenAI key as fallback (`OPENAI_API_KEY`) for direct calls if needed
  
- [ ] **Create Zoho developer account:**
  - [ ] Register OAuth app (gets client ID + secret)
  - [ ] Zoho Sandbox credentials (for staging)
  - [ ] Zoho Production credentials (for production - if available)
  
- [ ] **Set up GitHub environments:**
  - [ ] Create `preview` environment with staging secrets:
    - `NEXT_PUBLIC_SUPABASE_URL` (staging)
    - `NEXT_PUBLIC_SUPABASE_ANON_KEY` (staging)
    - `SUPABASE_SERVICE_ROLE_KEY` (staging)
    - `RAILWAY_AGENT_URL=https://staging-agents.railway.app` (or actual URL)
    - `ZOHO_CLIENT_ID`, `ZOHO_CLIENT_SECRET` (sandbox)
  - [ ] Create `production` environment with production secrets:
    - `NEXT_PUBLIC_SUPABASE_URL` (production)
    - `NEXT_PUBLIC_SUPABASE_ANON_KEY` (production)
    - `SUPABASE_SERVICE_ROLE_KEY` (production)
    - `RAILWAY_AGENT_URL=https://agents.railway.app` (or actual URL)
    - `ZOHO_CLIENT_ID`, `ZOHO_CLIENT_SECRET` (production)
  
- [ ] **Set up domains (optional for MVP):**
  - [ ] Custom domain on Vercel Production (e.g., gdt.yourdomain.com)
  
- [ ] **Set up Figma token (optional, post-MVP):**
  - [ ] Create Figma personal access token for design sync

## Phase 0.5 — Project Setup from Zero

**Duration:** ~2 hours  
**Owner:** You (manual commands) + Copilot (code generation)  
**Goal:** Create GitHub repo, scaffold monorepo, install Next.js + ShadCN, configure Supabase clients

> **Important:** These are all manual steps. After this phase, code generation is handled by Copilot. Do not skip any steps — each one is required for the later phases.

### Step 1: Create GitHub Repository

**Goal:** Set up the central repository on GitHub

```bash
# 1. Go to https://github.com/new
# 2. Fill in:
#    - Repository name: gdt
#    - Description: "GDT AI Sales Operating System"
#    - Visibility: Private (if company account) or Public (if OSS)
#    - Initialize with README: Yes
#    - Add .gitignore: Node
#    - Choose license: MIT (optional)
# 3. Click "Create repository"

# 4. Clone to your local machine
cd ~/Desktop/FUFTW/GITHUB
git clone https://github.com/YOUR_USERNAME/gdt.git
cd gdt
```

**Expected Output:**
```
Cloning into 'gdt'...
remote: Counting objects: 3, done.
Receiving objects: 100% (3/3), done.
branch 'main' set to track 'origin/main'.
```

### Step 2: Initialize Monorepo (Turborepo + pnpm)

**Goal:** Create the Turborepo structure so we can build multiple packages

**Important:** You are still in the `gdt/` directory from Step 1. Do NOT create a new folder.

---

**Step 2a: Run the Turborepo generator**

Type this command in your terminal:

```bash
pnpm dlx create-turbo@latest . --skip-install
```

**What happens:**
1. System downloads Turborepo generator
2. System asks you ONE question (see below)
3. Turborepo creates folders + config files

**What you see (after ~10 seconds):**
```
> using pnpm as package manager

? Which template would you like to use? (Use arrow keys)
❯ Create a new monorepo
  Basic Monorepo
  ... (other options)
```

**What you do:** Press Enter to select "Create a new monorepo" (default, already highlighted)

**What happens next:**
```
✔ Downloaded Turborepo v2.x.x (created template from ...remote...)
✔ Created Turborepo scaffold:
  ✓ turbo.json (monorepo config)
  ✓ package.json (root package)
  ✓ pnpm-workspace.yaml (workspace definition)
  ✓ apps/ folder
  ✓ packages/ folder
```

**After that completes, check what was created:**

```bash
ls -la
```

**You should see:**
```
drwxr-xr-x   apps/
drwxr-xr-x   packages/
-rw-r--r--   turbo.json
-rw-r--r--   package.json
-rw-r--r--   pnpm-workspace.yaml
```

---

**Step 2b: Install all root-level dependencies**

Now type:

```bash
pnpm install
```

**What happens:**
- pnpm reads `package.json` and `pnpm-workspace.yaml`
- Downloads all dependencies
- Links workspace packages together
- Creates `pnpm-lock.yaml` (lockfile, auto-generated)

**What you see (takes ~30 seconds):**
```
Lockfile is up-to-date
Progress: resolved X, reused Y, downloaded Z in 12s
Packages in scope: (root), @repo/eslint-config, @repo/typescript-config, @repo/ui
  ✓ installed (all dependencies linked)
  ✓ 234 packages in 28s
```

**After install completes, verify it worked:**

```bash
ls -la | grep pnpm-lock
```

**You should see:**
```
-rw-r--r--  pnpm-lock.yaml
```

---

**What's in the monorepo now:**

```
gdt/
├── apps/
│   └── docs/              # Default example app (we'll replace this in Step 3)
│   └── web/               # Will create here in Step 3
├── packages/
│   ├── ui/                # Shared UI components + design tokens
│   ├── eslint-config/     # ESLint configuration (shared)
│   └── typescript-config/ # TypeScript config (shared)
├── turbo.json             # Turborepo settings (cache, pipeline)
├── package.json           # Root workspace dependencies
├── pnpm-workspace.yaml    # Defines workspace packages
└── pnpm-lock.yaml         # Dependency lockfile
```

---

**Ready for Step 3?** You should now have a clean monorepo structure. Step 3 will create the Next.js app inside `apps/web/`.

### Step 3: Create Next.js App in apps/web

**Goal:** Set up Next.js 14 with App Router

**Important:** You're still in the `gdt/` root directory from Step 2.

---

**Step 3a: Remove the default Turborepo example apps**

Type:

```bash
rm -rf apps/web apps/docs
```

**What happens:**
- Removes the default `apps/docs/` example app
- Removes the `apps/web/` placeholder (if it exists)
- Clears the directory for our Next.js app

**Verify it's gone:**

```bash
ls apps/
```

**You should see (empty or very few items):**
```
# Empty or near-empty now
```

---

**Step 3b: Navigate into the apps folder and create Next.js app**

Type:

```bash
cd apps
```

**Now you're in:** `gdt/apps/`

Type:

```bash
pnpm create next-app@latest web \
  --typescript \
  --tailwind \
  --app \
  --skip-install \
  --eslint \
  --import-alias='@/*'
```

**What happens:**
- Wizard guides you through Next.js setup
- Creates the `apps/web/` folder with a fresh Next.js project
- `--skip-install` means it won't run `pnpm install` yet (we do it manually next)

**What you see:**
```
✔ Would you like to use TypeScript? › Yes
✔ Would you like to use ESLint? › Yes
✔ Would you like to use Tailwind CSS? › Yes
✔ Would you like your code inside a `src/` directory? › No
✔ Would you like to use App Router? (recommended) › Yes
✔ Would you like to customize the import alias (`@/*` by default)? › No
✔ Create Next.js app in ./web
✔ Created next-app in /Users/pabloescalante/Desktop/FUFTW/GITHUB/gdt/apps/web
```

---

**Step 3c: Enter the web folder and install dependencies**

Type:

```bash
cd web
```

**Now you're in:** `gdt/apps/web/`

Type:

```bash
pnpm install
```

**What happens:**
- pnpm installs all Next.js dependencies
- Creates a `pnpm-lock.yaml` in the web folder
- Links to root workspace packages

**What you see (takes ~1 minute):**
```
Lockfile is up-to-date
Progress: resolved X, reused Y, downloaded Z
 ✓ linked 234 packages in 45s
 ✓ all dependencies installed
```

---

**Step 3d: Return to the repo root**

Type:

```bash
cd ../..
```

**Verify you're back in root:**

```bash
pwd
```

**You should see:**
```
/Users/pabloescalante/Desktop/FUFTW/GITHUB/gdt
```

---

**Step 3e: Verify the Next.js app structure**

Type:

```bash
ls apps/web/app/
```

**You should see:**
```
layout.tsx
page.tsx
globals.css
```

Type:

```bash
ls apps/web/ | head -10
```

**You should see:**
```
app/
components/
lib/
node_modules/
public/
.eslintrc.json
next.config.js
tsconfig.json
tailwind.config.ts
package.json
```

---

**What's created:**

```
apps/web/
├── app/                           # App Router pages (not /pages folder)
│   ├── globals.css                # Global styles + Tailwind
│   ├── layout.tsx                 # Root layout component
│   ├── page.tsx                   # Home page (/)
│   └── api/                       # API routes go here (e.g., /api/calls)
├── components/                    # React components
├── lib/                           # Utilities, helpers
├── public/                        # Static assets (images, icons)
├── .eslintrc.json                 # ESLint config
├── next.config.js                 # Next.js config
├── tsconfig.json                  # TypeScript config
├── tailwind.config.ts             # Tailwind CSS config
└── package.json                   # Next.js dependencies
```

---

**Ready for Step 4?** You now have a clean Next.js 14 app with App Router, TypeScript, Tailwind, and ESLint configured.

### Step 4: Create LangGraph Agents App

**Goal:** Set up the agent runtime (LangGraph) that deploys to Railway

**Important:** You're still in the `gdt/` root directory from Step 3. (The frontend `apps/web/` is already complete from Step 3.)

---

**Step 4a: Create agents folder structure**

Type:

```bash
mkdir -p apps/agents/{src,config}
```

**Step 4b: Create agents package.json**

Type:

```bash
cat > apps/agents/package.json << 'EOF'
{
  "name": "agents",
  "version": "0.1.0",
  "private": true,
  "scripts": {
    "dev": "tsx watch src/index.ts",
    "build": "tsc",
    "start": "node dist/index.js"
  },
  "dependencies": {
    "@langchain/langgraph": "^0.1.0",
    "@langchain/core": "^0.1.0",
    "dotenv": "^16.3.1"
  },
  "devDependencies": {
    "@types/node": "^20.0.0",
    "typescript": "^5.3.3",
    "tsx": "^4.0.0"
  }
}
EOF
```

**Step 4c: Copy agent prompts from packages/prompts/**

Type:

```bash
cp packages/prompts/*.md apps/agents/config/
```

**Verify prompts copied:**

```bash
ls apps/agents/config/
# Should output:
# follow-up-agent.system.md
# orchestrator-policy.md
# reporting-agent.system.md
# sales-assistant-agent.system.md
# supervisor-agent.system.md
# technical-agent.system.md
```

**Step 4d: Create agents environment file**

Type:

```bash
cat > apps/agents/.env.local << 'EOF'
# LangGraph Agent Runtime (staging)
OPENROUTER_API_KEY=sk-or-... (from https://openrouter.ai/keys)
SUPABASE_URL=https://your-staging-project.supabase.co
SUPABASE_SERVICE_ROLE_KEY=eyJhbGc... (staging service role key)

# Agent Configuration
AGENT_LOG_LEVEL=debug
AGENT_TIMEOUT=30000
EOF
```

**Step 4e: Return to repo root**

Type:

```bash
cd ../..
pwd
# Should output: /Users/pabloescalante/Desktop/FUFTW/GITHUB/gdt
```

**What's created:**

```
apps/agents/
├── src/                           # Agent code (created in Phase 1)
├── config/                        # Agent prompts (copied from packages/prompts/)
├── package.json                   # Agent dependencies
└── .env.local                     # Agent environment variables
```

---

**Ready for Step 5?** Both `apps/web/` and `apps/agents/` are now scaffolded.

### Step 5: Initialize ShadCN/ui Preset

**Goal:** Add ShadCN components with preset b2Cin5UWY (24+ preloaded components)

**Important:** You must be in `gdt/apps/web/` directory for this step.

---

**Step 5a: Navigate to the web app folder**

Type:

```bash
cd apps/web
```

**Verify you're in the right place:**

```bash
pwd
```

**You should see:**
```
/Users/pabloescalante/Desktop/FUFTW/GITHUB/gdt/apps/web
```

---

**Step 5b: Run ShadCN preset initialization**

Type:

```bash
pnpm dlx shadcn@latest init --preset b2Cin5UWY --template next
```

**What happens:**
- ShadCN checks your project (Next.js 14, Tailwind detected ✓)
- Downloads preset b2Cin5UWY from ShadCN registry
- Installs 24 component files into `components/ui/`
- Creates/updates `components.json` with preset config
- Updates `app/globals.css` with component styles

**What you see (takes ~30 seconds):**
```
✔ Found and verified Next.js
✔ Found and verified Tailwind CSS
✔ 
✔ Initializing components.json
✔ Applying preset: b2Cin5UWY
✔ Upgrading Tailwind config... 
✔ Installing dependencies...
 > pnpm add -w ... (installs component dependencies like @radix-ui, lucide-react, etc.)
✔ 
✔ Done. 24 components installed in components/ui/

> Installation completed successfully
```

---

**Step 5c: Verify the preset was applied**

Type:

```bash
cat components.json
```

**You should see (at minimum):**
```json
{
  "preset": "b2Cin5UWY",
  "$schema": "https://ui.shadcn.com/schema.json",
  "style": "default",
  "rsc": true,
  "tsx": true,
  "baseColor": "slate",
  "css": "./app/globals.css",
  "componentsDir": "./components/ui",
  ...
}
```

**Look for `"preset": "b2Cin5UWY"` — that's the key line.**

---

**Step 5d: Count the components installed**

Type:

```bash
ls components/ui/ | wc -l
```

**You should see:**
```
24
```

(The exact number may vary slightly, but should be 20+)

Type:

```bash
ls components/ui/ | head -20
```

**You should see files like:**
```
accordion.tsx
alert.tsx
alert-dialog.tsx
avatar.tsx
badge.tsx
button.tsx
card.tsx
checkbox.tsx
command.tsx
dialog.tsx
dropdown-menu.tsx
form.tsx
input.tsx
label.tsx
pagination.tsx
popover.tsx
progress.tsx
scroll-area.tsx
select.tsx
separator.tsx
...
```

---

**What you have now:**

| Category | Components |
|---|---|
| Layout | Card, Separator, Sidebar, NavigationMenu |
| Input | Button, Input, Textarea, Select, Checkbox, RadioGroup, Switch, Combobox |
| Display | Badge, Avatar, Alert, Progress, Skeleton, Spinner |
| Feedback | Tooltip, Popover, AlertDialog, Dialog, Sheet |
| Charts | Recharts (area, bar, line, pie, scatter, radar) |
| Navigation | DropdownMenu, Breadcrumb, Pagination |
| Data | Table, Tabs, Accordion |

All 24+ components are **already installed and ready to use**. You don't need to add them individually.

---

**Important:** DO NOT run `pnpm exec shadcn-ui@latest add <component>` for individual components. The preset includes everything needed for Phases 1-3.

---

**Loading States Pattern: Use Skeletons for Long Operations**

Whenever data fetching or loading takes more than 200ms, **always show a skeleton loader** instead of a blank screen or spinner.

**Available:**
- `Skeleton` component (ShadCN) — use for placeholder UI
- Pair with `Suspense` boundaries for streaming Server Components

**Example:**
```typescript
// Bad: User sees blank screen during load
export function ClientList() {
  const clients = await fetchClients() // Blocks rendering
  return <div>{clients.map(...)}</div>
}

// Good: Skeleton shown immediately, content streams in
export function ClientList() {
  return (
    <Suspense fallback={<ClientListSkeleton />}>
      <ClientListContent />
    </Suspense>
  )
}

// Skeleton component
function ClientListSkeleton() {
  return (
    <div className="space-y-2">
      <Skeleton className="h-12 w-full" />
      <Skeleton className="h-12 w-full" />
      <Skeleton className="h-12 w-full" />
    </div>
  )
}
```

**When to use Skeleton:**
- ✅ API calls > 200ms
- ✅ Database queries > 200ms
- ✅ Image loading
- ✅ List/table rows (show 3-5 skeleton rows)
- ❌ Don't use for < 100ms operations (just let it load instantly)

---

**Optional: Add Pre-built Blocks (Faster UI Development)**

Beyond components, ShadCN also offers **blocks** — pre-built, production-ready page sections you can copy-paste. The GDT project uses these three blocks:

**1. Login Block** — Complete authentication UI
```bash
pnpm dlx shadcn@latest add login-05
```
Provides: LoginForm component for `/auth/login` page

**2. Sidebar Block** — Navigation and admin structure
```bash
pnpm dlx shadcn@latest add sidebar-07
```
Provides: Sidebar component for dashboard layout, role-based navigation

**3. Settings Sidebar Block** — Preferences and account settings
```bash
pnpm dlx shadcn@latest add sidebar-13
```
Provides: SettingsSidebar component for `/settings` pages

**Usage Example:**

```typescript
import { LoginForm } from "@/components/login-form"
import { Sidebar } from "@/components/sidebar"

export default function DashboardLayout({
  children,
}: {
  children: React.ReactNode
}) {
  return (
    <div className="flex h-screen gap-4">
      <Sidebar />
      <main className="flex-1 overflow-auto">
        {children}
      </main>
    </div>
  )
}
```

Browse all available blocks at: https://ui.shadcn.com/blocks

For Phase 0.5 (MVP), only the three blocks above are installed. Add more blocks during Phase 2+ when building specific pages.

---

**Ready for Step 6?** You now have all ShadCN components preinstalled.

### Step 6: Create Folder Structure

**Goal:** Create all standard folders and initial utility files

**Important:** You must be in `gdt/apps/web/` directory for this step.

---

**Step 6a: Create all necessary folder structure**

Type (all one command, or paste the whole block):

```bash
mkdir -p lib utils constants
mkdir -p types/{api,db,ui}
mkdir -p hooks/{auth,api,db}
mkdir -p components/{auth,call,crm,agents,common}
mkdir -p app/{api,auth,dashboard,calls,tasks,inbox,reporting}
mkdir -p styles
```

**What happens:**
- Creates directories for components, hooks, utilities, types, etc.
- The `/` nested syntax creates parent folders automatically
- Example: `types/{api,db,ui}` creates `types/api/`, `types/db/`, `types/ui/`

---

**Step 6b: Create initial utility files**

Type:

```bash
touch lib/constants.ts lib/utils.ts
touch types/api/index.ts types/db/index.ts types/ui/index.ts
touch hooks/auth/useAuth.ts hooks/api/useApi.ts
```

**What happens:**
- Creates empty TypeScript files where you'll add code later
- Files are scaffolds — Copilot will fill them in during Phase 1

---

**Step 6c: Verify the folder structure**

Type:

```bash
tree -L 2 -I 'node_modules'
```

**If `tree` is not installed, use:**

```bash
find . -maxdepth 2 -type d | grep -v node_modules | sort
```

**You should see:**
```
apps/web/
├── app/
│   ├── (auth)/
│   ├── api/
│   ├── dashboard/
│   ├── inbox/
│   └── ... (other folders)
├── components/
│   ├── auth/
│   ├── call/
│   ├── crm/
│   ├── agents/
│   ├── common/
│   └── ui/ (ShadCN components from Step 4)
├── hooks/
│   ├── auth/
│   ├── api/
│   └── db/
├── lib/
├── types/
│   ├── api/
│   ├── db/
│   └── ui/
├── utils/
├── constants/
├── styles/
├── public/
├── node_modules/
├── components.json
├── tsconfig.json
├── package.json
├── next.config.js
└── tailwind.config.ts
```

---

**Final structure is ready:**

```
apps/web/
├── app/
│   ├── api/                       # API routes (e.g., POST /api/calls/summary)
│   ├── auth/                      # Auth pages (login, callback)
│   ├── dashboard/                 # Dashboard views (rep, manager, director)
│   ├── inbox/                     # Inbox & conversations
│   ├── tasks/                     # Task views
│   ├── reporting/                 # Reporting pages
│   ├── globals.css                # Global styles + Tailwind
│   ├── layout.tsx                 # Root layout
│   └── page.tsx                   # Home page
├── components/
│   ├── auth/                      # LoginForm, LogoutButton, etc.
│   ├── call/                      # CallPanel, CallControls, LiveTranscript, etc.
│   ├── crm/                       # ClientCard, DealRoom, ContactList, etc.
│   ├── agents/                    # AgentCard, ApprovalQueue, etc.
│   ├── common/                    # SharedHeader, Sidebar, etc.
│   └── ui/                        # ShadCN components (24+)
├── hooks/
│   ├── auth/                      # useAuth(), useLogin()
│   ├── api/                       # useApi(), useFetch()
│   └── db/                        # useSupabaseQuery()
├── lib/
│   ├── constants.ts               # APP_NAME, API_URLS, etc.
│   ├── utils.ts                   # Utility functions
│   └── supabase/ (created later)  # Supabase clients
├── types/
│   ├── api/                       # API response types
│   ├── db/                        # Database row types
│   └── ui/                        # Component prop types
├── utils/
├── constants/
├── styles/
└── public/                        # Static assets (icons, images)
```

**Ready for Step 7?** Your folder structure is complete. Next step sets up Tailwind design tokens.

### Step 7: Configure Tailwind + Design Tokens

**Goal:** Set up design system with GDT brand colors and tokens

**File: `apps/web/tailwind.config.ts`**

```typescript
import type { Config } from "tailwindcss"
import defaultTheme from "tailwindcss/defaultTheme"

const config = {
  darkMode: ["class"],
  content: [
    './components/**/*.{ts,tsx}',
    './app/**/*.{ts,tsx}',
  ],
  theme: {
    extend: {
      fontFamily: {
        sans: ["Inter Tight", ...defaultTheme.fontFamily.sans],
      },
      colors: {
        // Segment colors (client classification)
        segment: {
          charales: "#059669",   // Green
          truchas: "#0891b2",    // Cyan
          atunes: "#f59e0b",     // Amber
          tiburones: "#dc2626",  // Red
          ballenas: "#7c3aed",   // Violet
        },
        // Deal stage colors
        stage: {
          prospecto: "#94a3b8",      // Slate
          descubrimiento: "#0ea5e9", // Sky
          cotizacion: "#f59e0b",     // Amber
          negociacion: "#ef4444",    // Red
          cerrado_ganado: "#10b981", // Emerald (Win)
          cerrado_perdido: "#6b7280", // Gray (Loss)
        },
        // Invoice/Payment colors
        invoice: {
          pendiente: "#f59e0b",   // Amber
          pagado: "#10b981",      // Emerald
          mora: "#ef4444",        // Red
          vencido: "#991b1b",     // Dark Red
          cancelado: "#6b7280",   // Gray
        },
      },
    },
  },
  plugins: [require("tailwindcss-animate")],
} satisfies Config

export default config
```

### Step 8: Configure Supabase Clients

**Goal:** Create typed Supabase client files for server + client contexts

**File: `apps/web/lib/supabase/client.ts`** (browser client)

```typescript
import { createBrowserClient } from '@supabase/ssr'

export const createClient = () =>
  createBrowserClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!
  )
```

**File: `apps/web/lib/supabase/server.ts`** (server-only client)

```typescript
import { createServerClient, parseCookieHeader, serializeCookieHeader } from '@supabase/ssr'
import { cookies } from 'next/headers'

export const createClient = async () => {
  const cookieStore = await cookies()

  return createServerClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_SERVICE_ROLE_KEY!,
    {
      cookies: {
        getAll() {
          return parseCookieHeader(cookieStore.toString())
        },
        setAll(cookiesToSet) {
          cookiesToSet.forEach(({ name, value, options }) =>
            cookieStore.set(name, value, options)
          )
        },
      },
    }
  )
}
```

### Step 9: Create Environment Files

**Goal:** Set up .env files (staging/production)

**File: `apps/web/.env.local`** (local development — gitignored)

```env
# Supabase (staging project for local dev)
NEXT_PUBLIC_SUPABASE_URL=https://your-staging-project.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=eyJhbGc... (staging anon key)
SUPABASE_SERVICE_ROLE_KEY=eyJhbGc... (staging service role key)

# OpenRouter (supports 100+ models)
OPENROUTER_API_KEY=sk-or-... (from https://openrouter.ai/keys)
OPENAI_API_KEY=sk-... (optional fallback)

# Railway agents (staging)
RAILWAY_AGENT_URL=https://staging-agents.railway.app

# Feature flags
FEATURE_REPORTING=true
FEATURE_INTEGRATIONS=false
```

**File: `apps/web/.env.production`** (for reference, never commit secrets)

```env
# NOTE: Production secrets are set in Vercel dashboard, NOT here
# See: Vercel Project Settings → Environment Variables
NEXT_PUBLIC_SUPABASE_URL=*** (set in Vercel dashboard)
NEXT_PUBLIC_SUPABASE_ANON_KEY=*** (set in Vercel dashboard)
SUPABASE_SERVICE_ROLE_KEY=*** (set in Vercel dashboard)
```

**Mark in .gitignore:**
```
# .gitignore (should already have this)
.env
.env.local
.env.*.local
```

### Step 10: Create Global Styles

**File: `apps/web/app/globals.css`**

```css
@tailwind base;
@tailwind components;
@tailwind utilities;

/* CSS variables for consistent dark mode */
@layer base {
  :root {
    --background: 0 0% 100%;
    --foreground: 0 0% 3.6%;
    --muted: 0 0% 96.1%;
    --muted-foreground: 0 0% 45.1%;
    --destructive: 0 84.2% 60.2%;
    --border: 0 0% 89.8%;
    --input: 0 0% 89.8%;
    --ring: 0 0% 3.6%;
    --radius: 0.5rem;
  }

  .dark {
    --background: 0 0% 3.6%;
    --foreground: 0 0% 98%;
    --muted: 0 0% 14.9%;
    --muted-foreground: 0 0% 63.9%;
    --destructive: 0 84.2% 60.2%;
    --border: 0 0% 14.9%;
    --input: 0 0% 14.9%;
    --ring: 0 0% 83.1%;
  }
}

/* Typography */
@layer base {
  body {
    @apply bg-background text-foreground;
    font-feature-settings: "rlig" 1, "calt" 1;
  }

  h1 {
    @apply scroll-m-20 text-4xl font-extrabold tracking-tight lg:text-5xl;
  }

  h2 {
    @apply scroll-m-20 text-3xl font-semibold tracking-tight first:mt-0;
  }
}

/* Smooth transitions */
@layer components {
  .transition-smooth {
    @apply transition-all duration-200 ease-in-out;
  }
}
```

### Step 11: Create Copilot Instructions

**Goal:** Import the Copilot instructions so VS Code Copilot knows GDT conventions

```bash
# From repo root
mkdir -p .github
# Copy COPILOT.md to .github/copilot-instructions.md
cp docs/COPILOT.md .github/copilot-instructions.md
```

**Verify it's in place:**
```bash
ls -la .github/copilot-instructions.md
# Should show: -rw-r--r--  1 user  staff  X .github/copilot-instructions.md
```

### Step 12: Add All Docs to Repo

**Goal:** Copy all specification documents into /docs

```bash
# Ensure all docs are in place
ls docs/
# Should output:
# ARCHITECTURE.md
# BRANCHING.md
# CHANGELOG.md
# COPILOT.md
# EPICS.md
# FLOWS.md
# MODULES.md
# PRD.md
# QUESTIONS.md
# TASKS.md (this file)
```

### Step 13: Commit Initial Scaffold

**Goal:** Push the clean scaffold to git

```bash
# From repo root (gdt/)
git add .
git commit -m "feat: Initial project scaffold (Turborepo, Next.js, ShadCN, Supabase clients)"

# Push to main (since this is the seed commit)
git push origin main
```

**Expected Output:**
```
On branch main
Your branch is ahead of 'origin/main' by 1 commit.
  (use "git push" to publish your local commits)
```

### Step 14: Verify Everything Builds

**Goal:** Make sure no TypeScript errors before handing off to Copilot

```bash
# From repo root
pnpm install  # Make sure dependencies are resolved
pnpm build    # Compile TypeScript
pnpm dev      # Start dev server (should NOT error)
```

**Expected Output - Build:**
```
> turbo build

▲ Next.js 14.0.4

  ▲ Build output is zero runtime JavaScript
  ✓ Generated: 42 edge functions
  ✓ Compiled successfully
```

**Expected Output - Dev:**
```
▲ Next.js 14.0.4

  ▲ Local:        http://localhost:3000
  ▲ Environments: .env.local

  ✓ Ready in 2.8s
```

> **Stop the dev server** (Ctrl+C) — you're not running locally, all testing happens in Vercel preview.

### Step 15: Create Initial Branches

**Goal:** Set up branch protection and environments

```bash
# Create develop branch locally
git checkout -b develop

# Push develop to remote
git push -u origin develop

# Now go to GitHub https://github.com/YOUR_USERNAME/gdt/settings/branches
# 1. Add branch protection rule:
#    - Branch name: main
#    - Require pull request reviews before merging: YES (1 required)
#    - Include administrators: YES
# 2. Add branch protection rule:
#    - Branch name: develop
#    - Require pull request reviews before merging: YES (1 required)
#    - Require status checks to pass: YES
```

### Step 16: Configure PWA (Progressive Web App for Mobile)

**Goal:** Set up offline support, mobile home screen install, and push notifications

**Substep 16a: Create Service Worker**

```bash
# From apps/web root
cat > public/sw.js << 'EOF'
const CACHE_NAME = 'gdt-v1'
const CRITICAL_ASSETS = ['/', '/inbox', '/tasks', '/deals', '/offline.html']

self.addEventListener('install', (e) => {
  e.waitUntil(caches.open(CACHE_NAME).then((c) => c.addAll(CRITICAL_ASSETS)))
})

self.addEventListener('fetch', (e) => {
  if (e.request.url.includes('/api/')) {
    e.respondWith(fetch(e.request).catch(() => new Response(JSON.stringify({error:'Offline'}), {status:503})))
  } else {
    e.respondWith(caches.match(e.request).then((r) => r || fetch(e.request)))
  }
})
EOF
```

**Substep 16b: Create Web App Manifest**

```bash
# From apps/web root
cat > public/manifest.json << 'EOF'
{
  "name": "GDT — Sales Operating System",
  "short_name": "GDT",
  "description": "AI-native sales workspace for follow-ups, deals, and tasks",
  "start_url": "/inbox",
  "scope": "/",
  "display": "standalone",
  "orientation": "portrait",
  "background_color": "#ffffff",
  "theme_color": "#059669",
  "icons": [
    {"src": "/icon-192x192.png", "sizes": "192x192", "type": "image/png", "purpose": "any"},
    {"src": "/icon-512x512.png", "sizes": "512x512", "type": "image/png", "purpose": "any"},
    {"src": "/icon-192x192-maskable.png", "sizes": "192x192", "type": "image/png", "purpose": "maskable"},
    {"src": "/icon-512x512-maskable.png", "sizes": "512x512", "type": "image/png", "purpose": "maskable"}
  ],
  "shortcuts": [
    {"name": "Inbox", "short_name": "Inbox", "url": "/inbox", "icons": [{"src": "/icon-96x96.png", "sizes": "96x96"}]},
    {"name": "Tasks", "short_name": "Tasks", "url": "/tasks", "icons": [{"src": "/icon-96x96.png", "sizes": "96x96"}]},
    {"name": "Log Call", "short_name": "Call", "url": "/call", "icons": [{"src": "/icon-96x96.png", "sizes": "96x96"}]}
  ]
}
EOF
```

**Substep 16c: Create PWA Icons**

```bash
# Option 1: Generate automatically using online tool
# Go to https://www.favicon-generator.org, upload a 512x512 PNG, download all sizes to public/

# Option 2: Use npm script (manual generation)
npm install --save-dev sharp
# Create scripts/generate-icons.js with sharp to generate all sizes from source image
# Then run: node scripts/generate-icons.js /path/to/source-512.png

# Verify all icons exist:
ls public/icon-*.png | wc -l
# Should output: 8 (72, 96, 128, 144, 152, 192, 384, 512) × 2 (maskable + regular)
```

**Expected Output:**
```
icon-72x72.png
icon-96x96.png
icon-128x128.png
icon-144x144.png
icon-152x152.png
icon-192x192.png
icon-384x384.png
icon-512x512.png
icon-192x192-maskable.png
icon-512x512-maskable.png
```

**Substep 16d: Update next.config.mjs for PWA**

```bash
# apps/web/next.config.mjs
cat >> next.config.mjs << 'EOF'

// PWA support
nextConfig.headers = async () => {
  return [
    {
      source: '/:path*',
      headers: [
        {
          key: 'Service-Worker-Allowed',
          value: '/'
        },
        {
          key: 'Cache-Control',
          value: 'public, max-age=31536000, immutable'
        }
      ]
    }
  ]
}

export default nextConfig
EOF
```

**Substep 16e: Create Offline Page**

```bash
# apps/web/app/offline.tsx
cat > app/offline.tsx << 'EOF'
export default function OfflinePage() {
  return (
    <div className="flex items-center justify-center h-screen bg-yellow-50">
      <div className="text-center">
        <h1 className="text-2xl font-bold text-yellow-900">📡 No Connection</h1>
        <p className="text-yellow-700 mt-4">You're offline. Some features are unavailable.</p>
        <p className="text-sm text-yellow-600 mt-2">Recent messages and tasks are cached locally.</p>
      </div>
    </div>
  )
}
EOF
```

**Verification:**
```bash
# Test locally with HTTP (not HTTPS needed for localhost dev)
pnpm dev

# In DevTools (F12):
# 1. Go to Application → Service Workers
# 2. Verify "sw.js" shows as "activated and running"
# 3. Go to Application → Manifest
# 4. Verify manifest.json loads with all icons
# 5. Go to Application → Cache Storage
# 6. Verify "gdt-v1" cache has critical assets
```

---

## Phase 1 — Verify Scaffold

- [ ] Run `pnpm install` at repo root and confirm no errors
- [ ] Run `pnpm build` and confirm all packages build
- [ ] Verify ShadCN components are in `apps/web/components/ui/` (should have 20+ files)
- [ ] Check that `.github/copilot-instructions.md` exists
- [ ] Verify all docs files are in `/docs` folder
- [ ] Verify `main` and `develop` branches exist on GitHub with protection rules
- [ ] **Verify PWA:** Service worker registered, manifest loads, icons present, offline page works

## Phase 2 — Apply Database

- [ ] Create Supabase projects (staging and production)
- [ ] Enable pgvector extension in both projects (`CREATE EXTENSION vector;` in SQL editor)
- [ ] Run migrations against staging Supabase project (SQL files in `packages/database/migrations/` — auto-generated by Copilot)
- [ ] Verify tables exist in Supabase dashboard

### Step 1: Apply RLS Policies (Phase 1 Foundation)

**Goal:** Enable Row Level Security on all tables and apply org-scoped + role-based policies.

**Reference:** [RLS_SETUP.md](/docs/RLS_SETUP.md) (complete guide with all SQL templates)

```bash
# In Supabase SQL Editor (staging project):

# 1. Enable RLS on all tables (from RLS_SETUP.md Section 5, Step 1)
# Paste the full "Enable RLS on all tables" SQL block

# 2. Create foundation policies (from RLS_SETUP.md Section 5, Step 2)
# Paste the complete policies for organizations, users, clients, contacts, deals, conversations, messages, tasks, agent_actions, calls
```

**Expected output:**
```
Query executed successfully (0 ms)
```

**Verification:**
```bash
# In Supabase SQL Editor, verify policies exist:
SELECT tablename, count(*) as policy_count
FROM pg_policies
WHERE schemaname = 'public'
GROUP BY tablename
ORDER BY tablename;

# Expected: Every table should have at least 1 policy
```

**Testing:**
```bash
# Test as anon (no auth)
SELECT * FROM clients;
# Expected: 0 rows (RLS blocks unauthenticated access)

# Test as authenticated user
# Set Auth Preview in Supabase Studio SQL editor to use test JWT
SELECT * FROM clients;
# Expected: Only rows in user's organization
```

See [RLS_SETUP.md Section 6](/docs/RLS_SETUP.md#6-testing-rls-policies) for detailed testing instructions.

### Step 2: Test RLS in Development

- [ ] Query as anon vs authenticated user (RLS blocks anon access)
- [ ] Query as different roles (rep, manager, director) — verify role-based policies
- [ ] Test cross-org access (should be blocked)
- [ ] Verify TaskSwipeCard, ApprovalCard components work with RLS-filtered data

## Phase 3 — Connect Services

- [ ] Create Vercel project, link to gdt repo, set environment variables (see BRANCHING.md)
- [ ] Create Railway project for `apps/agents`, link to repo, set `OPENROUTER_API_KEY` env var
- [ ] Verify Railway CI/CD pipeline deploys on push
- [ ] Verify Vercel CI/CD pipeline deploys on push
- [ ] Create GitHub environments (`preview` and `production`) with their respective secrets
- [ ] Verify RLS policies are replicated to production Supabase (compare staging vs production policy counts in Section 8 of RLS_SETUP.md)

## Phase 4 — Validate & Test (Local)

- [ ] Send a test message in inbox → confirm Realtime subscription is active
- [ ] Create a test client → verify it appears in client list
- [ ] Create a test deal → verify it appears in pipeline dashboard
- [ ] Check Vercel preview deployment shows no TypeScript errors
- [ ] Verify Railway staging deployment logs show no auth errors

## Phase 5 — Connect Integrations (Phase 2 onwards)

*These are blocked by open questions in QUESTIONS.md. Do not start until clarifications are received.*

- [ ] Validate Facturella API credentials (Q4)
- [ ] Validate Medicati webhook spec (Q5)
- [ ] Validate Zoho OAuth credentials
- [ ] Test Zoho → Supabase incremental sync (15-min cron)
- [ ] Test Supabase tasks → Zoho write-back (5-min cron)

## Phase 6 — Launch Prep

- [ ] Security review: Verify all RLS policies, API key rotation settings
- [ ] Performance check: Page load times, Realtime latency, API response times
- [ ] User acceptance testing with Miriam (manager) + 1–2 reps
- [ ] Promote staging environment to production (flip Supabase/Vercel/Railway to prod secrets)
- [ ] Final data migration from Zoho (one-time export)
- [ ] Monitor live usage (first 48 hours)

---

## How to Use This Document

1. **Phase 0** — Accounts & Infrastructure (you create Supabase/Vercel/Railway projects before coding starts)
2. **Phase 0.5** — Project Setup from Zero (detailed step-by-step with terminal commands and examples)
3. **Phase 2+** — All code generation is handled by Copilot

## Key Reference Documents

Before starting Phase 0.5, you must have these documents in `/docs`:
- **BRANCHING.md** — Git workflow (how to create branches, commit, push, create PRs)
- **MODULES.md** — Which modules build in which order (dependencies chart)
- **QUESTIONS.md** — Open business questions that block certain modules
- **FLOWS.md** — Complete business process flows (what happens in each workflow)
- **COPILOT.md** — Copilot coding instructions (copied to `.github/copilot-instructions.md`)

## Notes

- **Phases 0–0.5** are manual (you run these commands)
- **Phases 2+** require Supabase/Vercel/Railway accounts (set up in Phase 0 first)
- All code generation is handled by Copilot — do not write code manually
- Every PR must reference a GitHub issue (`Fixes #GDT-123`)
- Mark tasks `[x]` as you complete them
- If stuck on a question, check QUESTIONS.md — it may already be documented
- Keep CHANGELOG.md updated after every PR merge
