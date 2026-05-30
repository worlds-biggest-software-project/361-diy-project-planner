# Data Model Suggestion 2: Hybrid Relational + JSONB

> Project: DIY Project Planner · Created: 2026-05-25

## Philosophy

Core operational entities — users and projects — are relational tables with indexed columns for user-scoped queries and budget aggregation. Variable-structure data — phases, tasks, materials with supplier prices, expenses, contractor quotes, photos, permits, and sharing configurations — lives in JSONB columns with GIN indexes.

DIY project planners have a strongly project-centric access pattern: the dominant UI operation is "load a project dashboard showing phases, task progress, budget vs. actual, and upcoming deadlines." Embedding all project components into a single project row means the dashboard is a single-row read. The per-task and per-material detail is navigable within JSONB arrays.

The trade-off is that cross-project analytics (total spend across all projects, overdue tasks across projects) require JSONB extraction. But the most frequently queried fields (budget_cents, spent_cents, status) remain as top-level relational columns with indexes. For a consumer DIY app where each user typically has 1-5 active projects and the primary view is a project dashboard, the schema simplicity and offline-sync friendliness (fewer tables to sync) pay for themselves.

**Best for:** Teams building an MVP where rapid iteration on project features, minimal schema migrations, fast project dashboard loading, and efficient local-first sync are priorities.

**Trade-offs:**
- Pro: 4 tables — simple schema, fast to deploy, easy offline sync
- Pro: Project dashboard is a single-row read with full detail
- Pro: New task types, expense categories, and permit types require no migration
- Pro: Local-first sync requires syncing only 4 tables
- Con: Cross-project analytics require JSONB queries
- Con: Large projects with many tasks/materials can produce oversized JSONB
- Con: No FK enforcement on task dependencies within JSONB
- Con: Material price comparison operates within JSONB arrays

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ISO 19650 (BIM) | Project/phase/task structure in JSONB follows BIM hierarchy concepts |
| OpenAPI 3.1 | REST API documented in OpenAPI |
| JSON Schema 2020-12 | Project schema validation |
| OAuth 2.0 / OIDC | User auth |
| RFC 8252 / RFC 7636 | OAuth for native apps with PKCE |
| WCAG 2.2 | Accessibility compliance |
| GDPR / CCPA | Privacy compliance |
| OWASP MASVS 2.0 | Mobile security baseline |
| CRDT / SQLite | Local-first sync (4 tables simplifies sync) |
| MCP | AI assistant integration |

---

## Users

