# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: DIY Project Planner · Created: 2026-05-25

## Philosophy

Every project planning concept — projects, phases, tasks, materials, expenses, contractor quotes, photos, permits — gets its own dedicated table with typed columns, foreign keys, and purpose-built indexes. This approach mirrors how professional construction management platforms (Procore, Buildertrend, OpenProject) structure their data: projects contain phases, phases contain tasks, materials have a bill-of-materials with supplier pricing, and expenses are tracked against budget line items.

The dominant query pattern for a DIY planner is "show me a project dashboard with phase progress, budget vs. actual, and upcoming tasks" — which means a project-centric query joining phases, tasks, materials, and expenses filtered by project_id. Normalisation means each entity can be independently indexed and queried: overdue tasks across all projects, total material spend by supplier, contractor quotes by trade for comparison.

The trade-off is schema rigidity: adding new project attributes or expense categories requires migration. But for a domain with well-established data types (tasks with dependencies, bill-of-materials line items, expense ledger entries), the schema is stable, and the query performance and referential integrity benefits outweigh migration cost.

**Best for:** Teams building a production-grade DIY planner where data integrity, budget vs. actual accuracy, multi-supplier price comparison, and professional export formats are priorities.

**Trade-offs:**
- Pro: Full referential integrity across projects, tasks, materials, and expenses
- Pro: Budget vs. actual queries use indexed SUM aggregation on expense columns
- Pro: Multi-supplier price comparison uses indexed queries on material_prices
- Pro: Task dependencies modelled explicitly with FK relationships
- Pro: Maps cleanly to OpenProject's work package data model and IFC concepts
- Con: 14 tables — more complex schema
- Con: Adding new expense categories or task attributes requires migration
- Con: Project dashboard requires JOINs across multiple tables
- Con: Custom fields need a JSONB escape hatch

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ISO 19650 (BIM) | Project/phase/task hierarchy follows BIM information management concepts |
| ISO 23387 (BIM data templates) | Material item attributes follow construction object data template patterns |
| IFC (ISO 16739-1) | Room and element naming compatible with IFC data model |
| OpenAPI 3.1 | REST API documented in OpenAPI |
| JSON Schema 2020-12 | Request/response and export format validation |
| OAuth 2.0 / OIDC | User auth |
| RFC 8252 / RFC 7636 | OAuth 2.0 for native apps with PKCE |
| WCAG 2.2 / ISO 40500 | Accessibility compliance for web and mobile |
| GDPR / CCPA | Privacy compliance for home address and financial data |
| OWASP MASVS 2.0 | Mobile security baseline |
| CRDT / SQLite | Local-first sync for offline operation |
| MCP | AI assistant integration for project queries |

---

## Users

```sql
CREATE TABLE users (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email               TEXT UNIQUE NOT NULL,
    display_name        TEXT NOT NULL,
    avatar_url          TEXT,
    auth_provider       TEXT NOT NULL CHECK (auth_provider IN (
                            'email_password','google','apple'
                        )),
    timezone            TEXT NOT NULL DEFAULT 'America/New_York',
    locale              TEXT NOT NULL DEFAULT 'en-US',
    currency            TEXT NOT NULL DEFAULT 'USD',
    zip_code            TEXT,
    mcp_enabled         BOOLEAN NOT NULL DEFAULT FALSE,
    is_active           BOOLEAN NOT NULL DEFAULT TRUE,
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
    actual_end_date     DATE,
    is_archived         BOOLEAN NOT NULL DEFAULT FALSE,
    ai_generated        BOOLEAN NOT NULL DEFAULT FALSE,
    notes               TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_projects_user ON projects (user_id);
CREATE INDEX idx_projects_status ON projects (status) WHERE is_archived = FALSE;
```

---

## Phases

```sql
CREATE TABLE phases (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id          UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    name                TEXT NOT NULL,
    description         TEXT,
    sort_order          INTEGER NOT NULL DEFAULT 0,
    status              TEXT NOT NULL CHECK (status IN (
                            'not_started','in_progress','completed'
                        )) DEFAULT 'not_started',
    budget_cents        BIGINT NOT NULL DEFAULT 0,
    spent_cents         BIGINT NOT NULL DEFAULT 0,
    start_date          DATE,
    end_date            DATE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_phases_project ON phases (project_id, sort_order);
```

