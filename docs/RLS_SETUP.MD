# GDT — Row Level Security (RLS) Setup Guide

**Purpose:** Complete reference for implementing and testing Supabase Row Level Security policies across all GDT tables.

**Version:** 1.0.0  
**Last Updated:** 2026-03-25  
**Status:** Implementation-ready

---

## 1. RLS Philosophy

Row Level Security (RLS) in Supabase PostgreSQL ensures **data isolation** at the database layer—no row is accessible to any query unless an explicit policy permits it.

### Design Principles

| Principle | Rationale | Example |
|---|---|---|
| **Org-scoped** | All rows must belong to exactly one `organization_id`. Users can only access rows in their org. | Rep can only see clients, deals, tasks belonging to their company |
| **Role-scoped** | Some tables have role-based visibility rules. Only certain roles can perform certain operations. | Only `manager` role can approve quotes; `rep` can only edit own tasks |
| **Deny by default** | RLS denies all access unless an explicit **Allow** policy exists. | If no policy exists, even authenticated users get zero rows |
| **Service role bypass** | `SUPABASE_SERVICE_ROLE_KEY` bypasses RLS for server-only operations (migrations, data fixes). Never use in frontend. | Backend can bulk-import from Zoho without policy conflicts |

### RLS vs Authentication

- **Authentication** = "Who are you?" (handled by Supabase Auth JWT)
- **RLS** = "What data can you see?" (enforced by PostgreSQL policies)

Both are required. RLS is the enforcement layer.

---

## 2. Prerequisites

Before applying RLS:

1. **All tables must have `organization_id` column:**
   ```sql
   ALTER TABLE {table_name} ADD COLUMN organization_id UUID NOT NULL REFERENCES organizations(id);
   CREATE INDEX idx_{table_name}_org_id ON {table_name}(organization_id);
   ```

2. **All tables must have `created_at` and `updated_at` timestamps:**
   ```sql
   ALTER TABLE {table_name} ADD COLUMN created_at TIMESTAMP DEFAULT now();
   ALTER TABLE {table_name} ADD COLUMN updated_at TIMESTAMP DEFAULT now();
   ```

3. **RLS must be ENABLED on the table:**
   ```sql
   ALTER TABLE {table_name} ENABLE ROW LEVEL SECURITY;
   ```

4. **All existing INSERT/UPDATE/DELETE operations must be via authenticated users with valid JWT.**

---

## 3. Standard RLS Policy Templates

### Pattern: Org-Scoped Readable by All Members

**Use for:** `clients`, `contacts`, `deals`, `conversations`, `messages`, `tasks`, `calls`

**Policy template:**
```sql
-- Allow authenticated users to see rows in their organization
CREATE POLICY "users_can_view_org_rows" ON {table_name}
  FOR SELECT
  USING (organization_id = auth.jwt()->>'org_id');

-- Allow authenticated users to insert rows into their organization
CREATE POLICY "users_can_insert_into_org" ON {table_name}
  FOR INSERT
  WITH CHECK (organization_id = auth.jwt()->>'org_id');

-- Allow authenticated users to update rows in their organization
CREATE POLICY "users_can_update_org_rows" ON {table_name}
  FOR UPDATE
  USING (organization_id = auth.jwt()->>'org_id')
  WITH CHECK (organization_id = auth.jwt()->>'org_id');

-- Allow authenticated users to delete rows in their organization
CREATE POLICY "users_can_delete_org_rows" ON {table_name}
  FOR DELETE
  USING (organization_id = auth.jwt()->>'org_id');
```

**JWT structure required:**
```json
{
  "sub": "user-id-uuid",
  "email": "rep@gdt.com",
  "org_id": "org-uuid",
  "role": "rep",
  "iat": 1234567890
}
```

---

### Pattern: Org-Scoped + Owner-Edit Only

**Use for:** `tasks` (reps can only edit their own tasks), `expense_plans` (reps can only edit own expense plans)

