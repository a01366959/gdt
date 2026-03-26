# GDT — Open Questions & Business Rules Clarification

**Purpose:** Living artifact capturing all business rule questions that block implementation. Updated as clarifications arrive. Every blocking question here prevents either UI decisions or database schema choices — so keep this visible.

**Version:** 1.0.0  
**Last Updated:** 2026-03-25  
**Status:** Active (pre-sprint-1)

---

## How to Use This Document

- **Each question** has: ID, title, impact (what blocks it), owner, due date, status
- **Status values:** `open` | `tbd_by_design` | `clarified` | `implemented`
- **Update frequency:** After every client sync, before sprint planning
- When a question is clarified, move it to the "Clarified" section and link to where the decision was documented in PRD/FLOWS/ARCHITECTURE

---

## Section 1: Blocking Questions (Active)

### Q1: Deal Classification → Quota Calculation

**Question:**  
When a rep closes an `asignada` deal (assigned by Miriam) or a `heredada` deal (inherited from another rep), what % of the deal value counts toward the rep's quota?

**Impact:**  
- Deal schema needs a `quota_weight` field (0.0 to 1.0)
- Quota calculation engine depends on this
- Reports dashboard must display this correctly (qualification score)
- Affects comp/bonus calculations (business critical)

**Options:**
- A) 100% credit to closing rep (asignada and heredada both count full)
- B) Tiered: `propia` = 100%, `asignada` = 50%, `heredada` = 25%
- C) Tiered by origin: `asignada` from Miriam = 100%, `heredada` from other rep = 50%
- D) Something else (specify)

**Owner:** Miriam / Omar (quotas & comp)  
**Due:** Before Sprint 1  
**Status:** `open`

---

### Q2: Preplan Margin Threshold → Approval Authority

**Question:**  
In Flow 1C-i (Preplan de Gastos), if gross margin % < 30%, who has authority to approve the preplan?

**Impact:**
- Approval routing in the Approvals queue
- Role system must support this boundary
- Workflow engine must route to correct approver
- Affects deal progression speed (approval bottleneck)

**Options:**
- A) Miriam only
- B) Miriam OR Dirección (Omar), depending on deal value threshold
- C) Tiered: < $50k = Miriam, $50k–$100k = Miriam + Omar, > $100k = Omar
- D) Something else (specify)

**Owner:** Miriam  
**Due:** Before Sprint 1  
**Status:** `open`

---

### Q3: Arrastre Calculation → Historical Data

**Question:**  
The transcript mentions `arrastre` (cumulative sales that offset low months). Is this formula already calculated in Zoho today, or does GDT calculate it manually?

**Impact:**
- Quota calculation engine implementation
- Data migration from Zoho vs. fresh calculation
- May affect rep incentive interpretation

**Details Needed:**
- Is arrastre = sum of all sales in prior 12 months? Or different?
- Is it recalculated monthly or annually?
- Any adjustments for client segments or service types?
- Current calculation location (Excel? Zoho? Manual report?)

**Owner:** Pablo (Zoho access) + Omar (quota methodology)  
**Due:** Before Sprint 1  
**Status:** `open`

---

### Q4: Facturella API Validation & Credentials

**Question:**  
Are Facturella API docs available and credentials configured? What are the exact endpoints needed?

**Impact:**
- Flow 3A (invoice generation) cannot be implemented without this
- Blocks test/staging environment setup
- Affects payment auxiliary sync

**Details Needed:**
- POST `/facturas` signature & response format
- POST `/facturas/{id}/cancel` behavior
- POST `/facturas/{id}/sustituir` (correction flow)
- Webhook format for SAT status changes
- Auth method (API key? OAuth?)
- Rate limits & sandbox availability

**Owner:** Pablo (technical validation)  
**Due:** April 2nd week  
**Status:** `open`

---

### Q5: Medicati Integration → Webhook Spec & Auth

**Question:**  
What is Medicati's exact webhook or polling interface for medical results? Need API docs and engineering contact.

**Impact:**
- Flow 2C (service results delivery) blocked
- Medical data handling (HIPAA-equivalent compliance)
- Approval gate for results delivery

**Details Needed:**
- Webhook vs. polling? Which does Medicati support?
- Webhook payload format (JSON structure)
- Auth method (signature verification? mutual TLS?)
- Result file format (PDF? encrypted? XML?)
- Retry/delivery guarantees

**Owner:** Miriam (provide Diego contact) → Diego (Medicati engineering)  
**Due:** TBD (Miriam to facilitate intro)  
**Status:** `open`

---

### Q6: Credit Terms → Shipping & Sync

**Question:**  
Are client-specific credit terms (30/60/90 days) stored in Zoho today? Can they be synced to GDT or do we treat them as event-based (per PO)?

**Impact:**
- Auxiliar de Pagos (payment auxiliary) pre-population
- Invoice due date calculation
- Cash flow forecasting

