# Data Model Suggestion 3: Event-Sourced / Audit-First

> Project: DIY Project Planner · Created: 2026-05-25

## Philosophy

Every project action — creating a phase, completing a task, logging an expense, receiving a contractor quote, uploading a photo, applying for a permit — is captured as an immutable event in a single append-only event store. The current state of the project (dashboard, budget, task progress) is derived by replaying or projecting events into purpose-built read models (CQRS pattern). The event store is the source of truth; read models are disposable and rebuildable.

Home improvement projects benefit from event sourcing because cost overruns are the number one consumer complaint about renovations. An event-sourced architecture makes every financial decision traceable: when was the budget set, when did each expense occur, when was a contractor quote accepted, and how did the running total change over time? This enables "budget timeline" views showing exactly when and where the project went over budget — a feature no surveyed competitor provides.

Similarly, permit compliance requires audit trails (when was the permit applied for, when approved, when did the inspection pass), and task dependency tracking benefits from explicit event ordering. The AI scope generator can emit events that the user reviews and accepts, creating a clear boundary between AI-generated and user-confirmed project data.

The trade-off is query complexity: the project dashboard requires a materialised read model. But for a DIY planner where budget defensibility, contractor accountability, and permit compliance are core requirements, event sourcing provides capabilities that snapshot-based models cannot replicate.

**Best for:** Teams building a DIY planner where full cost audit trail, budget timeline visualisation, permit compliance tracking, and clear AI-generated vs. user-confirmed data boundaries are priorities.

**Trade-offs:**
- Pro: Complete cost audit trail — every expense and budget change preserved
- Pro: Budget timeline shows exactly when and where overruns occurred
- Pro: Permit compliance has a full audit trail
- Pro: AI-generated vs. user-confirmed data clearly separated as events
- Pro: Read models can be rebuilt when new export formats or analytics are added
- Con: Project dashboard requires materialised read models
- Con: Event replay for complex projects can be slow without snapshots
- Con: Higher storage costs — events are never deleted
- Con: More complex application code

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| CloudEvents 1.0 | Event envelope format (ce_source, ce_type, ce_specversion, ce_time) |
| ISO 19650 (BIM) | Project event types follow BIM information management lifecycle |
| OpenAPI 3.1 | REST API for commands and read model queries |
| JSON Schema 2020-12 | Event data and export format validation |
| OAuth 2.0 / OIDC | User auth |
| RFC 8252 / RFC 7636 | OAuth for native apps with PKCE |
| WCAG 2.2 | Accessibility compliance |
| GDPR / CCPA | Crypto-shredding for erasure compliance |
| OWASP MASVS 2.0 | Mobile security baseline |
| MCP | AI assistant integration via event-derived read models |

---

## Event Store (Infrastructure)

```sql
CREATE TABLE event_store (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_type         TEXT NOT NULL CHECK (stream_type IN (
                            'user','project','task','material',
                            'expense','contractor','permit',
                            'photo','ai','config'
                        )),
    stream_id           UUID NOT NULL,
    sequence_num        BIGINT NOT NULL,
    event_type          TEXT NOT NULL,
    event_data          JSONB NOT NULL,
    metadata            JSONB NOT NULL DEFAULT '{}',
    ce_source           TEXT NOT NULL DEFAULT '/diy-project-planner',
    ce_specversion      TEXT NOT NULL DEFAULT '1.0',
    ce_type             TEXT NOT NULL,
    ce_time             TIMESTAMPTZ NOT NULL DEFAULT now(),
    actor_id            UUID,
    actor_type          TEXT NOT NULL CHECK (actor_type IN (
                            'user','system','ai','contractor'
                        )),
    encryption_key_ref  TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (stream_id, sequence_num)
) PARTITION BY RANGE (created_at);

CREATE INDEX idx_events_stream ON event_store (stream_id, sequence_num);
CREATE INDEX idx_events_type ON event_store (event_type, created_at);
CREATE INDEX idx_events_actor ON event_store (actor_id, created_at);
CREATE INDEX idx_events_ce_type ON event_store (ce_type, ce_time);
```

---

## Stream Snapshots (Infrastructure)

```sql
CREATE TABLE stream_snapshots (
    stream_id           UUID NOT NULL,
    sequence_num        BIGINT NOT NULL,
    snapshot_data       JSONB NOT NULL,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (stream_id, sequence_num)
);
```