**Policy template:**
```sql
-- Others in org can view
CREATE POLICY "org_view_tasks" ON {table_name}
  FOR SELECT
  USING (organization_id = auth.jwt()->>'org_id');

-- Owner can edit
CREATE POLICY "owner_can_update_task" ON {table_name}
  FOR UPDATE
  USING (
    organization_id = auth.jwt()->>'org_id'
    AND assigned_to_user_id = auth.uid()
  )
  WITH CHECK (
    organization_id = auth.jwt()->>'org_id'
    AND assigned_to_user_id = auth.uid()
  );

-- Managers can also edit any task in org
CREATE POLICY "manager_can_update_any_task" ON {table_name}
  FOR UPDATE
  USING (
    organization_id = auth.jwt()->>'org_id'
    AND auth.jwt()->>'role' = 'manager'
  )
  WITH CHECK (
    organization_id = auth.jwt()->>'org_id'
    AND auth.jwt()->>'role' = 'manager'
  );
```

---

### Pattern: Org-Scoped + Role-Based Approval Gate

**Use for:** `quotes` (only manager can approve), `deal_stage_changes` (only manager can move to Won/Lost)

**Policy template:**
```sql
-- All org members can view
CREATE POLICY "org_view_quotes" ON {table_name}
  FOR SELECT
  USING (organization_id = auth.jwt()->>'org_id');

-- Creators and managers can insert
CREATE POLICY "quote_insert_policy" ON {table_name}
  FOR INSERT
  WITH CHECK (
    organization_id = auth.jwt()->>'org_id'
    AND (
      auth.jwt()->>'role' IN ('rep', 'manager')
      OR created_by_user_id = auth.uid()
    )
  );

-- Only manager can approve (change approval_status)
CREATE POLICY "manager_can_approve_quotes" ON {table_name}
  FOR UPDATE
  USING (
    organization_id = auth.jwt()->>'org_id'
    AND auth.jwt()->>'role' = 'manager'
  )
  WITH CHECK (
    organization_id = auth.jwt()->>'org_id'
    AND auth.jwt()->>'role' = 'manager'
  );

-- Creator can modify pending quotes
CREATE POLICY "creator_can_edit_pending_quote" ON {table_name}
  FOR UPDATE
  USING (
    organization_id = auth.jwt()->>'org_id'
    AND created_by_user_id = auth.uid()
    AND approval_status = 'pending'
  )
  WITH CHECK (
    organization_id = auth.jwt()->>'org_id'
    AND created_by_user_id = auth.uid()
    AND approval_status = 'pending'
  );
```

---

### Pattern: Director-Only Access

**Use for:** `financial_summaries` (director-only dashboards), `revenue_forecasts`

**Policy template:**
```sql
-- Only directors can view
CREATE POLICY "director_view_financials" ON {table_name}
  FOR SELECT
  USING (
    organization_id = auth.jwt()->>'org_id'
    AND auth.jwt()->>'role' = 'director'
  );

-- Only directors can insert/update
CREATE POLICY "director_edit_financials" ON {table_name}
  FOR INSERT
  WITH CHECK (
    organization_id = auth.jwt()->>'org_id'
    AND auth.jwt()->>'role' = 'director'
  );

CREATE POLICY "director_update_financials" ON {table_name}
  FOR UPDATE
  USING (
    organization_id = auth.jwt()->>'org_id'
    AND auth.jwt()->>'role' = 'director'
  )
  WITH CHECK (
    organization_id = auth.jwt()->>'org_id'
    AND auth.jwt()->>'role' = 'director'
  );
```

---

## 4. RLS Policy Index by Table

