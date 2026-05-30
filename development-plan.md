# DIY Project Planner — Phased Development Plan

> Project: 361-diy-project-planner · Created: 2026-05-30
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

This plan synthesises `research.md`, `features.md`, `standards.md`, `README.md`, and the three data-model suggestions. The canonical database schema follows **data-model-suggestion-1 (Entity-Centric Normalized Relational)** — the production-grade choice that gives full referential integrity for budget-vs-actual accuracy, multi-supplier price comparison, and clean OpenProject/IFC alignment.

---

## Product Summary

DIY Project Planner is an **AI-native, offline-first** home improvement planner for homeowners and DIY enthusiasts running multi-trade renovations. It unifies phased task management, a bill-of-materials builder with supplier price comparison, budget-vs-actual tracking, contractor quote comparison, photo documentation, and jurisdiction-aware permit checklists in one consumer-friendly tool. It replaces the spreadsheet/notes/browser-tab patchwork that homeowners use today.

**Primary personas:** (1) the DIY homeowner planning and executing a renovation themselves; (2) the homeowner-as-general-contractor coordinating hired trades; (3) the co-owner/spouse collaborator; (4) an invited contractor with scoped view access.

**Key differentiators:** offline-first local storage with CRDT-style background sync, supplier price comparison across retailers, jurisdiction-aware permit checklists, and AI-assisted scope generation that turns "renovate my kitchen" into a structured phase/task/material plan with a budget range.