---

## Projection Checkpoints (Infrastructure)

```sql
CREATE TABLE projection_checkpoints (
    projection_name     TEXT PRIMARY KEY,
    last_event_id       UUID NOT NULL,
    last_sequence_num   BIGINT NOT NULL,
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Event Types by Stream

### User Stream
- `user_registered` — profile, auth provider, zip code
- `user_settings_changed` — notification prefs, currency
- `user_deactivated`

### Project Stream
- `project_created` — name, type, address, zip_code, budget_cents, dates
- `project_updated` — changed fields
- `project_status_changed` — planning → in_progress → completed
- `phase_added` — name, sort_order, budget_cents
- `phase_updated` — changed fields
- `phase_completed` — all tasks done
- `phase_removed`
- `budget_adjusted` — new budget_cents with reason
- `project_shared` — email, role, access_token
- `share_revoked`
- `project_archived`

### Task Stream
- `task_created` — name, type, phase_id, priority, estimated_hours, due_date
- `task_updated` — changed fields
- `task_started` — status → in_progress
- `task_completed` — actual_hours, completed_at
- `task_blocked` — reason, blocking_task_id
- `task_unblocked`
- `task_skipped` — reason
- `dependency_added` — depends_on_task_id
- `dependency_removed`

### Material Stream
- `material_added` — name, category, phase_id, quantity, unit, estimated_cost
- `material_updated` — changed fields
- `supplier_price_added` — supplier, unit_price, product_url, availability
- `supplier_price_updated` — changed price or availability
- `material_purchased` — actual_cost, supplier, receipt_url
- `material_removed`

### Expense Stream
- `expense_logged` — type, description, amount_cents, vendor, receipt_url, payment_method
- `expense_updated` — corrected amount or category
- `expense_deleted` — removed from budget tracking
- `receipt_uploaded` — OCR-scanned receipt with extracted data

### Contractor Stream
- `contractor_added` — name, company, trade, contact info, license
- `contractor_updated` — changed fields
- `contractor_rated` — rating, review_notes
- `quote_received` — amount, scope, valid_until, document_url
- `quote_accepted` — quote awarded
- `quote_rejected` — reason
- `quote_expired` — valid_until passed

### Permit Stream
- `permit_identified` — type, jurisdiction (AI or manual)
- `permit_applied` — application_date, fee_cents
- `permit_approved` — approval_date, permit_number, expiry_date
- `permit_denied` — reason
- `inspection_scheduled` — date, type
- `inspection_passed`
- `inspection_failed` — corrections needed
- `permit_expired`

### Photo Stream
- `photo_uploaded` — url, type (before/during/after/issue), caption, phase_id, task_id
- `photo_removed`

### AI Stream
- `scope_generated` — AI-created phases, tasks, materials, budget estimate
- `scope_accepted` — user accepted AI scope (triggers project events)
- `scope_modified` — user modified AI scope before accepting
- `scope_rejected`
- `material_alternative_suggested` — cheaper material recommendation
- `budget_alert_generated` — overrun detection
- `permit_checklist_generated` — jurisdiction-specific permit requirements
- `query_asked` — natural-language project query
- `query_answered` — AI response

---

## Read Model: Project Dashboard

```sql
CREATE TABLE rm_project_dashboard (
    user_id             UUID NOT NULL,
    project_id          UUID NOT NULL,
    name                TEXT NOT NULL,
    project_type        TEXT,
    status              TEXT NOT NULL,
    budget_cents        BIGINT NOT NULL DEFAULT 0,
    spent_cents         BIGINT NOT NULL DEFAULT 0,
    remaining_cents     BIGINT NOT NULL DEFAULT 0,
    budget_pct_used     NUMERIC(5,1) NOT NULL DEFAULT 0,
    phases_json         JSONB NOT NULL DEFAULT '[]',
    -- [{
    --   "id": "uuid", "name": "Demolition", "status": "completed",
    --   "budget_cents": 200000, "spent_cents": 185000,
    --   "tasks_total": 5, "tasks_completed": 5
    -- }]
    upcoming_tasks_json JSONB NOT NULL DEFAULT '[]',
    recent_expenses_json JSONB NOT NULL DEFAULT '[]',
    budget_timeline_json JSONB NOT NULL DEFAULT '[]',
    -- [{
    --   "date": "2026-05-01", "event": "expense_logged",
    --   "description": "Dumpster rental", "amount_cents": 45000,
    --   "running_total_cents": 45000, "budget_cents": 1500000
    -- }, {
    --   "date": "2026-05-10", "event": "expense_logged",
    --   "description": "Cabinet set", "amount_cents": 329900,
    --   "running_total_cents": 374900, "budget_cents": 1500000
    -- }]
    overdue_count       INTEGER NOT NULL DEFAULT 0,
    photos_count        INTEGER NOT NULL DEFAULT 0,
    start_date          DATE,
    target_end_date     DATE,
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (user_id, project_id)
);
```

---

## Read Model: Material List

```sql
CREATE TABLE rm_material_list (
    user_id             UUID NOT NULL,
    project_id          UUID NOT NULL,
    materials_json      JSONB NOT NULL DEFAULT '[]',
    -- [{
    --   "id": "uuid", "name": "Cabinet set", "category": "fixtures",
    --   "quantity": 1, "unit": "set",
    --   "estimated_cost_cents": 350000,
    --   "actual_cost_cents": 329900, "is_purchased": true,
    --   "lowest_price": {"supplier": "Lowe's", "cents": 329900},
    --   "prices": [
    --     {"supplier": "Home Depot", "cents": 349900},
    --     {"supplier": "Lowe's", "cents": 329900}
    --   ]
    -- }]
    total_estimated_cents BIGINT NOT NULL DEFAULT 0,
    total_actual_cents  BIGINT NOT NULL DEFAULT 0,
    items_purchased     INTEGER NOT NULL DEFAULT 0,
    items_pending       INTEGER NOT NULL DEFAULT 0,
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (user_id, project_id)
);
```

---

## Read Model: Contractor Comparison

```sql
CREATE TABLE rm_contractor_comparison (
    user_id             UUID NOT NULL,
    project_id          UUID NOT NULL,
    quotes_json         JSONB NOT NULL DEFAULT '[]',
    -- [{
    --   "trade": "electrical",
    --   "quotes": [{
    --     "contractor_name": "Mike's Electric",
    --     "amount_cents": 250000, "status": "accepted",
    --     "rating": 4, "is_insured": true,
    --     "scope": "Full kitchen electrical",
    --     "valid_until": "2026-05-20"
    --   }, {
    --     "contractor_name": "City Electricians",
    --     "amount_cents": 310000, "status": "rejected",
    --     "rating": 3, "is_insured": true
    --   }]
    -- }]
    total_quoted_cents  BIGINT NOT NULL DEFAULT 0,
    total_awarded_cents BIGINT NOT NULL DEFAULT 0,
    expiring_quotes     INTEGER NOT NULL DEFAULT 0,
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (user_id, project_id)
);
```

---

## Read Model: Permit Tracker

```sql
CREATE TABLE rm_permit_tracker (
    user_id             UUID NOT NULL,
    project_id          UUID NOT NULL,
    permits_json        JSONB NOT NULL DEFAULT '[]',
    -- [{
    --   "id": "uuid", "type": "electrical",
    --   "jurisdiction": "San Francisco DBI",
    --   "status": "approved", "permit_number": "EP-2026-1234",
    --   "application_date": "2026-04-15",
    --   "approval_date": "2026-04-25",
    --   "expiry_date": "2027-04-25",
    --   "fee_cents": 35000,
    --   "inspections": [
    --     {"type": "rough_in", "date": "2026-05-20", "result": "passed"},
    --     {"type": "final", "date": null, "result": null}
    --   ]
    -- }]
    total_fees_cents    BIGINT NOT NULL DEFAULT 0,
    permits_pending     INTEGER NOT NULL DEFAULT 0,
    permits_approved    INTEGER NOT NULL DEFAULT 0,
    inspections_pending INTEGER NOT NULL DEFAULT 0,
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (user_id, project_id)
);
```

---

## Read Model: Cost Dashboard

```sql
CREATE TABLE rm_cost_dashboard (
    user_id             UUID NOT NULL,
    total_projects      INTEGER NOT NULL DEFAULT 0,
    total_budget_cents  BIGINT NOT NULL DEFAULT 0,
    total_spent_cents   BIGINT NOT NULL DEFAULT 0,
    by_project_json     JSONB NOT NULL DEFAULT '[]',
    -- [{
    --   "project_id": "uuid", "name": "Kitchen Reno",
    --   "budget_cents": 1500000, "spent_cents": 874900,
    --   "pct_used": 58.3, "status": "in_progress"
    -- }]
    by_category_json    JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "material": 524900, "labour": 250000,
    --   "permit": 35000, "tool_rental": 15000,
    --   "other": 50000
    -- }
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (user_id)
);
```

---

## Example Event Sequences

### AI-generated project scope → user acceptance

```
1. scope_generated      {stream: ai, actor: ai}
   → input: "renovate my kitchen, 10x12, budget $15k"
   → generated: 4 phases, 18 tasks, 12 materials, budget $14,200