```sql
CREATE TABLE users (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email               TEXT UNIQUE NOT NULL,
    display_name        TEXT NOT NULL,
    auth_provider       TEXT NOT NULL CHECK (auth_provider IN (
                            'email_password','google','apple'
                        )),
    timezone            TEXT NOT NULL DEFAULT 'America/New_York',
    locale              TEXT NOT NULL DEFAULT 'en-US',
    currency            TEXT NOT NULL DEFAULT 'USD',
    zip_code            TEXT,
    contractors_json    JSONB NOT NULL DEFAULT '[]',
    -- [{
    --   "id": "uuid", "name": "Mike's Electric",
    --   "company": "Mike's Electric LLC", "trade": "electrical",
    --   "phone": "+1-555-0123", "email": "mike@mikeselectric.com",
    --   "license_number": "EC-12345", "is_insured": true,
    --   "rating": 4, "review_notes": "Good work, showed up on time",
    --   "is_preferred": true
    -- }]
    settings_json       JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "notifications_enabled": true,
    --   "deadline_reminder_days": 3,
    --   "quote_expiry_reminder_days": 7,
    --   "auto_lowest_price": true
    -- }
    mcp_json            JSONB,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Projects

```sql
CREATE TABLE projects (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id             UUID NOT NULL REFERENCES users(id),
    name                TEXT NOT NULL,
    description         TEXT,
    project_type        TEXT CHECK (project_type IN (
                            'kitchen','bathroom','bedroom','living_room',
                            'basement','attic','garage','outdoor',
                            'whole_house','addition','landscape',
                            'electrical','plumbing','hvac','roofing',
                            'flooring','painting','custom'
                        )),
    status              TEXT NOT NULL CHECK (status IN (
                            'planning','in_progress','on_hold',
                            'completed','cancelled'
                        )) DEFAULT 'planning',
    address             TEXT,
    zip_code            TEXT,
    budget_cents        BIGINT NOT NULL DEFAULT 0,
    spent_cents         BIGINT NOT NULL DEFAULT 0,
    start_date          DATE,
    target_end_date     DATE,
    is_archived         BOOLEAN NOT NULL DEFAULT FALSE,
    ai_generated        BOOLEAN NOT NULL DEFAULT FALSE,
    phases_json         JSONB NOT NULL DEFAULT '[]',
    -- [{
    --   "id": "uuid", "name": "Demolition", "sort_order": 1,
    --   "status": "completed", "budget_cents": 200000,
    --   "start_date": "2026-05-01", "end_date": "2026-05-05",
    --   "tasks": [{
    --     "id": "uuid", "name": "Remove existing cabinets",
    --     "type": "demolition", "status": "completed",
    --     "priority": "high", "assignee": "Self",
    --     "estimated_hours": 4, "actual_hours": 5,
    --     "due_date": "2026-05-02", "completed_at": "2026-05-02T16:00:00Z",
    --     "depends_on": null, "sort_order": 1
    --   }, {
    --     "id": "uuid", "name": "Remove old flooring",
    --     "type": "demolition", "status": "completed",
    --     "depends_on": "uuid-previous-task", "sort_order": 2
    --   }]
    -- }, {
    --   "id": "uuid", "name": "Rough-In", "sort_order": 2,
    --   "status": "in_progress", "budget_cents": 500000,
    --   "tasks": [...]
    -- }]
    materials_json      JSONB NOT NULL DEFAULT '[]',
    -- [{
    --   "id": "uuid", "name": "Cabinet set - Shaker White",
    --   "category": "fixtures", "phase_id": "uuid",
    --   "quantity": 1, "unit": "set",
    --   "estimated_unit_cost_cents": 350000,
    --   "actual_unit_cost_cents": 329900,
    --   "is_purchased": true, "purchased_at": "2026-05-10",
    --   "barcode": "012345678901",
    --   "prices": [
    --     {"supplier": "Home Depot", "unit_price_cents": 349900,
    --      "product_url": "https://...", "is_lowest": false},
    --     {"supplier": "Lowe's", "unit_price_cents": 329900,
    --      "product_url": "https://...", "is_lowest": true},
    --     {"supplier": "IKEA", "unit_price_cents": 289900,
    --      "product_url": "https://...", "is_lowest": false,
    --      "is_available": false}
    --   ],
    --   "notes": "Measured for 10ft run"
    -- }]
    expenses_json       JSONB NOT NULL DEFAULT '[]',
    -- [{
    --   "id": "uuid", "type": "material", "description": "Cabinet set",
    --   "amount_cents": 329900, "vendor": "Lowe's",
    --   "receipt_url": "s3://...", "material_id": "uuid",
    --   "paid_at": "2026-05-10", "payment_method": "credit_card"
    -- }, {
    --   "id": "uuid", "type": "labour", "description": "Electrician - rough-in",
    --   "amount_cents": 150000, "vendor": "Mike's Electric",
    --   "paid_at": "2026-05-15"
    -- }]
    quotes_json         JSONB NOT NULL DEFAULT '[]',
    -- [{
    --   "id": "uuid", "contractor_id": "uuid",
    --   "contractor_name": "Mike's Electric",
    --   "trade": "electrical", "phase_id": "uuid",
    --   "amount_cents": 250000,
    --   "scope": "Full kitchen electrical: 6 outlets, 2 dedicated circuits, under-cabinet lighting",
    --   "quote_date": "2026-04-20", "valid_until": "2026-05-20",
    --   "status": "accepted", "document_url": "s3://..."
    -- }]
    photos_json         JSONB NOT NULL DEFAULT '[]',
    -- [{
    --   "id": "uuid", "url": "s3://...", "thumbnail_url": "s3://...",
    --   "type": "before", "phase_id": null, "task_id": null,
    --   "caption": "Kitchen before demolition",
    --   "taken_at": "2026-04-28T10:00:00Z"
    -- }]
    permits_json        JSONB NOT NULL DEFAULT '[]',
    -- [{
    --   "id": "uuid", "type": "electrical",
    --   "jurisdiction": "San Francisco DBI",
    --   "status": "approved", "permit_number": "EP-2026-1234",
    --   "application_date": "2026-04-15",
    --   "approval_date": "2026-04-25",
    --   "expiry_date": "2027-04-25",
    --   "fee_cents": 35000, "document_url": "s3://..."
    -- }]
    shares_json         JSONB NOT NULL DEFAULT '[]',
    -- [{
    --   "id": "uuid", "email": "spouse@example.com",
    --   "name": "Jane Doe", "role": "editor",
    --   "access_token": "...", "is_active": true
    -- }]
    notes               TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_projects_user ON projects (user_id);
CREATE INDEX idx_projects_status ON projects (status) WHERE is_archived = FALSE;
CREATE INDEX idx_projects_phases ON projects USING GIN (phases_json);
CREATE INDEX idx_projects_materials ON projects USING GIN (materials_json);
CREATE INDEX idx_projects_expenses ON projects USING GIN (expenses_json);
```

---

## AI Suggestions

```sql
CREATE TABLE ai_suggestions (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id             UUID NOT NULL REFERENCES users(id),
    project_id          UUID REFERENCES projects(id),
    suggestion_type     TEXT NOT NULL CHECK (suggestion_type IN (
                            'project_scope','task_list','material_list',
                            'budget_estimate','material_alternative',
                            'permit_checklist','budget_alert',
                            'query_response'
                        )),
    title               TEXT NOT NULL,
    body                TEXT NOT NULL,
    suggestion_data     JSONB,
    -- {
    --   "generated_phases": [...],
    --   "generated_materials": [...],
    --   "estimated_budget_cents": 1500000,
    --   "permit_types_needed": ["building","electrical"],
    --   "llm_model": "claude-sonnet-4-6",
    --   "tokens_used": 1200
    -- }
    is_applied          BOOLEAN NOT NULL DEFAULT FALSE,
    is_dismissed        BOOLEAN NOT NULL DEFAULT FALSE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_suggestions_user ON ai_suggestions (user_id, created_at DESC);
CREATE INDEX idx_suggestions_project ON ai_suggestions (project_id)
    WHERE project_id IS NOT NULL;
```

---

## Audit Log

```sql
CREATE TABLE audit_log (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id             UUID NOT NULL REFERENCES users(id),
    actor_type          TEXT NOT NULL CHECK (actor_type IN (
                            'user','system','ai','contractor'
                        )),
    action              TEXT NOT NULL,
    entity_type         TEXT NOT NULL,
    entity_id           UUID NOT NULL,
    changes_json        JSONB,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

CREATE INDEX idx_audit_user ON audit_log (user_id, created_at);
CREATE INDEX idx_audit_entity ON audit_log (entity_type, entity_id);
```

---

## Example Queries

### Project dashboard — single row read

```sql
SELECT name, project_type, status, budget_cents / 100.0 AS budget,
       spent_cents / 100.0 AS spent,
       phases_json, materials_json, expenses_json,
       quotes_json, photos_json, permits_json
FROM projects
WHERE id = 'project-uuid' AND user_id = 'user-uuid';
```

### Budget vs. actual by expense type

```sql
SELECT e->>'type' AS expense_type,
       COUNT(*) AS entries,
       SUM((e->>'amount_cents')::BIGINT) / 100.0 AS total
FROM projects p,
     jsonb_array_elements(p.expenses_json) AS e
WHERE p.id = 'project-uuid'
GROUP BY e->>'type'
ORDER BY total DESC;
```

### Overdue tasks across all projects

```sql
SELECT p.name AS project_name,
       ph->>'name' AS phase_name,
       t->>'name' AS task_name,
       (t->>'due_date')::DATE AS due_date,
       t->>'priority' AS priority
FROM projects p,
     jsonb_array_elements(p.phases_json) AS ph,
     jsonb_array_elements(ph->'tasks') AS t
WHERE p.user_id = 'user-uuid'
  AND p.is_archived = FALSE
  AND t->>'status' NOT IN ('completed','skipped')
  AND (t->>'due_date')::DATE < CURRENT_DATE
ORDER BY (t->>'due_date')::DATE ASC;
```

### Lowest-price supplier per material

```sql
SELECT m->>'name' AS material,
       (m->>'quantity')::NUMERIC AS quantity,
       m->>'unit' AS unit,
       pr->>'supplier' AS supplier,
       (pr->>'unit_price_cents')::BIGINT / 100.0 AS unit_price
FROM projects p,
     jsonb_array_elements(p.materials_json) AS m,
     jsonb_array_elements(m->'prices') AS pr
WHERE p.id = 'project-uuid'
  AND (pr->>'is_lowest')::BOOLEAN = TRUE
ORDER BY m->>'name';
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Users | 1 | users (embeds contractors, settings, MCP) |
| Projects | 1 | projects (embeds phases, tasks, materials, expenses, quotes, photos, permits, shares) |
| AI | 1 | ai_suggestions |
| Audit | 1 | audit_log (partitioned) |
| **Total** | **4** | |

---

## Key Design Decisions

1. **`phases_json` with nested `tasks` on projects** — a DIY project typically has 3-8 phases with 5-20 tasks each. Embedding phases with their tasks in the project row means the project dashboard and task list are a single-row read. Task dependencies reference sibling task IDs within the same JSONB array.

2. **`materials_json` with embedded `prices` array** — each material item carries an array of supplier prices, enabling in-row price comparison without a separate prices table. The `is_lowest` flag pre-computes the cheapest option.

3. **`expenses_json` on projects** — expense entries are project-scoped transactions. Embedding them keeps budget vs. actual calculations within a single row read, though cross-project spend analysis requires JSONB extraction.

4. **Top-level `budget_cents` and `spent_cents` on projects** — these are the most frequently queried financial fields (dashboard, progress bars) and benefit from being indexed relational columns rather than computed from JSONB.

5. **`contractors_json` on users** — a user's contractor directory (typically 5-20 contractors) is embedded on the user row. Project quotes reference contractor IDs from this array.

6. **`quotes_json` on projects** — contractor quotes are project-specific with status tracking and expiry dates. Embedding on the project row enables side-by-side comparison within a single row read.

7. **`permits_json` on projects** — permit requirements vary by project type and jurisdiction. Embedding as a JSONB array accommodates variable permit types without a separate table.

8. **`ai_suggestions` as a separate table** — AI-generated project scopes, material lists, and budget estimates are persisted separately so users can review, apply, or dismiss them independently of the project data.

9. **4 tables for offline sync** — local-first operation benefits from fewer tables to sync between device and cloud. With only 4 tables, CRDT-based sync is simpler and more reliable than syncing 14 tables.

10. **4 tables** — DIY projects have a strongly project-centric data model; embedding all project components (phases, tasks, materials, expenses, quotes, photos, permits) into a single project row minimises joins and simplifies offline-first sync.