---

## Tasks

```sql
CREATE TABLE tasks (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    phase_id            UUID NOT NULL REFERENCES phases(id) ON DELETE CASCADE,
    project_id          UUID NOT NULL REFERENCES projects(id),
    name                TEXT NOT NULL,
    description         TEXT,
    task_type           TEXT NOT NULL CHECK (task_type IN (
                            'labour','material_purchase','inspection',
                            'permit','design','demolition','cleanup',
                            'custom'
                        )) DEFAULT 'labour',
    status              TEXT NOT NULL CHECK (status IN (
                            'not_started','in_progress','completed',
                            'blocked','skipped'
                        )) DEFAULT 'not_started',
    priority            TEXT CHECK (priority IN ('low','medium','high','urgent')),
    assignee            TEXT,
    estimated_hours     NUMERIC(6,1),
    actual_hours        NUMERIC(6,1),
    due_date            DATE,
    completed_at        TIMESTAMPTZ,
    depends_on_task_id  UUID REFERENCES tasks(id),
    sort_order          INTEGER NOT NULL DEFAULT 0,
    notes               TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_tasks_phase ON tasks (phase_id, sort_order);
CREATE INDEX idx_tasks_project ON tasks (project_id);
CREATE INDEX idx_tasks_due ON tasks (due_date)
    WHERE status NOT IN ('completed','skipped');
CREATE INDEX idx_tasks_dependency ON tasks (depends_on_task_id)
    WHERE depends_on_task_id IS NOT NULL;
```

---

## Materials

```sql
CREATE TABLE materials (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id          UUID NOT NULL REFERENCES projects(id),
    phase_id            UUID REFERENCES phases(id),
    name                TEXT NOT NULL,
    description         TEXT,
    category            TEXT CHECK (category IN (
                            'lumber','drywall','electrical','plumbing',
                            'flooring','tile','paint','hardware',
                            'fixtures','appliance','insulation',
                            'roofing','landscaping','concrete','adhesive',
                            'fastener','tool_rental','other'
                        )),
    quantity            NUMERIC(10,2) NOT NULL,
    unit                TEXT NOT NULL DEFAULT 'each',
    estimated_unit_cost_cents BIGINT,
    actual_unit_cost_cents BIGINT,
    total_cost_cents    BIGINT,
    supplier            TEXT,
    is_purchased        BOOLEAN NOT NULL DEFAULT FALSE,
    purchased_at        DATE,
    barcode             TEXT,
    product_url         TEXT,
    notes               TEXT,
    sort_order          INTEGER NOT NULL DEFAULT 0,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_materials_project ON materials (project_id);
CREATE INDEX idx_materials_phase ON materials (phase_id);
CREATE INDEX idx_materials_category ON materials (category);
```

---

## Material Prices (Supplier Comparison)

```sql
CREATE TABLE material_prices (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    material_id         UUID NOT NULL REFERENCES materials(id) ON DELETE CASCADE,
    supplier            TEXT NOT NULL,
    unit_price_cents    BIGINT NOT NULL,
    is_available        BOOLEAN NOT NULL DEFAULT TRUE,
    product_url         TEXT,
    last_checked_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    is_lowest           BOOLEAN NOT NULL DEFAULT FALSE,
    notes               TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_mat_prices_material ON material_prices (material_id);
CREATE INDEX idx_mat_prices_lowest ON material_prices (material_id)
    WHERE is_lowest = TRUE;
```

---

## Expenses

```sql
CREATE TABLE expenses (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id          UUID NOT NULL REFERENCES projects(id),
    phase_id            UUID REFERENCES phases(id),
    expense_type        TEXT NOT NULL CHECK (expense_type IN (
                            'material','labour','permit','tool_rental',
                            'tool_purchase','delivery','dumpster',
                            'inspection','design','other'
                        )),
    description         TEXT NOT NULL,
    amount_cents        BIGINT NOT NULL,
    vendor              TEXT,
    receipt_url         TEXT,
    material_id         UUID REFERENCES materials(id),
    paid_at             DATE NOT NULL DEFAULT CURRENT_DATE,
    payment_method      TEXT CHECK (payment_method IN (
                            'credit_card','debit','cash','check',
                            'transfer','financing'
                        )),
    notes               TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_expenses_project ON expenses (project_id);
CREATE INDEX idx_expenses_phase ON expenses (phase_id);
CREATE INDEX idx_expenses_type ON expenses (expense_type);
CREATE INDEX idx_expenses_date ON expenses (paid_at);
```