**Details Needed:**
- Does Zoho have a standard field for credit terms?
- Are they client-level (all orders from this cliente = 30 days) or PO-level (varies per order)?
- Any exceptions or escalation rules?

**Owner:** Pablo (Zoho access)  
**Due:** April 2nd week  
**Status:** `open`

---

### Q7: Role System & Admin Authority

**Question:**  
The current PRD mentions only `rep` and `manager` roles. The flows now introduce Dirección (Omar), Operaciones, Administración, and potentially others. What is the complete role hierarchy and what can each role do?

**Impact:**
- User management admin panel
- Approval routing (who can approve what)
- Dashboard visibility & filters (what each role sees)
- Database RLS policies (org-scoped + role-scoped)

**Roles Identified:**
- `rep` — sales representative (5 people)
- `manager` — Miriam, supervisory + approvals
- `director` — Omar, Dirección, strategic approvals
- `operaciones` — logistics & service scheduling
- `admin` — user management (Miriam? separate admin?)

**Details Needed:**
- Complete list of roles
- Approval authority per role (who can approve quotes, stage changes, etc.)
- Dashboard access per role (what dashboards each role sees)
- Can roles be custom-created in Admin panel or fixed set?

**Owner:** Miriam + Omar  
**Due:** Before Sprint 1  
**Status:** `open`

---

### Q8: Three-Horizon KPI Calculation

**Question:**  
The flows define three distinct pipeline horizons (Prospección / Forecast / Business Case). How does each horizon's value get calculated, and is Forecast really "90% probability" or another heuristic?

**Impact:**
- Pipeline dashboard data model
- Agent recommendations
- Executive reporting accuracy

**Details Needed:**
- How is "90% probability" determined? (Deal stage? Explicit rep confidence field?)
- Prospección value = all deals not yet in forecast?
- Business case value = all deals not yet in forecast but qualified?
- Any manual override by Miriam?

**Owner:** Miriam  
**Due:** Before Sprint 2  
**Status:** `open`

---

### Q9: Remanente Expiry Rule — Hard Cutoff or Configurable?

**Question:**  
Flow 4D defines "3-month expiry on open PO balance." Is this a hard rule across all clients or configurable per PO?

**Impact:**
- Purchase orders table schema (expiry_date logic)
- Agent reminder/alert schedule
- Revenue recognition questions (is expired PO revenue lost?)

**Owner:** Miriam / Omar  
**Due:** Before Sprint 2  
**Status:** `open`

---

## Section 2: Clarified Questions → Decisions Made

*(No entries yet. This section grows as clarifications arrive.)*

---

## Section 3: Notes on Business Rules Extracted from Transcript

**Items captured in FLOWS.md but still need validation:**

1. **Monthly contact minimum** — Every active client must be touched at least once/month (KPI)
   - Status: Documented in Flow 6D, needs validation from Miriam
   
2. **Mora escalation sequence** — Days overdue triggers different escalations (collect task at +1, Miriam alert at +15, Dirección at +30)
   - Status: Documented in Flow 4A, timeline needs confirmation
   
3. **Service scheduling** — Field team must confirm actual employee count before service date (may trigger repricing?)
   - Status: Documented in Flow 2A, repricing logic TBD
   
4. **Dormant reactivation** — 90-day no-activity threshold triggers automatic reactivation deal creation
   - Status: Documented in Flow 5, threshold confirmed in transcript
   
5. **Expense plan templates** — Can Miriam reuse expense plan templates for similar clients?
   - Status: Affects UX (preplan form), TBD

---

## How to Update This Document

**When a question gets clarified:**
1. Confirm the exact answer
2. Move the Q# entry to "Clarified Questions" section (below)
3. Update the corresponding section in FLOWS.md, PRD.md, or ARCHITECTURE.md with the decision
4. Link to the commit/PR where it was implemented
5. Update CHANGELOG.md

**When a new question surfaces during development:**
1. Add it here immediately (don't wait until a meeting)
2. Mark status as `open`, assign owner, set due date
3. Link to the Slack/email where it was raised
4. Add to sprint retrospective if it blocked work

---

## Blocked Work Until Clarified

| Question ID | Currently Blocks | Estimated Impact |
|---|---|---|
| Q1 | Quota calculation engine, reports schema | Must fix before rep dashboard |
| Q2 | Approval routing workflow, deal progression | Must fix before preplan form |
| Q3 | Quota engine calculation | Must fix before reporting agent |
| Q4 | Invoice generation, Flow 3A | Blocks Phase 2 |
| Q5 | Medical results delivery, Flow 2C | Blocks Phase 2 |
| Q6 | Payment auxiliary pre-population | Nice-to-have for Phase 1 |
| Q7 | Role system, approval routing, RLS | Must fix before Phase 1 |
| Q8 | Pipeline dashboard, agent logic | Must fix before Phase 1 |
| Q9 | PO remanente logic | Phase 2 |