| Table | Pattern | Select | Insert | Update | Delete |
|---|---|---|---|---|
| `organizations` | Org-only | ✅ | ❌ (service role only) | ❌ (service role) | ❌ (service role) |
| `users` | Org-only | ✅ | ❌ (manager/service role) | ⚠️ (self + manager) | ❌ (service role) |
| `clients` | Org-scoped | ✅ All | ✅ All | ✅ All | ✅ Manager |
| `contacts` | Org-scoped | ✅ All | ✅ All | ✅ All | ✅ Manager |
| `deals` | Org-scoped | ✅ All | ✅ Rep+ | ⚠️ Manager gates Won/Lost | ✅ Manager |
| `tasks` | Org + owner | ✅ All | ✅ All | ⚠️ Owner/Manager | ⚠️ Owner/Manager |
| `quotes` | Org + manager approval | ✅ All | ✅ All | ⚠️ Manager approves | ✅ Manager |
| `conversations` | Org-scoped | ✅ All | ✅ Agent system | ⚠️ Owner/Agent | ❌ (archive only) |
| `messages` | Org-scoped | ✅ All | ✅ System/Agent | ⚠️ Edit within 5 min | ❌ (soft-delete) |
| `agent_actions` | Org-scoped | ✅ All | ✅ Agent | ❌ (service role) | ❌ (service role) |
| `calls` | Org-scoped | ✅ All | ✅ Rep/System | ✅ Owner/Manager | ❌ (archive) |
| `expense_plans` | Org + owner | ✅ All | ✅ All | ⚠️ Owner/Manager | ✅ Manager |
| `invoices` | Org-scoped | ✅ All | ✅ System | ❌ (immutable) | ❌ (archive) |
| `payments` | Org-scoped | ✅ All | ✅ System | ❌ (immutable) | ❌ (archive) |
| `financial_summaries` | Director-only | ✅ Director | ✅ System | ✅ System | ❌ (archive) |

**Legend:** ✅ = Allowed | ❌ = Blocked | ⚠️ = Role/state-gated

---

## 5. Setup Instructions (Phase 1)

### Step 1: Enable RLS on All Tables

```sql
-- Run in Supabase SQL Editor for EACH table

-- Auth & org tables
ALTER TABLE organizations ENABLE ROW LEVEL SECURITY;
ALTER TABLE users ENABLE ROW LEVEL SECURITY;

-- CRM tables
ALTER TABLE clients ENABLE ROW LEVEL SECURITY;
ALTER TABLE contacts ENABLE ROW LEVEL SECURITY;
ALTER TABLE deals ENABLE ROW LEVEL SECURITY;

-- Conversation & messaging
ALTER TABLE conversations ENABLE ROW LEVEL SECURITY;
ALTER TABLE messages ENABLE ROW LEVEL SECURITY;

-- Task & approval
ALTER TABLE tasks ENABLE ROW LEVEL SECURITY;
ALTER TABLE approvals ENABLE ROW LEVEL SECURITY;

-- Agents & calls
ALTER TABLE agent_actions ENABLE ROW LEVEL SECURITY;
ALTER TABLE calls ENABLE ROW LEVEL SECURITY;

-- Expand as you add tables (quotes, expenses, invoices, etc.)
```

### Step 2: Create Foundation Policies for Phase 1