---

## Contractors

```sql
CREATE TABLE contractors (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id             UUID NOT NULL REFERENCES users(id),
    name                TEXT NOT NULL,
    company             TEXT,
    trade               TEXT NOT NULL CHECK (trade IN (
                            'general','electrical','plumbing','hvac',
                            'roofing','flooring','painting','carpentry',
                            'masonry','landscaping','demolition',
                            'drywall','tile','cabinet','countertop',
                            'window_door','insulation','other'
                        )),
    phone               TEXT,
    email               TEXT,
    license_number      TEXT,
    is_insured          BOOLEAN,
    rating              INTEGER CHECK (rating BETWEEN 1 AND 5),
    review_notes        TEXT,
    is_preferred        BOOLEAN NOT NULL DEFAULT FALSE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_contractors_user ON contractors (user_id);
CREATE INDEX idx_contractors_trade ON contractors (trade);
```

---

## Contractor Quotes

```sql
CREATE TABLE contractor_quotes (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id          UUID NOT NULL REFERENCES projects(id),
    contractor_id       UUID NOT NULL REFERENCES contractors(id),
    phase_id            UUID REFERENCES phases(id),
    amount_cents        BIGINT NOT NULL,
    scope_description   TEXT,
    quote_date          DATE NOT NULL DEFAULT CURRENT_DATE,
    valid_until         DATE,
    status              TEXT NOT NULL CHECK (status IN (
                            'received','under_review','accepted',
                            'rejected','expired'
                        )) DEFAULT 'received',
    document_url        TEXT,
    notes               TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_quotes_project ON contractor_quotes (project_id);
CREATE INDEX idx_quotes_contractor ON contractor_quotes (contractor_id);
CREATE INDEX idx_quotes_expiry ON contractor_quotes (valid_until)
    WHERE status = 'received' OR status = 'under_review';
```

---

## Photos

```sql
CREATE TABLE photos (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id          UUID NOT NULL REFERENCES projects(id),
    phase_id            UUID REFERENCES phases(id),
    task_id             UUID REFERENCES tasks(id),
    photo_url           TEXT NOT NULL,
    thumbnail_url       TEXT,
    photo_type          TEXT NOT NULL CHECK (photo_type IN (
                            'before','during','after','issue',
                            'receipt','permit','measurement','other'
                        )),
    caption             TEXT,
    taken_at            TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_photos_project ON photos (project_id);
CREATE INDEX idx_photos_task ON photos (task_id) WHERE task_id IS NOT NULL;
```

---

## Permits

```sql
CREATE TABLE permits (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id          UUID NOT NULL REFERENCES projects(id),
    permit_type         TEXT NOT NULL CHECK (permit_type IN (
                            'building','electrical','plumbing','mechanical',
                            'demolition','grading','zoning','hoa_approval',
                            'other'
                        )),
    jurisdiction        TEXT,
    status              TEXT NOT NULL CHECK (status IN (
                            'not_needed','to_apply','pending',
                            'approved','denied','expired'
                        )) DEFAULT 'to_apply',
    application_date    DATE,
    approval_date       DATE,
    expiry_date         DATE,
    permit_number       TEXT,
    fee_cents           BIGINT,
    document_url        TEXT,
    notes               TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_permits_project ON permits (project_id);
CREATE INDEX idx_permits_status ON permits (status)
    WHERE status IN ('to_apply','pending');
```

---

## Project Shares

