# GDT — Git Branching & Deployment Strategy

**Purpose:** Clear, documented workflow for code organization, branch hygiene, and environment promotion.

**Version:** 1.0.0  
**Last Updated:** 2026-03-25

---

## Branch Model

GDT uses a **modified Git Flow** tailored for two environments (develop + production) with feature/hotfix branches.

```
main (production)
  ├── hotfix/* (critical fixes from main)
  ├── v*.*.* (release tags)
  └─merge back to develop after hotfix

develop (staging/preview)
  ├── feature/* (feature branches off develop)
  ├── bugfix/* (bug fixes off develop)
  ├── chore/* (deps, docs, refactoring)
  └─merge back to develop after PR review
```

### Branch Naming Convention

Pattern: `{type}/{ticket-id}-{short-description}`

**Types:**
- `feature/` — new feature or capability (e.g., `feature/GDT-42-quotation-builder`)
- `bugfix/` — bug fix (e.g., `bugfix/GDT-15-call-logging-crash`)
- `chore/` — dependency updates, docs, refactoring, no feature change (e.g., `chore/upgrade-shadcn-components`)
- `hotfix/` — critical production fix (e.g., `hotfix/payment-calculation-error`)

**Examples:**
```
GOOD: feature/GDT-123-cotizacion-builder
GOOD: bugfix/GDT-45-transcript-spanish
GOOD: chore/update-nextjs-14-2
GOOD: hotfix/approval-queue-blank

BAD: new-feature
BAD: fix-stuff
BAD: pablo-work
```

---

## Branch Lifecycle

### 1. Creating a Feature Branch

```bash
git checkout develop
git pull origin develop
git checkout -b feature/GDT-{ticket}-{description}
```

**Rules:**
- Always branch off `develop`, never off `main` (except hotfix)
- Always pull latest `develop` before branching
- One logical feature per branch (not "add everything")
- Commit size: keep commits atomic (one idea per commit)

### 2. During Development

**Commit messages:**

> **Copilot rule:** Always suggest a commit message following the format below whenever committing code changes. Do not allow commits without a properly formatted message.

```
Type: Short description

Optional longer explanation if needed.
Fixes #GDT-123
```

**Types:** `feat`, `fix`, `docs`, `style`, `refactor`, `test`, `chore`

**Examples:**
```
feat: Add quotation builder form using ShadCN components
feat: Create ClientCard component with ShadCN Badge + Card
fix: Correct Spanish locale in transcript decoder
fix: Use organization_id in clients query RLS
docs: Update FLOWS.md with preplan margin rules
docs: Add component usage examples to copilot-instructions
style: Refactor button styling to use ShadCN Button
chore: Bump @supabase/ssr to 0.94.0
chore: Add ShadCN Recharts integration for analytics
```

**UI Component commits:** Always mention ShadCN when building components (`feat: Add ItemCard with ShadCN components`). This signals to reviewers to check for ShadCN compliance.

**Copilot directive:** When a file edit is complete, always output the suggested commit message in the format above before awaiting user confirmation (e.g., `feat: Add quotation builder form validation`). This ensures commits are always properly named.

**Pushing:**
```bash
git push origin feature/GDT-{ticket}-{description}
```

### 3. Pull Request → Review → Merge

**PR Requirements:**
- Title: `[GDT-123] Short description` (include ticket ID)
- Description: 2–3 sentences on what changed, why, what to test
- Assign reviewer(s) — typically you + Copilot-created code means: Pablo reviews logic, Miriam reviews business rules if applicable
- Mark `feature/*` PRs as "Draft" if still WIP, "Ready for Review" when done
- All tests passing (or explicitly noted if skipped)