```sql
-- ===== ORGANIZATIONS TABLE (service role only) =====
CREATE POLICY "service_role_access_orgs" ON organizations
  FOR ALL
  USING (auth.role() = 'service_role');

-- ===== USERS TABLE =====
-- Users can see themselves + org members
CREATE POLICY "users_view_org_members" ON users
  FOR SELECT
  USING (organization_id = auth.jwt()->>'org_id');

-- Managers can update user roles
CREATE POLICY "managers_can_manage_users" ON users
  FOR UPDATE
  USING (
    organization_id = auth.jwt()->>'org_id'
    AND auth.jwt()->>'role' = 'manager'
  )
  WITH CHECK (
    organization_id = auth.jwt()->>'org_id'
    AND auth.jwt()->>'role' = 'manager'
  );

-- ===== CLIENTS TABLE =====
CREATE POLICY "clients_org_access" ON clients
  FOR SELECT
  USING (organization_id = auth.jwt()->>'org_id');

CREATE POLICY "clients_insert_org" ON clients
  FOR INSERT
  WITH CHECK (organization_id = auth.jwt()->>'org_id');

CREATE POLICY "clients_update_org" ON clients
  FOR UPDATE
  USING (organization_id = auth.jwt()->>'org_id')
  WITH CHECK (organization_id = auth.jwt()->>'org_id');

CREATE POLICY "clients_delete_manager_only" ON clients
  FOR DELETE
  USING (
    organization_id = auth.jwt()->>'org_id'
    AND auth.jwt()->>'role' IN ('manager', 'director')
  );

-- ===== CONTACTS TABLE =====
CREATE POLICY "contacts_org_access" ON contacts
  FOR SELECT
  USING (organization_id = auth.jwt()->>'org_id');

CREATE POLICY "contacts_insert_org" ON contacts
  FOR INSERT
  WITH CHECK (organization_id = auth.jwt()->>'org_id');

CREATE POLICY "contacts_update_org" ON contacts
  FOR UPDATE
  USING (organization_id = auth.jwt()->>'org_id')
  WITH CHECK (organization_id = auth.jwt()->>'org_id');

-- ===== DEALS TABLE =====
CREATE POLICY "deals_org_access" ON deals
  FOR SELECT
  USING (organization_id = auth.jwt()->>'org_id');

CREATE POLICY "deals_insert_org" ON deals
  FOR INSERT
  WITH CHECK (organization_id = auth.jwt()->>'org_id');

CREATE POLICY "deals_update_org" ON deals
  FOR UPDATE
  USING (organization_id = auth.jwt()->>'org_id')
  WITH CHECK (organization_id = auth.jwt()->>'org_id');

-- ===== CONVERSATIONS TABLE =====
CREATE POLICY "conversations_org_access" ON conversations
  FOR SELECT
  USING (organization_id = auth.jwt()->>'org_id');

CREATE POLICY "conversations_insert_system" ON conversations
  FOR INSERT
  WITH CHECK (organization_id = auth.jwt()->>'org_id');

-- ===== MESSAGES TABLE =====
CREATE POLICY "messages_org_access" ON messages
  FOR SELECT
  USING (organization_id = auth.jwt()->>'org_id');

CREATE POLICY "messages_insert_system" ON messages
  FOR INSERT
  WITH CHECK (organization_id = auth.jwt()->>'org_id');

-- ===== TASKS TABLE =====
CREATE POLICY "tasks_org_view" ON tasks
  FOR SELECT
  USING (organization_id = auth.jwt()->>'org_id');

CREATE POLICY "tasks_insert_org" ON tasks
  FOR INSERT
  WITH CHECK (organization_id = auth.jwt()->>'org_id');

CREATE POLICY "tasks_update_owner_or_manager" ON tasks
  FOR UPDATE
  USING (
    organization_id = auth.jwt()->>'org_id'
    AND (
      assigned_to_user_id = auth.uid()
      OR auth.jwt()->>'role' = 'manager'
    )
  )
  WITH CHECK (
    organization_id = auth.jwt()->>'org_id'
    AND (
      assigned_to_user_id = auth.uid()
      OR auth.jwt()->>'role' = 'manager'
    )
  );

-- ===== AGENT_ACTIONS TABLE =====
CREATE POLICY "agent_actions_org_view" ON agent_actions
  FOR SELECT
  USING (organization_id = auth.jwt()->>'org_id');

CREATE POLICY "agent_actions_insert_system" ON agent_actions
  FOR INSERT
  WITH CHECK (organization_id = auth.jwt()->>'org_id');

-- ===== CALLS TABLE =====
CREATE POLICY "calls_org_access" ON calls
  FOR SELECT
  USING (organization_id = auth.jwt()->>'org_id');

CREATE POLICY "calls_insert_org" ON calls
  FOR INSERT
  WITH CHECK (organization_id = auth.jwt()->>'org_id');

CREATE POLICY "calls_update_owner_or_manager" ON calls
  FOR UPDATE
  USING (
    organization_id = auth.jwt()->>'org_id'
    AND (
      recorded_by_user_id = auth.uid()
      OR auth.jwt()->>'role' = 'manager'
    )
  )
  WITH CHECK (
    organization_id = auth.jwt()->>'org_id'
    AND (
      recorded_by_user_id = auth.uid()
      OR auth.jwt()->>'role' = 'manager'
    )
  );
```