2. scope_modified       {stream: ai, actor: user}
   → user removed 2 tasks, adjusted cabinet budget upward

3. scope_accepted       {stream: ai, actor: user}
   → triggers: project_created, 4x phase_added, 16x task_created, 12x material_added
   → rm_project_dashboard created
   → rm_material_list created
```

### Expense logging with budget overrun detection

```
1. expense_logged       {stream: expense, actor: user}
   → type: material, amount: $3,299, vendor: Lowe's
   → rm_project_dashboard.spent_cents updated
   → rm_project_dashboard.budget_timeline_json appended

2. budget_alert_generated {stream: ai, actor: ai}
   → "Material spending is 15% over phase budget.
   →  Cabinet set cost $3,299 vs estimated $3,500 (on track),
   →  but tile + grout combined is $1,200 over estimate."
   → notification sent to user
```

### Contractor quote lifecycle

```
1. quote_received       {stream: contractor, actor: user}
   → Mike's Electric, $2,500, valid until 2026-05-20
   → rm_contractor_comparison updated

2. quote_received       {stream: contractor, actor: user}
   → City Electricians, $3,100, valid until 2026-05-25
   → rm_contractor_comparison updated (side-by-side)

3. quote_accepted       {stream: contractor, actor: user}
   → Mike's Electric awarded
   → rm_contractor_comparison.total_awarded_cents updated