**Review criteria:**
1. Code follows `.github/copilot-instructions.md` conventions
2. Database migrations are included if schema changed
3. Environment variables documented if new ones added
4. TypeScript strict mode compiles without errors
5. No hardcoded values (use constants)
6. **Supabase RLS policies updated if table access changed** — see [RLS_SETUP.md](/docs/RLS_SETUP.md#7-development-checklist) for checklist
7. **All UI components use ShadCN exclusively** — no custom components. Check [copilot-instructions.md#when-building-components](/.github/copilot-instructions.md#when-building-components) for rules
8. **Colors from ShadCN preset only** — no custom hex/rgb values. Check `globals.css` for available theme colors
9. **No invention** — imports use existing libraries, not custom implementations. Check [copilot-instructions.md#never-invent](/.github/copilot-instructions.md#never-invent)

**Merging:**
- Use "Squash and merge" for feature branches (keeps history clean)
- Use "Create a merge commit" for develop → main (preserves integration history)
- Delete branch after merge (GitHub setting: auto-delete)

### 4. Promotion to Production

**Path:** `feature/*` → `develop` (via PR) → `main` (via Release)

**Release from develop to main:**

```bash
# On local machine, checkout develop
git checkout develop
git pull origin develop

# Create release branch
git checkout -b release/v{major}.{minor}.{patch}

# Update version in package.json, CHANGELOG.md
# Commit these changes
git add package.json CHANGELOG.md docs/CHANGELOG.md
git commit -m "chore: Release v1.2.0"

# Merge to main
git checkout main
git pull origin main
git merge --no-ff release/v{major}.{minor}.{patch}

# Tag release
git tag -a v{major}.{minor}.{patch} -m "Version {major}.{minor}.{patch}"

# Push to main & tag
git push origin main
git push origin --tags

# Merge back to develop (in case hot fixes were in release)
git checkout develop
git merge --no-ff release/v{major}.{minor}.{patch}
git push origin develop

# Delete release branch
git branch -d release/v{major}.{minor}.{patch}
```

### 5. Production Hotfix

**Only for critical production bugs.**

```bash
# Branch off main, NEVER develop
git checkout main
git pull origin main
git checkout -b hotfix/GDT-{ticket}-{description}

# Make fix, test, commit
git add .
git commit -m "fix: Critical production issue description"

# Merge back to main
git checkout main
git merge --no-ff hotfix/GDT-{ticket}-{description}
git tag -a v{major}.{minor}.{patch+1} -m "Hotfix version"
git push origin main --tags

# Also merge back to develop (important!)
git checkout develop
git merge --no-ff hotfix/GDT-{ticket}-{description}
git push origin develop

# Delete hotfix branch
git branch -d hotfix/GDT-{ticket}-{description}
```

---

## Environment Mapping

| Environment | Branch | Deployment | When | Who |
|---|---|---|---|---|
| **Develop** | `develop` | Vercel Preview + Railway staging | Every push to develop | Auto (GitHub Actions) |
| **Production** | `main` (tags only) | Vercel Production + Railway production | On git tag v*.*.* | Manual promotion after approval |

### Vercel Configuration

- **Main branch** → Production environment (gdt.vercel.app)
- **develop branch** → Preview environment (preview-gdt.vercel.app)
- **Other branches** → Automatic preview per PR (visible in PR checks)

### Railway Configuration

- **main branch** → Production agent runtime
- **develop branch** → Staging agent runtime
- **Pull requests** → Optional preview runtime (if team agrees)

---

## Environment Variables Strategy

### Local Development

Create `.env.local` in `apps/web` and `apps/agents` (gitignored):

```
# .env.local — NEVER commit

# Supabase (staging)
NEXT_PUBLIC_SUPABASE_URL=https://your-staging-project.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=eyJhbGc...
SUPABASE_SERVICE_ROLE_KEY=eyJhbGc...

# OpenRouter (multi-model LLM provider)
OPENROUTER_API_KEY=sk-or-...
OPENAI_API_KEY=sk-... (optional fallback)
```

### Vercel

**Staging environment** (develop):
```
NEXT_PUBLIC_SUPABASE_URL=https://your-staging-project.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=eyJhbGc...   (staging anon key)
SUPABASE_SERVICE_ROLE_KEY=eyJhbGc...        (staging service role)
RAILWAY_AGENT_URL=https://staging-agents.railway.app
```

**Production environment** (main):
```
NEXT_PUBLIC_SUPABASE_URL=https://your-prod-project.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=eyJhbGc...   (production anon key)
SUPABASE_SERVICE_ROLE_KEY=eyJhbGc...        (production service role)
RAILWAY_AGENT_URL=https://agents.railway.app
```

> **Rule:** Never use production secrets locally. Always test code against staging Supabase first.

### GitHub Environments

Create two GitHub environments (`preview` and `production`) with their own secrets:

1. Go to repo Settings → Environments
2. Create `preview` environment
   - Add all staging secrets above
   - Protection rule: `develop` branch only
3. Create `production` environment
   - Add all production secrets above
   - Protection rule: `main` branch only + require approval

---

## Workflow Example: Adding a Feature

**Scenario:** You're building the quotation builder (feature GDT-42).

```bash
# 1. Create branch off develop
git checkout develop
git pull origin develop
git checkout -b feature/GDT-42-quotation-builder

# 2. Implement the feature across multiple commits
# (Each commit is a logical chunk)
git add apps/web/components/QuotationBuilder.tsx
git commit -m "feat: Add quotation builder form component"

git add apps/web/lib/quoteService.ts
git commit -m "feat: Implement quote calculation logic"

git add packages/database/migrations/040-quote-tables.sql
git commit -m "feat: Add quote tables to database schema"

# 3. Push when ready for feedback
git push origin feature/GDT-42-quotation-builder

# 4. Open PR in GitHub
# - Title: "[GDT-42] Add quotation builder form"
# - Description: "Allows reps to create quotes from preplan. Integrates margin validation..."
# - Link issue
# - Request reviewer(s)

# 5. CI runs automatically
# - TypeScript check PASS
# - Tests PASS (if applicable)
# - Vercel preview builds (link in PR) PASS

# 6. Review feedback → address → push new commits (don't squash yet)
git add .
git commit -m "fix: Address PR feedback on margin calculation"
git push origin feature/GDT-42-quotation-builder

# 7. Once approved: squash & merge in GitHub UI
# (GitHub will create one clean commit with your branch history)

# 8. Feature deployed to Vercel staging (develop environment)
# - Test at https://preview-gdt.vercel.app

# 9. When ready for production:
# - Create release branch
# - Merge develop → main → tag
# - Vercel auto-deploys main → Production
```

---

## CI/CD Pipeline Status

The GitHub Actions workflow automatically:
- Runs TypeScript compiler on all pull requests
- Runs tests (if they exist)
- Triggers Vercel preview build (visible in PR)
- On merge to main: triggers Vercel production deploy + Railway production deploy

**Current status:** See `.github/workflows/` in the repo for exact pipeline config.

---

## Common Questions

**Q: Can I push directly to develop?**  
A: No. Always use feature branches + PR. Direct pushes bypass code review.

**Q: What if develop and main drift apart?**  
A: They shouldn't. Use release branches to keep them in sync. If they do, we've broken workflow — flag it.

**Q: Can I merge my PR without approval?**  
A: No. At minimum: self-review + one other person (even if it's Copilot's review suggestion).

**Q: What if I need to push an urgent fix?**  
A: Use `hotfix/*` branching only for critical production bugs. Requires immediate review but skips the staging queue.

**Q: Do I need to create a GitHub issue for every feature?**  
A: Yes. Every PR should reference a GitHub issue (`Fixes #GDT-123`). This links code to requirements.