### Step 3: Test Policies

See **Section 6: Testing RLS** below.

---

## 6. Testing RLS Policies

### Local Testing with Supabase Studio

**Goal:** Verify that a user with a given JWT can/cannot access rows.

#### Test Case 1: Org Isolation (Rep A cannot see Org B data)

```sql
-- Run as ANON (no auth)
SELECT * FROM clients;
-- Expected: 0 rows (RLS denies)

-- Run as authenticated user (Org A, rep role)
-- In Supabase Studio: Auth -> Set Auth headers -> Paste JWT for Rep A
SELECT * FROM clients;
-- Expected: Rows for Org A only

-- Manually verify:
-- Before running query, set JWT in "Auth Preview" dropdown
-- Auth header used: Bearer <JWT token>
```

#### Test Case 2: Role-Based Approval (Rep cannot approve, Manager can)

```sql
-- As Rep (auth.jwt()->>'role' = 'rep')
UPDATE quotes SET approval_status = 'approved' WHERE id = '...';
-- Expected: UPDATE affects 0 rows (policy blocked)

-- As Manager (auth.jwt()->>'role' = 'manager')
UPDATE quotes SET approval_status = 'approved' WHERE id = '...';
-- Expected: UPDATE affects 1 row ✅
```

#### Test Case 3: Owner-Edit Only (Rep can edit own tasks, not others')

```sql
-- As Rep A (assigned_to_user_id = rep_a_uuid)
UPDATE tasks SET title = 'New title' WHERE assigned_to_user_id = rep_a_uuid;
-- Expected: UPDATE affects 1 row ✅

-- As Rep B (trying to edit Rep A's task)
UPDATE tasks SET title = 'Hacked!' WHERE assigned_to_user_id = rep_a_uuid;
-- Expected: UPDATE affects 0 rows (policy blocked)
```

### Testing via TypeScript (supabase-js client)

```typescript
import { createClient } from '@supabase/supabase-js';

const supabase = createClient(URL, ANON_KEY);

// Test Case: Authenticated user can see org clients
const { data: clients, error } = await supabase
  .from('clients')
  .select('*')
  .eq('organization_id', 'org-uuid');

console.assert(clients?.length > 0, 'Should see org clients');
console.assert(!error, 'Should have no auth error');

// Test Case: Cannot insert into other org
const { error: insertError } = await supabase
  .from('clients')
  .insert({
    organization_id: 'OTHER_ORG_UUID', // Wrong org
    name: 'Hack attempt',
  });

console.assert(insertError?.code === 'PGRST301', 'Should be policy violation');
```

### Automated Testing (CI/CD)

Add to `.github/workflows/test-rls.yml`:

```yaml
name: Test RLS Policies
on: [push]

jobs:
  rls-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: 18

      - name: Install dependencies
        run: pnpm install

      - name: Run RLS policy tests
        env:
          SUPABASE_URL: ${{ secrets.STAGING_SUPABASE_URL }}
          SUPABASE_ANON_KEY: ${{ secrets.STAGING_SUPABASE_ANON_KEY }}
        run: pnpm test:rls

      - name: Verify no auth errors
        run: pnpm test:auth
```

---

## 7. Development Checklist

When adding a new table or modifying existing schema:

- [ ] **Table has `organization_id` column?**
  ```sql
  ALTER TABLE new_table ADD COLUMN organization_id UUID NOT NULL REFERENCES organizations(id);
  ```

- [ ] **RLS is enabled?**
  ```sql
  ALTER TABLE new_table ENABLE ROW LEVEL SECURITY;
  ```

- [ ] **All SELECT/INSERT/UPDATE/DELETE policies are defined?**
  - Check [Section 4](#4-rls-policy-index-by-table) for your table pattern

- [ ] **Policies tested locally?**
  - Use Supabase Studio SQL editor
  - Test as different roles (rep, manager, director)
  - Test cross-org access (should be blocked)

- [ ] **`SUPABASE_SERVICE_ROLE_KEY` used only on backend?**
  - Never expose in frontend `.env`
  - Only used in `/app/api/*` routes and agent runtime

- [ ] **Supabase RLS docs consulted for complex policies?**
  - Reference: https://supabase.com/docs/guides/auth/row-level-security

---

## 8. Common Mistakes

| Mistake | Symptom | Fix |
|---|---|---|
| **Missing `organization_id` column** | "Column 'organization_id' does not exist" | Add column before enabling RLS |
| **RLS disabled** | Users see rows from ALL orgs | Run `ALTER TABLE X ENABLE ROW LEVEL SECURITY` |
| **JWT doesn't include `org_id` claim** | User sees 0 rows even when policy should allow | Update Supabase Auth JWT template or manually add claim |
| **Using ANON key in frontend against RLS table** | 401 Unauthorized or 0 rows | Use authenticated user client with JWT |
| **Policy uses `auth.uid()` but user_id column doesn't match** | User can't edit their own rows | Verify column name matches when writing policy |
| **No `FOR SELECT` policy** | Users can't query the table | Add `CREATE POLICY ... FOR SELECT USING (...)` |
| **Service role bypassed in frontend code** | Accidentally exposed RLS bypass | Remove service role key from frontend; only use in `/app/api` |

---

## 9. Monitoring & Auditing RLS

### View Active RLS Policies

```sql
SELECT
  tablename,
  policyname,
  cmd,
  qual,
  with_check
FROM pg_policies
WHERE schemaname = 'public'
ORDER BY tablename, policyname;
```

**Output example:**
```
tablename   | policyname                  | cmd    | qual                                        | with_check
------------|-----------------------------|-----------|--------------------------------------------|--------------------------------------------
clients     | clients_org_access          | SELECT | (organization_id = (auth.jwt() ->> 'org_id')) | NULL
clients     | clients_insert_org          | INSERT | NULL                                         | (organization_id = (auth.jwt() ->> 'org_id'))
deals       | deals_update_org            | UPDATE | (organization_id = (auth.jwt() ->> 'org_id')) | (organization_id = (auth.jwt() ->> 'org_id'))
```

### Audit Query: Check All Tables Have RLS

```sql
SELECT
  tablename,
  (SELECT count(*) FROM pg_policies WHERE pg_policies.tablename = pg_tables.tablename) as policy_count
FROM pg_tables
WHERE schemaname = 'public'
ORDER BY tablename;
```

**Expected:** Every table should have `policy_count > 0`

---

## 10. Next Steps

1. **Phase 1 (Now):** Apply foundation RLS policies (Section 5)
2. **Phase 2:** Add role-based approval policies for quotes, invoices, finanzas
3. **Phase 3:** Add director-only financial policies
4. **Ongoing:** Test policies in CI/CD on every schema change

---

## References

- [Supabase RLS Documentation](https://supabase.com/docs/guides/auth/row-level-security)
- [PostgreSQL Security — Policies](https://www.postgresql.org/docs/current/sql-createpolicy.html)
- [GDT PRD — Constraint #12](./PRD.md#5-non-negotiable-constraints)
- [MODULES.md — Phase 1 RLS Section](./MODULES.md#11---users-and-role-admin-3-days)