```sql
CREATE TABLE project_shares (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id          UUID NOT NULL REFERENCES projects(id),
    user_id             UUID NOT NULL REFERENCES users(id),
    shared_with_email   TEXT NOT NULL,
    shared_with_name    TEXT,
    role                TEXT NOT NULL CHECK (role IN (
                            'viewer','editor','contractor'
                        )),
    access_token        TEXT UNIQUE NOT NULL,
    is_active           BOOLEAN NOT NULL DEFAULT TRUE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_shares_project ON project_shares (project_id);
CREATE INDEX idx_shares_token ON project_shares (access_token) WHERE is_active = TRUE;
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

### Project budget vs. actual dashboard

```sql
SELECT p.name AS project_name, p.budget_cents / 100.0 AS budget,
       COALESCE(SUM(e.amount_cents), 0) / 100.0 AS spent,
       (p.budget_cents - COALESCE(SUM(e.amount_cents), 0)) / 100.0 AS remaining,
       CASE WHEN p.budget_cents > 0
            THEN ROUND(COALESCE(SUM(e.amount_cents), 0) * 100.0 / p.budget_cents, 1)
            ELSE 0 END AS pct_spent
FROM projects p
LEFT JOIN expenses e ON e.project_id = p.id
WHERE p.id = 'project-uuid'
GROUP BY p.id;
```

### Budget breakdown by expense type

```sql
SELECT e.expense_type,
       COUNT(*) AS entries,
       SUM(e.amount_cents) / 100.0 AS total
FROM expenses e
WHERE e.project_id = 'project-uuid'
GROUP BY e.expense_type
ORDER BY total DESC;
```

### Contractor quote comparison per trade

```sql
SELECT c.name AS contractor, c.company, c.trade,
       c.rating, c.is_insured,
       cq.amount_cents / 100.0 AS quote_amount,
       cq.quote_date, cq.valid_until, cq.status
FROM contractor_quotes cq
JOIN contractors c ON c.id = cq.contractor_id
WHERE cq.project_id = 'project-uuid'
ORDER BY c.trade, cq.amount_cents ASC;
```

### Lowest-price supplier per material

```sql
SELECT m.name AS material, m.quantity, m.unit,
       mp.supplier, mp.unit_price_cents / 100.0 AS unit_price,
       (m.quantity * mp.unit_price_cents) / 100.0 AS line_total,
       mp.product_url
FROM materials m
JOIN material_prices mp ON mp.material_id = m.id AND mp.is_lowest = TRUE
WHERE m.project_id = 'project-uuid'
ORDER BY m.sort_order;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Users | 1 | users |
| Projects | 3 | projects, phases, tasks |
| Materials | 2 | materials, material_prices |
| Financial | 1 | expenses |
| Contractors | 2 | contractors, contractor_quotes |
| Documentation | 2 | photos, permits |
| Sharing | 1 | project_shares |
| Audit | 1 | audit_log (partitioned) |
| **Total** | **14** | |

---

## Key Design Decisions

1. **Project → phase → task hierarchy** — mirrors how renovation projects are naturally structured (e.g., Kitchen Reno → Demolition → Remove cabinets, Remove flooring). Tasks have dependencies for sequencing multi-trade work.

2. **`materials` with separate `material_prices` table** — the bill-of-materials lists items with quantities, while the prices table enables per-supplier comparison. The `is_lowest` flag on material_prices pre-computes the cheapest option.

3. **`expenses` as a separate ledger** — every expense is a dated, categorised transaction linked to a project and optionally to a phase or material. This enables budget vs. actual variance tracking at project, phase, and expense-type granularity.

4. **`contractors` as a user-scoped directory** — contractors are reusable across projects. Each contractor has a trade, license info, insurance status, and user rating, enabling the contractor evaluation workflow.

5. **`contractor_quotes` linked to both project and contractor** — quotes are project-specific but reference the contractor directory. The `valid_until` date with status tracking enables quote expiry reminders.

6. **`permits` as a separate table** — permit requirements vary by jurisdiction and project type. Separate permit records with status tracking and expiry dates enable jurisdiction-aware compliance checklists.

7. **`photos` with typed categories** — before/during/after photos, issue documentation, receipts, and measurements are categorised and linked to projects, phases, or tasks for structured documentation.

8. **BIGINT cents for all monetary values** — budget_cents, spent_cents, amount_cents, unit_price_cents follow the Stripe convention to avoid floating-point precision issues.

9. **`depends_on_task_id` for task dependencies** — a simple parent reference enables sequencing constraints (e.g., plumbing rough-in must complete before drywall). More complex dependency graphs would need a junction table.

10. **14 tables** — the normalised schema separates every project management concern into its own table, enabling independent queries for dashboards, budget reports, material lists, contractor comparisons, and compliance checklists.