**Deployment model:** Hybrid. A **client-server** architecture: native-feeling **mobile-first PWA / cross-platform client** (offline-capable, local SQLite) that syncs to a **self-hostable backend** (REST API + Postgres). The backend is also packaged as a Docker image for self-hosting and exposes an MCP server for AI assistant access.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Backend language | **Python 3.12** | The product is AI-heavy (scope generation, material estimation, anomaly detection, MCP). Python has the strongest LLM SDK ecosystem (`anthropic`, `openai`) and excellent data-validation tooling. |
| API framework | **FastAPI** | Native OpenAPI 3.1 generation (required by `standards.md`), Pydantic v2 validation that maps directly to JSON Schema 2020-12, async support for LLM and retailer-API calls, first-class dependency injection for auth. |
| Server database | **PostgreSQL 16** | The chosen normalized model (14 tables, FK integrity, indexed SUM aggregation, JSONB audit log, partitioned `audit_log`) needs a real RDBMS. Postgres supports `gen_random_uuid()`, `JSONB`/GIN, partitioning, and is the standard sync target for local-first SQLite. |
| Client database | **SQLite** | De-facto local-first store (`standards.md`). Enables full offline read/write on-site. Mirrors the server schema with sync metadata columns. |
| Sync strategy | **Last-writer-wins + per-row version vectors, hybrid logical clocks** | Pragmatic CRDT-lite. Each row carries `updated_at_hlc` and `device_id`; field-level merge for independent fields, LWW for conflicts. Avoids a full CRDT library while meeting the offline merge requirement. |
| Client framework | **React + TypeScript + Vite PWA** | Mobile-first responsive web companion that installs as a PWA, runs offline via service worker + IndexedDB/SQLite-WASM, and reuses one codebase for web. Native iOS/Android wrappers (Capacitor) are a backlog item. |
| Local client storage | **SQLite-WASM (OPFS) on web; Capacitor SQLite on native** | Same SQL schema across platforms; OPFS gives durable offline storage in the browser. |
| Auth | **OAuth 2.0 + OIDC, PKCE (AppAuth pattern)** | `standards.md` mandates RFC 8252 + RFC 7636 for native/public clients. Supports Google/Apple social login and email/password. Backend issues short-lived JWT access tokens + rotating refresh tokens. |
| Task queue | **Arq (Redis-backed async queue)** | Async workloads: LLM scope generation, retailer price refresh, push-notification scheduling, quote-expiry reminders. Lightweight, asyncio-native, pairs with FastAPI. |
| Cache / broker | **Redis 7** | Backs Arq, caches retailer price lookups, holds rate-limit and reminder schedules. |
| LLM provider | **Anthropic Claude (via `anthropic` SDK), provider-abstracted** | AI scope generation, material estimation, anomaly detection. Abstracted behind an `LLMClient` interface so providers can be swapped. Prompt caching enabled. |
| Object storage | **S3-compatible (MinIO for self-host, AWS S3 for SaaS)** | Photos, receipts, permit documents, PDF exports. Pre-signed upload URLs. |
| Retailer data | **Traject BigBox API (Home Depot + Lowe's unified)** | `standards.md` identifies this as the most practical live-pricing path; abstracted behind a `RetailerPriceProvider` interface with a manual-entry fallback. |
| PDF/CSV export | **WeasyPrint (PDF), Python `csv`** | HTML-to-PDF for material lists and cost summaries; deterministic CSV. |
| MCP server | **`mcp` Python SDK** | Exposes read project state / generate tasks / update materials tools to AI assistants per the emerging MCP standard. |
| Push notifications | **Web Push (VAPID) + FCM/APNs bridge** | Task deadlines, quote expiry, contractor follow-ups. Web Push for PWA; FCM/APNs for native wrappers later. |
| Testing (backend) | **pytest + pytest-asyncio + httpx.AsyncClient + testcontainers** | Unit + integration; testcontainers spins real Postgres/Redis for integration; respx mocks HTTP. |
| Testing (client) | **Vitest + React Testing Library + Playwright** | Unit/component + E2E including an offline-mode E2E. |
| Lint/format/type (backend) | **Ruff + Mypy (strict)** | Fast lint+format, static typing. |
| Lint/format/type (client) | **ESLint + Prettier + tsc** | Standard TS toolchain. |
| Migrations | **Alembic** | Versioned Postgres migrations for the 14-table schema. |
| Packaging / deploy | **Docker + docker-compose** | Self-host story: api + worker + postgres + redis + minio in one compose file. |
| Package managers | **uv (Python), pnpm (client)** | Fast, reproducible installs. |
| Accessibility | **WCAG 2.2 / ISO 40500 + WCAG2Mobile** | Consumer non-technical audience used on-site; enforced via axe-core in Playwright. |

### Project Structure

```
diy-project-planner/
├── README.md
├── docker-compose.yml
├── .env.example
├── backend/
│   ├── pyproject.toml
│   ├── uv.lock
│   ├── Dockerfile
│   ├── alembic.ini
│   ├── migrations/                  # Alembic versions
│   ├── src/diyplanner/
│   │   ├── __init__.py
│   │   ├── main.py                  # FastAPI app factory, router wiring
│   │   ├── config.py                # Pydantic Settings (env vars)
│   │   ├── db/
│   │   │   ├── engine.py            # async SQLAlchemy engine/session
│   │   │   ├── models.py            # ORM models (14 tables)
│   │   │   └── base.py
│   │   ├── schemas/                 # Pydantic request/response models
│   │   │   ├── project.py
│   │   │   ├── phase.py
│   │   │   ├── task.py
│   │   │   ├── material.py
│   │   │   ├── expense.py
│   │   │   ├── contractor.py
│   │   │   ├── photo.py
│   │   │   ├── permit.py
│   │   │   ├── share.py
│   │   │   └── sync.py
│   │   ├── api/                     # FastAPI routers
│   │   │   ├── deps.py              # auth deps, current_user, db session
│   │   │   ├── auth.py
│   │   │   ├── projects.py
│   │   │   ├── phases.py
│   │   │   ├── tasks.py
│   │   │   ├── materials.py
│   │   │   ├── expenses.py
│   │   │   ├── contractors.py
│   │   │   ├── photos.py
│   │   │   ├── permits.py
│   │   │   ├── shares.py
│   │   │   ├── dashboard.py
│   │   │   ├── ai.py
│   │   │   ├── exports.py
│   │   │   └── sync.py
│   │   ├── services/                # business logic
│   │   │   ├── budget.py
│   │   │   ├── pricing.py
│   │   │   ├── permits.py
│   │   │   ├── notifications.py
│   │   │   ├── audit.py
│   │   │   └── sync.py
│   │   ├── integrations/
│   │   │   ├── llm/
│   │   │   │   ├── client.py        # LLMClient interface + Anthropic impl
│   │   │   │   ├── prompts.py       # prompt templates
│   │   │   │   └── scope.py         # scope generation
│   │   │   ├── retailers/
│   │   │   │   ├── base.py          # RetailerPriceProvider interface
│   │   │   │   ├── bigbox.py        # Traject BigBox impl
│   │   │   │   └── manual.py        # manual-entry fallback
│   │   │   └── storage/
│   │   │       └── s3.py            # pre-signed URL helpers
│   │   ├── workers/
│   │   │   ├── arq_app.py           # Arq worker settings
│   │   │   ├── pricing_jobs.py
│   │   │   ├── reminder_jobs.py
│   │   │   └── ai_jobs.py
│   │   ├── mcp/
│   │   │   └── server.py            # MCP tool server
│   │   ├── auth/
│   │   │   ├── oauth.py             # OIDC/PKCE flows
│   │   │   ├── jwt.py               # token issue/verify
│   │   │   └── password.py          # argon2 hashing
│   │   └── permitrules/
│   │       └── rules.yml            # jurisdiction → required permits map
│   └── tests/
│       ├── conftest.py
│       ├── unit/
│       ├── integration/
│       └── fixtures/
├── client/
│   ├── package.json
│   ├── vite.config.ts
│   ├── src/
│   │   ├── main.tsx
│   │   ├── app/                     # routes
│   │   ├── components/
│   │   ├── db/                      # SQLite-WASM schema + queries
│   │   ├── sync/                    # sync engine (push/pull, merge)
│   │   ├── api/                     # generated OpenAPI client
│   │   ├── store/                   # state (Zustand)
│   │   └── sw/                      # service worker
│   └── tests/
│       ├── unit/
│       └── e2e/                     # Playwright
└── docs/
    └── openapi.json                 # generated, committed for client codegen
```

---

## Phase 1: Foundation & Schema

### Purpose
Establish the backend skeleton, configuration, the complete 14-table Postgres schema via Alembic, the ORM models, and the local dev environment (docker-compose with Postgres + Redis + MinIO). After this phase the database exists, migrations run cleanly, the FastAPI app boots with a health check, and CI runs lint/type/test. Nothing user-facing yet, but every later phase builds on this without restructuring.

### Tasks

#### 1.1 — Project scaffold, config, and tooling

**What**: Create the backend package, dependency manifest, settings, Dockerfile, docker-compose, and CI.

**Design**:
- `pyproject.toml` with deps: `fastapi`, `uvicorn[standard]`, `sqlalchemy[asyncio]`, `asyncpg`, `alembic`, `pydantic`, `pydantic-settings`, `argon2-cffi`, `pyjwt`, `arq`, `redis`, `anthropic`, `httpx`, `boto3`, `weasyprint`, `mcp`. Dev: `pytest`, `pytest-asyncio`, `respx`, `testcontainers[postgres,redis]`, `ruff`, `mypy`.
- `config.py` Pydantic `Settings`:

```python
class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env", env_prefix="DIY_")
    database_url: str
    redis_url: str = "redis://localhost:6379/0"
    jwt_secret: str
    jwt_access_ttl_seconds: int = 900
    jwt_refresh_ttl_seconds: int = 60 * 60 * 24 * 30
    s3_endpoint: str
    s3_bucket: str = "diyplanner"
    s3_access_key: str
    s3_secret_key: str
    anthropic_api_key: str | None = None
    bigbox_api_key: str | None = None
    default_currency: str = "USD"
    cors_origins: list[str] = ["http://localhost:5173"]
```

- `main.py`: `create_app()` factory wiring routers, CORS, exception handlers, and a `GET /healthz` returning `{"status":"ok","db":<bool>,"redis":<bool>}`.
- `docker-compose.yml`: services `api`, `worker`, `postgres:16`, `redis:7`, `minio`.
- CI (GitHub Actions): `ruff check`, `ruff format --check`, `mypy`, `pytest`.

**Testing**:
- `Unit: Settings loads from env with DIY_ prefix → correct typed values, defaults applied`
- `Unit: Settings missing required DIY_DATABASE_URL → ValidationError naming database_url`
- `Integration (real, testcontainers): GET /healthz with live Postgres+Redis → 200, db=true, redis=true`
- `Integration: GET /healthz with Redis down → 200, redis=false (degraded, not 500)`

#### 1.2 — Database schema & migrations

**What**: Implement the full normalized schema (data-model-suggestion-1) as SQLAlchemy models and the initial Alembic migration, plus sync-support columns.

**Design**:
- 14 tables exactly as specified in `data-model-suggestion-1.md`: `users`, `projects`, `phases`, `tasks`, `materials`, `material_prices`, `expenses`, `contractors`, `contractor_quotes`, `photos`, `permits`, `project_shares`, `audit_log` (range-partitioned by `created_at`), plus all CHECK constraints and indexes listed there.
- **Sync extension**: every user-mutable table gains `updated_at_hlc TEXT NOT NULL` (hybrid logical clock string), `origin_device_id UUID`, and `deleted_at TIMESTAMPTZ` (soft delete / tombstone for sync). Add `auth_credentials` table for email/password (argon2 hash, separate from `users` for security): `user_id`, `password_hash`, `updated_at`.
- All monetary columns are `BIGINT` cents (Stripe convention, per the data model).
- Alembic env configured for async engine; initial migration creates all tables, enums-as-CHECK, indexes, and the first monthly `audit_log` partition.

**Testing**:
- `Integration (testcontainers): alembic upgrade head → all 14 tables + auth_credentials exist; downgrade base → clean`
- `Integration: insert project referencing non-existent user_id → FK violation`
- `Integration: insert task with status='bogus' → CHECK violation`
- `Integration: insert two material_prices for one material, set is_lowest on cheaper → lowest-price query returns the cheaper row`
- `Unit: HLC generator produces monotonically increasing strings across rapid calls`

#### 1.3 — Audit log service

**What**: A reusable `record_audit(...)` service that writes immutable change events.

**Design**:

```python
async def record_audit(
    session, *, user_id: UUID, actor_type: Literal["user","system","ai","contractor"],
    action: str, entity_type: str, entity_id: UUID, changes: dict | None = None,
) -> None: ...
```
Inserts into `audit_log` with `changes_json`. Called by service-layer mutations (project/expense/quote/permit changes especially, per the event-sourcing rationale in suggestion-3, captured here as an audit trail without full CQRS).

**Testing**:
- `Unit (mocked session): record_audit builds row with correct entity_type/action/changes_json`
- `Integration: record three expense edits → audit query by entity returns 3 ordered rows`

---

## Phase 2: Authentication & Users

### Purpose
Implement secure authentication (OAuth 2.0 + OIDC + PKCE for public clients, plus email/password) and user account management. This gates every other API and establishes `current_user` dependency injection used everywhere downstream. Standards: RFC 8252, RFC 7636, OIDC Core, OWASP MASVS-L1.

### Tasks

#### 2.1 — Email/password & JWT issuance

**What**: Registration, login, refresh, and token verification.

**Design**:
- `auth/password.py`: argon2id hash/verify.
- `auth/jwt.py`: `issue_access(user_id) -> str`, `issue_refresh(user_id) -> str` (rotating, stored hashed), `verify(token) -> Claims`. HS256 with `jwt_secret`; access TTL 15 min, refresh 30 days, refresh rotation on use.
- Endpoints:
  - `POST /auth/register` `{email, password, display_name}` → `201 {user, access, refresh}`
  - `POST /auth/login` `{email, password}` → `200 {access, refresh}`
  - `POST /auth/refresh` `{refresh}` → `200 {access, refresh}` (old refresh invalidated)
  - `POST /auth/logout` → `204`
- `api/deps.py`: `get_current_user` extracts/validates Bearer access token → `User`.

**Testing**:
- `Unit: argon2 verify true for correct password, false otherwise`
- `Integration: register then login → 200 with valid JWT whose sub == user id`
- `Integration: login wrong password → 401, generic message (no user enumeration)`
- `Integration: refresh with rotated/old token → 401`
- `Integration: protected route without Bearer → 401`

#### 2.2 — OAuth 2.0 / OIDC + PKCE (Google, Apple)

**What**: Authorization-code-with-PKCE flow for social login.

**Design**:
- `auth/oauth.py`: `GET /auth/oauth/{provider}/start` returns provider authorize URL + `code_challenge` (S256), state stored in Redis (TTL 10 min). `GET /auth/oauth/{provider}/callback?code&state` exchanges code (+`code_verifier`) for ID token, validates JWT signature against provider JWKS, upserts `users` row keyed on verified email, issues app JWTs.
- No embedded webviews (RFC 8252): clients use the system browser / ASWebAuthenticationSession; backend only handles start+callback.

**Testing**:
- `Integration (respx-mocked provider): start → returns authorize URL containing code_challenge & S256`
- `Integration (mocked JWKS): callback with valid code → new user upserted, app tokens returned`
- `Integration: callback with mismatched/expired state → 400, no user created`
- `Integration: callback with invalid ID token signature → 401`

#### 2.3 — User profile & privacy controls (GDPR/CCPA)

**What**: Profile read/update, data export, and account deletion.

**Design**:
- `GET /me`, `PATCH /me` (display_name, timezone, locale, currency, zip_code, mcp_enabled).
- `GET /me/export` → async job assembles a JSON+media bundle of all user data (right of access).
- `DELETE /me` → soft-anonymise then hard-delete after grace window (right to erasure); cascades to owned projects.

**Testing**:
- `Integration: PATCH /me currency=EUR → persisted, returned`
- `Integration: DELETE /me → user inactive, projects no longer accessible`
- `Integration: GET /me/export → bundle contains user's projects and excludes other users' data`

---

## Phase 3: Core Project Domain (Projects, Phases, Tasks)

### Purpose
Deliver the heart of the planner: create projects, organise them into phases, and break phases into tasks with dependencies, status, due dates, and ordering. This is the project-management spine every other feature attaches to. After this phase a user can fully plan a renovation's structure via the API.

### Tasks

#### 3.1 — Projects CRUD

**What**: Full lifecycle for `projects`, scoped to the owner.

**Design**:
- Pydantic `ProjectCreate`/`ProjectUpdate`/`ProjectOut` mapping the `projects` table (name, description, project_type enum, status enum, address, zip_code, budget_cents, start/target/actual dates, is_archived, ai_generated, notes).
- Endpoints (all under `get_current_user`, owner-scoped):
  - `POST /projects` → 201
  - `GET /projects?status=&archived=` → list
  - `GET /projects/{id}` → 200/404
  - `PATCH /projects/{id}`, `DELETE /projects/{id}` (soft delete → tombstone)
- Every mutation calls `record_audit`.

**Testing**:
- `Unit: ProjectCreate rejects project_type='spaceship' → ValidationError`
- `Integration: create project → 201, owner=current_user`
- `Integration: GET another user's project → 404 (not 403, avoid existence leak)`
- `Integration: PATCH status planning→in_progress → audit row written`

#### 3.2 — Phases CRUD & ordering

**What**: Phases nested under a project with `sort_order` and status.

**Design**:
- `POST /projects/{pid}/phases`, `GET /projects/{pid}/phases` (ordered by sort_order), `PATCH /phases/{id}`, `DELETE /phases/{id}` (cascade tasks).
- `POST /projects/{pid}/phases/reorder` `{phase_ids: [...]}` → atomic re-sequence.

**Testing**:
- `Integration: create 3 phases, reorder → GET returns new order`
- `Integration: delete phase → its tasks cascade-deleted`
- `Integration: add phase to project you don't own → 404`

#### 3.3 — Tasks CRUD, dependencies & status

**What**: Tasks under phases, with `depends_on_task_id`, status state machine, ordering, hours, due dates.

**Design**:
- Status machine: `not_started → in_progress → (completed | blocked | skipped)`; setting `completed` stamps `completed_at`.
- Endpoints: `POST /phases/{id}/tasks`, `GET /projects/{pid}/tasks?status=&overdue=`, `PATCH /tasks/{id}`, `DELETE`, `POST /phases/{id}/tasks/reorder`.
- Dependency rule: cannot mark a task `completed` while its `depends_on_task_id` task is not completed → `409 DEPENDENCY_NOT_MET`. Cycle prevention: reject setting a dependency that would create a cycle (walk chain).

**Testing**:
- `Unit: completing task with incomplete dependency → 409`
- `Unit: setting depends_on that creates A→B→A cycle → 422`
- `Integration: PATCH task status=completed → completed_at set`
- `Integration: GET tasks?overdue=true → only past-due, non-terminal tasks`

---

## Phase 4: Materials, Pricing & Budget Tracking

### Purpose
Add the financial and procurement core: the bill-of-materials builder, multi-supplier price comparison with auto-lowest-cost highlighting, the expense ledger, and budget-vs-actual roll-ups at project, phase, and category granularity. This is the second pillar of the product's value proposition.

### Tasks

#### 4.1 — Materials (bill-of-materials) CRUD

**What**: Materials under a project/phase with quantity, unit, costs, supplier, purchase status.

**Design**:
- Maps the `materials` table (category enum, quantity NUMERIC, estimated/actual unit cost cents, total_cost_cents computed = quantity × actual-or-estimated unit cost, is_purchased, barcode, product_url).
- Endpoints: `POST /projects/{pid}/materials`, `GET /projects/{pid}/materials?phase_id=&category=&purchased=`, `PATCH`, `DELETE`, `POST .../reorder`.
- On write, recompute `total_cost_cents` in the service layer.

**Testing**:
- `Unit: total_cost_cents = round(quantity * unit_cost) given qty=2.5, unit=400 → 1000`
- `Integration: create material → appears in project list`
- `Integration: filter category=paint → only paint items`

#### 4.2 — Material prices & lowest-cost comparison

**What**: Multiple supplier prices per material with `is_lowest` maintenance.

**Design**:
- `POST /materials/{id}/prices`, `GET /materials/{id}/prices`, `PATCH`, `DELETE`.
- `services/pricing.py: recompute_lowest(material_id)` sets exactly one available price's `is_lowest=true` (cheapest available); runs after every price mutation.
- `GET /projects/{pid}/materials/lowest` → bill of materials joined to lowest available price with line totals (the suggestion-1 example query).

**Testing**:
- `Unit: recompute_lowest with prices [500,300(unavail),400] → 400 flagged lowest`
- `Unit: all prices unavailable → no row flagged lowest`
- `Integration: add cheaper price → is_lowest moves to it`
- `Integration: lowest-cost report returns expected line totals`

#### 4.3 — Expense ledger

**What**: Dated, categorised expense transactions linked to project/phase/material.

**Design**:
- Maps `expenses` (expense_type enum, amount_cents, vendor, receipt_url, material_id, paid_at, payment_method).
- Endpoints: `POST /projects/{pid}/expenses`, `GET /projects/{pid}/expenses?type=&from=&to=`, `PATCH`, `DELETE`.
- On expense write, `services/budget.py` updates `projects.spent_cents` and `phases.spent_cents` (denormalised cache; recomputed by SUM to stay correct). Each edit audited.

**Testing**:
- `Integration: add 3 expenses → project.spent_cents == sum`
- `Integration: delete expense → spent_cents decreases correctly`
- `Integration: expense edit → audit trail records before/after amount`

#### 4.4 — Budget dashboard & variance

**What**: Budget-vs-actual aggregation endpoints.

**Design**:
- `services/budget.py`:
  - `project_summary(pid)` → `{budget, spent, remaining, pct_spent}` (suggestion-1 dashboard query).
  - `breakdown_by_type(pid)` → list of `{expense_type, entries, total}`.
  - `phase_rollup(pid)` → per-phase budget/spent/remaining.
- `GET /projects/{pid}/dashboard/budget`, `.../budget/by-type`, `.../budget/by-phase`.

**Testing**:
- `Unit: project_summary with budget=100000c, spent=75000c → remaining=25000, pct=75.0`
- `Unit: budget=0 → pct_spent=0 (no divide-by-zero)`
- `Integration: by-type ordered by total desc`

---

## Phase 5: Contractors, Quotes, Permits & Photos

### Purpose
Complete the remaining MVP entities: a reusable contractor directory with ratings/insurance, side-by-side quote comparison per trade with expiry tracking, jurisdiction-aware permit checklists, and typed photo documentation (before/during/after) with media upload. After this phase the entire MVP data surface is reachable.

### Tasks

#### 5.1 — Contractors directory & quotes

**What**: User-scoped contractor directory and project-scoped quotes.

**Design**:
- `contractors` CRUD (trade enum, license_number, is_insured, rating 1–5, is_preferred), user-scoped and reusable across projects.
- `contractor_quotes` CRUD linked to project + contractor (+optional phase), with status machine `received → under_review → (accepted | rejected | expired)` and `valid_until`.
- `GET /projects/{pid}/quotes/compare?trade=` → suggestion-1 comparison query (contractor, company, rating, insured, amount, dates, status) ordered by trade then amount.

**Testing**:
- `Unit: rating=6 → ValidationError (1–5)`
- `Integration: two electrical quotes → compare returns both ordered cheapest-first`
- `Integration: accept one quote → others optionally flagged; status persisted`

#### 5.2 — Photo & document upload

**What**: Typed media attached to project/phase/task via pre-signed S3 uploads.

**Design**:
- `POST /projects/{pid}/photos/upload-url` `{filename, content_type, photo_type}` → `{upload_url, photo_id}` (pre-signed PUT to MinIO/S3). Client uploads directly; `POST /photos/{id}/confirm` finalises the `photos` row with `photo_url`/`thumbnail_url`.
- Thumbnail generation enqueued to Arq.
- photo_type enum: before/during/after/issue/receipt/permit/measurement/other.

**Testing**:
- `Integration (mocked S3): upload-url returns valid pre-signed URL + pending photo row`
- `Integration: confirm before upload → 409`
- `Integration: list photos?photo_type=before → filtered`

#### 5.3 — Permit checklist (jurisdiction-aware)

**What**: Auto-suggest required permits by project type + zip/jurisdiction, with status tracking and reminders.

**Design**:
- `permitrules/rules.yml`: maps `(project_type, jurisdiction_pattern) → [permit_type,...]` with notes. Default ruleset covers common US cases; jurisdiction resolved from project zip_code (lookup table) with a generic fallback.
- `services/permits.py: suggest_permits(project) -> list[PermitSuggestion]`.
- `POST /projects/{pid}/permits/suggest` → creates suggested permit rows (status `to_apply`/`not_needed`) the user can confirm. `permits` CRUD with status machine and `expiry_date`.

**Testing**:
- `Unit: suggest_permits(kitchen, generic) → includes building + electrical + plumbing`
- `Unit: suggest_permits(painting, generic) → no permits (not_needed)`
- `Integration: suggest → permit rows created; confirm/apply transitions persisted`

---

## Phase 6: Offline-First Sync Engine

### Purpose
Deliver the core differentiator: full offline read/write on the client with reliable background sync and automatic conflict resolution. This makes the app usable on a job site without connectivity. Requires Phases 1–5 (the entities to sync) and the client shell (built incrementally from Phase 8, but the sync protocol is server-side here).

### Tasks

#### 6.1 — Sync protocol (push/pull)

**What**: A delta-sync REST protocol over the entity tables using HLCs and tombstones.

**Design**:
- `POST /sync/pull` `{since_hlc, table_cursors}` → `{changes: {table: [rows...]}, server_hlc}` returning rows where `updated_at_hlc > cursor`, including tombstones (`deleted_at`).
- `POST /sync/push` `{device_id, mutations: [{table, op, row}]}` → applies with **field-level last-writer-wins by HLC**: for each incoming row, if incoming `updated_at_hlc` > stored, accept; else keep server and return it in `conflicts`. Response `{applied, conflicts, server_hlc}`.
- Owner/permission checks per row; mutations to non-owned rows rejected.
- All sync mutations bypass the per-endpoint audit but write a compact `actor_type='user'` audit batch.

**Testing**:
- `Unit: merge incoming HLC > stored → accept; < stored → reject and report conflict`
- `Integration: push create on device A, pull on device B → row appears`
- `Integration: concurrent edits same field, A older HLC → server keeps B, A gets conflict`
- `Integration: push delete (tombstone) → pull surfaces deletion`

#### 6.2 — Idempotency & ordering guarantees

**What**: Safe retries and dependency-respecting apply order.

**Design**:
- Each mutation carries a client `mutation_id` (UUID); server stores applied IDs in Redis (TTL 7d) → duplicate push is a no-op.
- Apply order within a push: parents before children (users→projects→phases→tasks/materials→prices/expenses…) to satisfy FKs; server topologically sorts by table rank.

**Testing**:
- `Integration: replay identical push twice → second is no-op, no duplicates`
- `Integration: push child before parent in payload → server reorders, both persist`

---

## Phase 7: AI-Native Features

### Purpose
Deliver the AI-native advantage: turn a plain-language description into a structured phase/task/material plan with a budget range, estimate material quantities, detect budget-overrun anomalies, and recommend lower-cost material alternatives. Requires the core domain (Phase 3–4). LLM calls run async via Arq.

### Tasks

#### 7.1 — LLM client abstraction & scope generation

**What**: Provider-abstracted LLM client and the "describe → structured plan" generator.

**Design**:
- `integrations/llm/client.py`: `LLMClient` Protocol with `complete_structured(system, user, schema) -> dict` (JSON-schema-constrained); `AnthropicClient` impl with prompt caching of the system prompt.
- `integrations/llm/prompts.py` — scope system prompt (abbreviated):

```
You are a home-renovation planning assistant. Given a homeowner's description,
produce a structured renovation plan as JSON conforming to the provided schema:
phases (ordered), each with tasks (name, task_type, estimated_hours) and materials
(name, category, quantity, unit, estimated_unit_cost_cents), plus a budget range
(low_cents, high_cents). Use US national-average costs. Be realistic and conservative;
flag any work that typically requires a permit.
```

- `POST /projects/{pid}/ai/scope` `{description, project_type?, zip_code?}` → enqueues Arq job; returns `{job_id}`. `GET /ai/jobs/{job_id}` → status + result.
- Result is a **proposed plan** (not auto-committed): `POST /projects/{pid}/ai/scope/{job_id}/apply` creates phases/tasks/materials with `ai_generated=true`, audited as `actor_type='ai'`. Clear AI-vs-user boundary (suggestion-3 rationale).

**Testing**:
- `Unit (mocked LLMClient): scope returns JSON validated against schema → proposal object`
- `Unit: malformed LLM output → retried once, then job failed with message`
- `Integration: apply proposal → phases/tasks/materials created, ai_generated=true, audit actor_type=ai`

#### 7.2 — Material quantity estimation

**What**: Estimate quantities from room dimensions (paint area, tile/flooring area).

**Design**:
- `POST /materials/estimate` `{material_category, room: {length_m, width_m, height_m?}, coverage?}` → deterministic formulas first (paint: wall area ÷ coverage × coats; flooring/tile: floor area ÷ unit area + waste%); LLM fallback for ambiguous categories. Returns `{quantity, unit, assumptions}`.

**Testing**:
- `Unit: paint 4×3×2.5 room, coverage 10m²/L, 2 coats → expected litres`
- `Unit: tile floor 4×3, 0.09m² tiles, 10% waste → expected count`
- `Unit: unknown category → LLM fallback invoked (mocked)`

#### 7.3 — Budget anomaly detection

**What**: Flag categories trending over budget.

**Design**:
- `services/budget.py: detect_anomalies(pid)` compares per-category actual vs. planned and run-rate vs. phase progress; returns `[{category, severity, message}]`. Heuristic (z-score / threshold) first; LLM explanation optional.
- `GET /projects/{pid}/dashboard/anomalies`.

**Testing**:
- `Unit: category spent 150% of planned → high-severity anomaly`
- `Unit: all within 10% → empty list`

#### 7.4 — Lower-cost material recommendations

**What**: Suggest cheaper alternatives meeting spec.

**Design**:
- `POST /materials/{id}/alternatives` → uses retailer provider (Phase 9) + LLM to propose alternatives with `{name, est_unit_cost_cents, rationale, tradeoffs}`. Returns suggestions only (no auto-apply).

**Testing**:
- `Unit (mocked retailer+LLM): returns ranked alternatives cheaper than current`
- `Unit: no cheaper option → empty list with explanation`

---

## Phase 8: Client Application (PWA)

### Purpose
Build the mobile-first, offline-capable PWA that homeowners actually use: project dashboard, phase/task boards, bill-of-materials, budget views, contractor comparison, permit checklist, photo capture, and the AI scope wizard — all backed by local SQLite and the Phase 6 sync engine. WCAG 2.2 compliant.

### Tasks

#### 8.1 — Client shell, local DB & API client

**What**: Vite PWA scaffold, SQLite-WASM (OPFS) mirroring the server schema, generated OpenAPI client, auth flow.

**Design**:
- `client/src/db/`: SQLite-WASM schema matching server tables + HLC/tombstone columns; query helpers.
- `client/src/api/`: TypeScript client generated from `docs/openapi.json`.
- Auth: OAuth PKCE via system browser + email/password; tokens in secure storage; silent refresh.
- Service worker caches app shell; app boots and is fully usable offline against local SQLite.

**Testing**:
- `Component: login screen validates email format`
- `E2E (Playwright): login → dashboard renders`
- `E2E offline: load app, go offline, create project → persists in local SQLite`

#### 8.2 — Project / phase / task UI

**What**: Dashboard, project detail with phase accordions, task lists with status toggles, drag-reorder, dependency indicators.

**Design**:
- Dashboard cards: per-project budget bar (spent/budget), phase progress, upcoming/overdue tasks.
- Task checkbox toggles status; blocked-by-dependency tasks visually disabled with reason.
- All writes go to local SQLite + sync queue.

**Testing**:
- `Component: budget bar at 75% renders correct width + label`
- `Component: task blocked by dependency is disabled with tooltip`
- `E2E: create phase + tasks, toggle complete → progress updates`

#### 8.3 — Materials, budget, contractors, permits, photos UI

**What**: BoM table with per-row lowest-price highlight, budget dashboard charts, contractor compare table, permit checklist, photo capture/upload.

**Design**:
- BoM rows show lowest supplier highlighted; tap to add supplier prices.
- Budget views render project_summary, by-type, by-phase.
- Contractor compare = side-by-side table per trade.
- Photo capture uses device camera → pre-signed upload (queued offline, uploaded on reconnect).
- All screens meet WCAG 2.2: ≥44px touch targets, contrast, labels.

**Testing**:
- `Component: BoM highlights lowest-price supplier row`
- `E2E offline: capture photo offline → queued, uploads on reconnect`
- `A11y (axe in Playwright): key screens have zero critical violations`

#### 8.4 — AI scope wizard UI

**What**: "Describe your project" flow that proposes a plan for review before applying.

**Design**:
- Text input + optional project_type/zip → calls `ai/scope`, polls job, renders editable proposed phases/tasks/materials with budget range; user edits then Apply.

**Testing**:
- `E2E (mocked AI): submit description → proposal shown → apply → entities created`
- `Component: user can edit/remove a proposed task before applying`

---

## Phase 9: Integrations & Notifications

### Purpose
Connect the planner to the outside world: live retailer pricing, PDF/CSV export for contractors/lenders, role-based sharing, push notifications for deadlines/quote-expiry/follow-ups, and the MCP server for AI-assistant access. These deepen value but depend on the core being complete.

### Tasks

#### 9.1 — Retailer price integration

**What**: Live price lookup via Traject BigBox with manual fallback.

**Design**:
- `integrations/retailers/base.py`: `RetailerPriceProvider.search(query, zip) -> list[PriceResult]` and `by_identifier(upc|model)`.
- `bigbox.py` impl (httpx, API key, Redis-cached 24h); `manual.py` no-op fallback when no key.
- `POST /materials/{id}/refresh-prices` enqueues a job that fetches and upserts `material_prices`, then `recompute_lowest`.

**Testing**:
- `Unit (respx): BigBox response parsed into PriceResult list`
- `Integration: refresh-prices upserts prices + recomputes lowest`
- `Unit: no API key configured → manual provider returns empty, no crash`

#### 9.2 — PDF / CSV export

**What**: Export material lists and cost summaries.

**Design**:
- `GET /projects/{pid}/export/materials.csv`, `.../materials.pdf`, `.../cost-summary.pdf`.
- WeasyPrint renders an HTML template (project header, BoM with lowest prices + line totals, budget summary, contractor quotes). CSV is deterministic column order.

**Testing**:
- `Integration: CSV export → header row + one line per material with line totals`
- `Integration: PDF export → non-empty application/pdf, contains project name (text extraction)`

#### 9.3 — Role-based sharing

**What**: Share a project with viewer/editor/contractor roles via tokenised links.

**Design**:
- `project_shares` CRUD: `POST /projects/{pid}/shares` `{email, role}` → unique `access_token`. `GET /shared/{token}` resolves a read/scoped view. Editor can mutate; contractor sees scoped subset (quotes/tasks for their trade); viewer read-only.
- Permission layer enforces role on every shared-context request.

**Testing**:
- `Integration: viewer token → GET allowed, PATCH → 403`
- `Integration: contractor token → sees only their-trade quotes/tasks`
- `Integration: revoked share token → 401`

#### 9.4 — Notifications & reminders

**What**: Scheduled push for task deadlines, quote expiry, contractor follow-ups.

**Design**:
- `services/notifications.py` + Arq cron: daily scan for due/overdue tasks, quotes nearing `valid_until`, stale follow-ups → Web Push (VAPID); FCM/APNs bridge stubbed for native.
- Client registers push subscription; `POST /me/push-subscriptions`.

**Testing**:
- `Unit: scan flags task due tomorrow and quote expiring in 2 days`
- `Integration (mocked web-push): due task → push payload dispatched once (idempotent per day)`

#### 9.5 — MCP server

**What**: Expose project state and actions to AI assistants.

**Design**:
- `mcp/server.py` tools: `list_projects`, `get_project_dashboard`, `generate_scope`, `add_material`, `log_expense` — scoped to the authenticated user (only when `users.mcp_enabled`). Reuses service layer; never bypasses auth.

**Testing**:
- `Integration: MCP get_project_dashboard returns budget summary for owner`
- `Integration: MCP call for mcp_enabled=false user → denied`

---

## Phase 10: Hardening, Compliance & Release

### Purpose
Make the system production-ready and self-hostable: security hardening to OWASP MASVS-L1, accessibility verification, performance, observability, full documentation, and a one-command self-host release.

### Tasks

#### 10.1 — Security & privacy hardening

**What**: Rate limiting, security headers, dependency/secret scanning, GDPR/CCPA controls verified end-to-end.

**Design**:
- Per-IP and per-user rate limits (Redis token bucket) on auth + AI endpoints; security headers (HSTS, CSP, X-Content-Type-Options); input size limits; CORS locked to configured origins. CCPA "Do Not Sell/Share" toggle on profile; GDPR export/erase from Phase 2 verified.

**Testing**:
- `Integration: 11th login attempt/min → 429`
- `Integration: oversized JSON body → 413`
- `E2E: erase account → all data inaccessible, export empty`

#### 10.2 — Accessibility & performance pass

**What**: WCAG 2.2/WCAG2Mobile audit and load/latency checks.

**Design**:
- axe-core gating in Playwright across all screens; manual touch-target + contrast check; dashboard P95 < 300ms server-side on a 50-task/100-material project; sync of 1k rows < 2s.

**Testing**:
- `E2E a11y: all primary screens zero critical/serious axe violations`
- `Perf: dashboard endpoint P95 < 300ms under seeded data`

#### 10.3 — Docs, OpenAPI & self-host release

**What**: Finalise OpenAPI 3.1 spec, write self-host + API docs, ship docker-compose release.

**Design**:
- Commit generated `docs/openapi.json`; verify all endpoints present and schema-valid. README self-host quickstart (`docker compose up`). Seed/demo script. Tag `v1.0.0`.

**Testing**:
- `Integration: generated OpenAPI validates against OAS 3.1 meta-schema; every router path present`
- `E2E: fresh docker compose up → register → create AI-scoped project → export PDF (full happy path)`

---

## Phase Summary & Dependencies

```
Phase 1: Foundation & Schema            ─── required by everything
    │
Phase 2: Auth & Users                   ─── requires Phase 1
    │
Phase 3: Core Domain (Proj/Phase/Task)  ─── requires Phase 2
    │
Phase 4: Materials/Pricing/Budget       ─── requires Phase 3
    │
Phase 5: Contractors/Quotes/Permits/Photos ─ requires Phase 3 (4 for budget links)
    │
    ├── Phase 6: Offline Sync Engine    ─── requires Phases 3–5 (entities to sync)
    ├── Phase 7: AI Features            ─── requires Phases 3–4   ┐ can parallel
    └── Phase 9: Integrations/Notif/MCP ─── requires Phases 3–5   ┘ with 6 & 7
         │
Phase 8: Client PWA                     ─── requires Phase 6 (sync) + 2 (auth);
    │                                        consumes 3–5, 7 incrementally
Phase 10: Hardening/Compliance/Release  ─── requires all prior phases
```

**Parallelism opportunities:**
- After Phase 5: **Phases 6, 7, and 9 can be developed concurrently** (different teams: sync engine, AI, integrations) since they share only the completed service layer.
- **Phase 8 (client)** can begin its shell (8.1) in parallel with Phase 6 once the sync protocol contract is fixed, then integrate 6/7/9 features as they land.
- Within Phase 9, tasks 9.1–9.5 are independent and parallelisable.

---

## Definition of Done (per phase)

Every phase is complete only when:

1. All tasks implemented as specified.
2. All unit and integration tests pass (`pytest` backend, `vitest`/Playwright client).
3. Linting and formatting pass (`ruff check`, `ruff format --check`; ESLint/Prettier).
4. Type checking passes (`mypy --strict`; `tsc --noEmit`).
5. Alembic migration created and `upgrade head`/`downgrade base` run cleanly (schema-affecting phases).
6. Docker build succeeds and `docker compose up` boots the affected services.
7. The phase's feature works end-to-end (demonstrable via API or UI).
8. New/changed endpoints appear in the generated OpenAPI 3.1 spec and validate.
9. New config options documented in `.env.example` and README.
10. Audit-relevant mutations write `audit_log` entries with the correct `actor_type`.
11. Accessibility checks pass for any new client screens (axe-core, no critical/serious violations).
12. Security-sensitive endpoints (auth, sharing, AI, sync) have negative-path tests (401/403/404/409/429).
```