4. quote_rejected       {stream: contractor, actor: user}
   → City Electricians rejected
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Infrastructure | 3 | event_store (partitioned), stream_snapshots, projection_checkpoints |
| Read Models | 5 | rm_project_dashboard, rm_material_list, rm_contractor_comparison, rm_permit_tracker, rm_cost_dashboard |
| **Total** | **8** | |

---

## Key Design Decisions

1. **`budget_timeline_json` on the project dashboard** — the unique advantage of event sourcing for a DIY planner is the budget timeline: a chronological record of every financial event showing how the running total changed over time. This enables "where did the money go?" visualisations that no surveyed competitor provides.

2. **AI scope generation as events** — when AI generates a project scope, the generated data is an event. When the user modifies or accepts it, those are separate events. This creates a clear boundary between AI-suggested and user-confirmed data, with full audit trail.

3. **Separate expense stream** — expenses are the highest-frequency financial events and benefit from their own stream. Budget vs. actual calculations are derived by projecting expense events against budget events.

4. **Contractor stream with quote lifecycle** — quotes move through received → accepted/rejected/expired states as events. The `rm_contractor_comparison` read model enables side-by-side comparison per trade.

5. **Permit stream with inspection events** — the permit lifecycle (identified → applied → approved → inspected) is modelled as events, creating a compliance audit trail. The `rm_permit_tracker` read model shows current permit status and upcoming inspections.

6. **`rm_cost_dashboard` per user** — a single-row read model aggregating spend across all projects by category, enabling cross-project financial analysis without replaying events.

7. **Task dependencies as events** — `dependency_added` and `dependency_removed` events on the task stream enable flexible dependency management. The project dashboard read model pre-computes blocked tasks.

8. **CloudEvents envelope** — every event carries standard CloudEvents fields for interoperability. Project events can be published to MCP-connected AI assistants for natural-language project queries.

9. **Photo and document events** — photo uploads are events on the photo stream, creating a chronological visual record of the project. Before/during/after photos are timestamped events that can be replayed as a project timelapse.

10. **8 tables (3 infrastructure + 5 read models)** — the event-sourced architecture separates the write path from the read path, with read models tailored to each workflow: dashboard (project overview), materials (bill of materials with prices), contractors (quote comparison), permits (compliance tracker), and costs (cross-project spend).
