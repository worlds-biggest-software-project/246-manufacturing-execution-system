# Manufacturing Execution System (MES) — Phased Development Plan

> Project: 246-manufacturing-execution-system · Created: 2026-05-29
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

This plan delivers an open-source, AI-native MES aligned with ISA-95, ISA-88, ISO 22400, OPC-UA, and 21 CFR Part 11. The architecture is a relational ISA-95 backbone augmented with an append-only domain-event log that powers the audit trail, AI/ML pipelines, and projections such as OEE.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---|---|---|
| Primary backend language | Python 3.12 | Strong ecosystem for ML/AI (the differentiating capability), mature OPC-UA / MQTT / Sparkplug libraries (`asyncua`, `paho-mqtt`, `tahu`), Pydantic for ISA-95/B2MML data validation. |
| API framework | FastAPI 0.115+ | Native async (required for OPC-UA / MQTT bridges and streaming OEE), first-class OpenAPI 3.1 generation (a project requirement), Pydantic v2 models. |
| Web UI framework | Next.js 15 (App Router) + React 19 + TypeScript | Operator panels need touch-friendly, real-time, role-based UIs; SSR for executive dashboards; mature ecosystem for SPC charting and shop-floor displays. |
| UI component library | shadcn/ui + Tailwind CSS | Composable, accessible, themable for industrial dashboards; avoids vendor lock-in. |
| Charting | Apache ECharts + Recharts | ECharts for SPC/OEE waterfalls; Recharts for simple dashboards. |
| Primary database | PostgreSQL 16 | ISA-95 relational integrity, JSONB for ISA-95 extension attributes, range partitioning for time-series, GIN indexes for event metadata search, mature audit/RLS support for 21 CFR Part 11. |
| Time-series extension | TimescaleDB 2.16 (PostgreSQL extension) | Hypertables compress and partition the high-cardinality `machine_data_point` and `event_store` tables; continuous aggregates power minute/hour/day OEE rollups. |
| Database migrations | Alembic | Standard for SQLAlchemy / Python; supports auto-generation against the ORM. |
| ORM / data layer | SQLAlchemy 2.0 (async) + SQLModel for API DTOs | Async-aware for FastAPI; SQLModel unifies Pydantic + ORM where appropriate. |
| Cache / queue broker | Redis 7 (Stack) | Pub/sub for the real-time projection bus, RedisStream for command/event fan-out, RedisJSON for the operator-panel state cache. |
| Task queue | Celery 5 (with Redis broker) | Mature retry/backoff for OPC-UA reconnects, scheduled jobs (OEE rollups, certificate expiry, KPI alerts), AI inference dispatch. |
| Async streaming | Kafka 3.7 (optional; Redis Streams in MVP) | When deployed at scale, Kafka replaces Redis Streams for the event bus; the design keeps Kafka and Redis Streams behind a common `EventBus` interface. |
| MQTT broker (recommended runtime) | EMQX 5 (HiveMQ also supported) | Open source, Sparkplug B aware, supports the unified-namespace pattern. |
| OPC-UA library | `asyncua` (Python, AGPL-compatible) | Async OPC-UA client and server in pure Python; supports subscriptions, security, and the ISA-95 companion spec. |
| MQTT / Sparkplug B library | `paho-mqtt` + Eclipse Tahu (`pytahu`) | Standards-tracking Sparkplug B implementation. |
| AI / LLM framework | OpenAI-compatible client + LangChain Core + Anthropic SDK | LLM-agnostic (vLLM, Anthropic, OpenAI, Azure OpenAI) for the natural-language reporting agent; LangChain only for the agent loop, not as a hard dependency. |
| Vector search | pgvector (PostgreSQL extension) | Keeps the deployment to one database; supports the "ask shop floor" semantic search over shift reports, SOPs, and inspection notes. |
| ML / numerical | scikit-learn 1.5, pandas 2.2, PyTorch 2.4 (optional) | scikit-learn for OEE root-cause clustering; PyTorch for edge vision models in the optional CV phase. |
| Computer vision (edge) | ONNX Runtime + OpenCV (Python edge agent) | Vendor-neutral inference; runs on ARM and x86 edge gateways. |
| Authentication | Keycloak 25 (OIDC / OAuth 2.0) — embedded `python-jose` + `Authlib` client | OIDC is the standard for industrial IT; Keycloak supports federated AD/LDAP for OT environments; satisfies IEC 62443 SL-2. |
| Authorization | Casbin (Python) | RBAC + ABAC for site/role/equipment scopes (`person_role(site_id)` enforcement); declarative policy independent of code. |
| Secrets management | HashiCorp Vault (production) / Docker secrets (dev) | Required for IEC 62443 SL-2 baseline. |
| Containerisation | Docker + Docker Compose (dev/small sites) / Helm 3 + Kubernetes (multi-site) | Standard self-hosted distribution; Helm chart for multi-site or cloud deployments. |
| Edge runtime | k3s + a "MES Edge Agent" container | Lightweight Kubernetes for plant-floor edge boxes that run OPC-UA bridge, MQTT bridge, and inference. |
| API documentation | OpenAPI 3.1 (FastAPI native) + Redocly | Required by the project. |
| Testing — Python | pytest, pytest-asyncio, pytest-cov, hypothesis (property-based), Testcontainers (PostgreSQL/Redis/EMQX) | Real dependencies for integration; mocks only for external SaaS. |
| Testing — Frontend | Vitest + React Testing Library + Playwright (E2E) | Industry-standard for Next.js. |
| Load / soak testing | k6 | Validate sustained 10k events/s ingestion. |
| Linting / formatting | Ruff (Python), Biome (TS/JS), Prettier for Markdown | Single tool per language, fast. |
| Type checking | mypy (strict mode) + Pyright as a second pass; TypeScript strict | Strong typing is the cheapest defence against MES correctness bugs. |
| Package management | uv (Python), pnpm (TS) | Faster, lockfile-based. |
| CI / CD | GitHub Actions | Matrix builds for Python and Node, Docker image publishing to GHCR, Trivy security scan. |
| Observability | OpenTelemetry SDK + Prometheus + Grafana + Loki | Operator-grade telemetry for both the application and the OPC-UA / MQTT bridges. |
| Error tracking | Sentry (self-hosted) | Optional but recommended; redacts PHI/PII before ship. |
| Documentation site | MkDocs Material | Standards mapping pages, operator guide, integrator guide. |
| Licensing | Apache 2.0 (server), AGPL-3.0 (optional dual licence for enterprise extensions) | Apache 2.0 for adoption; allows commercial deployment without copyleft burden. |

### Data Model Strategy

The plan adopts a hybrid of **data-model-suggestion-1 (ISA-95 relational backbone)** and **data-model-suggestion-2 (event-sourced audit trail)**:

- The relational ISA-95 schema (personnel, equipment, material, product definition, routing, recipe, production order, work order, OEE, quality) is the system of record for operational state.
- A single append-only `event_store` table (TimescaleDB hypertable) records every domain event — work-order transitions, equipment-state changes, material consumption, inspection results, e-signatures. This satisfies 21 CFR Part 11 / EU Annex 11 by construction and is the input to all AI/ML pipelines.
- Read models are the relational tables themselves, updated transactionally by command handlers in the same database transaction as the event append (outbox pattern within PostgreSQL). This eliminates eventual-consistency edge cases for shop-floor UIs.
- Suggestion-3-style JSONB `properties` columns are retained on `equipment`, `material_definition`, `inspection_result`, and `work_order` for vertical-specific extension fields (PPAP, GMP, allergen, etc.).
- Suggestion-4's graph queries are implemented via recursive CTEs over the `work_order_material` genealogy table — sufficient for MVP and v1.1. A Neptune/Neo4j sync is left to the backlog.

### Project Structure

```
mes-platform/
├── README.md
├── LICENSE
├── CONTRIBUTING.md
├── docs/                                # MkDocs site
│   ├── architecture/
│   ├── standards/                       # ISA-95, ISA-88, ISO 22400, 21 CFR Part 11 mappings
│   ├── operator-guide/
│   └── integrator-guide/
├── deploy/
│   ├── docker-compose.yml               # Dev + small-site deployment
│   ├── docker-compose.edge.yml          # Edge agent only
│   ├── helm/                            # Multi-site / Kubernetes
│   └── env/.env.example
├── server/                              # Python backend (FastAPI)
│   ├── pyproject.toml
│   ├── uv.lock
│   ├── Dockerfile
│   ├── alembic.ini
│   ├── alembic/
│   │   └── versions/
│   ├── src/mes/
│   │   ├── __init__.py
│   │   ├── main.py                      # FastAPI app entry
│   │   ├── config.py                    # Pydantic Settings
│   │   ├── auth/                        # OIDC + Casbin
│   │   ├── core/
│   │   │   ├── db.py                    # Async engine, session
│   │   │   ├── events.py                # Event bus interface
│   │   │   ├── audit.py                 # 21 CFR Part 11 helpers
│   │   │   └── time.py                  # ISO 22400 time-state classifier
│   │   ├── domain/                      # ISA-95 aggregates (pure Python)
│   │   │   ├── personnel/
│   │   │   ├── equipment/
│   │   │   ├── material/
│   │   │   ├── product/
│   │   │   ├── recipe/                  # ISA-88
│   │   │   ├── production/              # work_order, production_order
│   │   │   ├── quality/
│   │   │   ├── oee/                     # ISO 22400
│   │   │   └── maintenance/
│   │   ├── api/                         # FastAPI routers
│   │   │   ├── v1/
│   │   │   │   ├── work_orders.py
│   │   │   │   ├── equipment.py
│   │   │   │   ├── oee.py
│   │   │   │   ├── quality.py
│   │   │   │   ├── materials.py
│   │   │   │   ├── auth.py
│   │   │   │   ├── webhooks.py
│   │   │   │   ├── b2mml.py             # ISA-95 B2MML import/export
│   │   │   │   └── mcp.py               # MCP server endpoints
│   │   ├── integrations/
│   │   │   ├── opcua/                   # asyncua client + bridge
│   │   │   ├── mqtt/                    # paho + Sparkplug B
│   │   │   ├── erp/                     # SAP/Oracle webhook + REST clients
│   │   │   ├── b2mml/                   # ISA-95 XML schema bindings
│   │   │   └── mtconnect/               # CNC HTTP client
│   │   ├── ai/
│   │   │   ├── rca/                     # OEE root-cause analysis
│   │   │   ├── scheduler/               # Adaptive scheduling
│   │   │   ├── yield_prediction/
│   │   │   ├── nl_reporting/            # LLM agent + RAG
│   │   │   └── vision/                  # Optional CV inference glue
│   │   ├── workers/                     # Celery tasks
│   │   ├── tasks/                       # Scheduled rollups (Celery beat)
│   │   └── telemetry/                   # OTel, Prometheus
│   └── tests/
│       ├── unit/
│       ├── integration/
│       ├── e2e/
│       └── fixtures/
├── edge-agent/                          # Lightweight Python edge runtime
│   ├── pyproject.toml
│   ├── Dockerfile
│   └── src/edge/
│       ├── opcua_bridge.py
│       ├── mqtt_bridge.py
│       ├── vision_runtime.py
│       └── store_and_forward.py         # SQLite outbox for disconnected ops
├── web/                                 # Next.js operator + executive UIs
│   ├── package.json
│   ├── pnpm-lock.yaml
│   ├── Dockerfile
│   ├── next.config.mjs
│   ├── app/
│   │   ├── (operator)/
│   │   │   ├── work-orders/
│   │   │   ├── inspections/
│   │   │   └── downtime/
│   │   ├── (supervisor)/
│   │   │   ├── oee/
│   │   │   ├── scheduling/
│   │   │   └── reports/
│   │   ├── (admin)/
│   │   │   ├── equipment/
│   │   │   ├── recipes/
│   │   │   └── users/
│   │   └── api/                         # BFF (proxy to FastAPI)
│   ├── components/
│   ├── lib/
│   └── tests/
├── schemas/                             # Shared JSON Schema / B2MML XSD
│   ├── events/                          # JSON Schema per event type
│   └── b2mml/                           # Vendored ISA-95 XSDs
└── .github/
    └── workflows/                       # CI matrix
```

---

## Phase 1: Foundation — Repository, CI, and Core Schema

### Purpose

Lay the technical foundation: monorepo layout, package management, PostgreSQL + TimescaleDB, Alembic, configuration, base OIDC authentication, Docker Compose for local development, CI pipeline, and the minimal ISA-95 organisation schema (tenant → site → area → work_center → equipment). After this phase, contributors can run the stack locally with `docker compose up` and a healthcheck endpoint returns 200.

### Tasks

#### 1.1 — Monorepo and tooling bootstrap

**What**: Initialise the repository structure, package managers, formatters, linters, type checkers, and CI matrix.

**Design**:
- Top-level layout per "Project Structure" above.
- Python: `uv init` in `server/` and `edge-agent/`; pin Python 3.12. `pyproject.toml` sets:
  ```toml
  [project]
  name = "mes-server"
  requires-python = ">=3.12,<3.13"
  dependencies = [
      "fastapi>=0.115", "uvicorn[standard]>=0.32", "pydantic>=2.9",
      "pydantic-settings>=2.5", "sqlalchemy[asyncio]>=2.0",
      "asyncpg>=0.30", "alembic>=1.13", "redis>=5.1",
      "celery>=5.4", "casbin>=1.36",
      "asyncua>=1.1", "paho-mqtt>=2.1",
      "structlog>=24.4", "opentelemetry-sdk>=1.27",
  ]
  [tool.ruff]
  line-length = 100
  select = ["E","F","I","B","UP","ANN","SIM","TID","RUF"]
  [tool.mypy]
  strict = true
  ```
- TS: `pnpm init` in `web/`; install Next 15, React 19, Tailwind, shadcn/ui, ECharts, Vitest, Playwright.
- Pre-commit: `pre-commit-hooks`, `ruff`, `biome`, `mypy`, `pytest -q`.
- GitHub Actions workflows: `ci-server.yml` (lint, type, test on 3.12), `ci-web.yml` (lint, vitest, build), `ci-edge.yml`, `docker.yml` (build + push to GHCR on tag), `security.yml` (Trivy + pip-audit).

**Testing**:
- Unit: `ruff check` passes on empty `src/mes/__init__.py`.
- Unit: `mypy --strict src/` passes on a stub module.
- Integration: `pnpm build` in `web/` produces a `.next/` directory.
- E2E (CI): A push to a feature branch triggers all three CI jobs and all pass green.

#### 1.2 — Configuration and secrets

**What**: Pydantic Settings configuration loaded from environment variables and `.env`, with secret indirection via Docker secrets / Vault.

**Design**:
```python
# server/src/mes/config.py
from pydantic_settings import BaseSettings, SettingsConfigDict
from pydantic import PostgresDsn, RedisDsn, AnyUrl, SecretStr

class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env", env_prefix="MES_")

    env: str = "dev"
    debug: bool = False

    database_url: PostgresDsn
    redis_url: RedisDsn
    broker_url: RedisDsn  # Celery
    result_backend: RedisDsn

    oidc_issuer: AnyUrl
    oidc_audience: str
    oidc_client_id: str
    oidc_client_secret: SecretStr

    secret_key: SecretStr            # JWT cookie signing for the BFF
    encryption_key: SecretStr        # AES-256 for at-rest field encryption

    opcua_default_security_policy: str = "Basic256Sha256"
    mqtt_broker_url: AnyUrl | None = None
    mqtt_use_sparkplug: bool = True

    llm_provider: str = "anthropic"  # anthropic | openai | azure | vllm
    llm_model: str = "claude-opus-4-7"
    llm_api_key: SecretStr | None = None

    iec_62443_security_level: int = 2

settings = Settings()  # type: ignore[call-arg]
```

**Testing**:
- Unit: missing required env var → `ValidationError` mentions the missing field name.
- Unit: `MES_DATABASE_URL=postgresql+asyncpg://...` parsed into a `PostgresDsn`.
- Unit: `SecretStr` value not present in `repr(settings)`.

#### 1.3 — Docker Compose stack for local dev

**What**: A `deploy/docker-compose.yml` that brings up PostgreSQL+TimescaleDB+pgvector, Redis, EMQX, Keycloak, MailHog (for alert testing), and the server in hot-reload mode.

**Design**:
- Services: `db` (timescale/timescaledb-ha:pg16-all with pgvector extension), `redis` (redis/redis-stack:7.4), `mqtt` (emqx/emqx:5.8), `auth` (quay.io/keycloak/keycloak:25.0 — dev mode, realm import from `deploy/keycloak/mes-realm.json`), `server` (build from `server/Dockerfile`), `web`, `worker` (Celery), `beat`, `mailhog`.
- Healthchecks on every service; named volumes for DB and EMQX.
- `make up`, `make down`, `make logs`, `make psql`, `make test` shortcuts in `Makefile`.

**Testing**:
- Integration: `docker compose up -d && docker compose ps` shows all services healthy within 60s.
- Integration: `curl http://localhost:8000/healthz` returns `{"status":"ok","db":"up","redis":"up","mqtt":"up"}`.
- Integration: Keycloak admin console reachable at `localhost:8080/admin`.

#### 1.4 — Database baseline migration and TimescaleDB enablement

**What**: First Alembic migration that enables `timescaledb` and `pgvector`, creates the `tenant`, `site`, `area`, `work_center` tables and the `equipment` table with self-referential ISA-95 hierarchy.

**Design**:
- DDL from `data-model-suggestion-1.md` sections "Core Infrastructure Tables" and "Equipment Model" verbatim, plus:
  ```sql
  CREATE EXTENSION IF NOT EXISTS timescaledb;
  CREATE EXTENSION IF NOT EXISTS vector;
  CREATE EXTENSION IF NOT EXISTS pgcrypto;
  ```
- `equipment.properties JSONB DEFAULT '{}'` added (suggestion-3 extension pattern).
- Row-level security policies created for every tenant-scoped table:
  ```sql
  ALTER TABLE site ENABLE ROW LEVEL SECURITY;
  CREATE POLICY tenant_isolation ON site
      USING (tenant_id = current_setting('mes.tenant_id')::uuid);
  ```
- Each table gets `created_by`, `updated_by` columns referencing `person(id)`.

**Testing**:
- Unit: `alembic upgrade head` against a fresh Postgres image succeeds; `alembic downgrade base` reverses cleanly.
- Integration (Testcontainers): inserting a `site` for tenant A and connecting as tenant B → SELECT returns zero rows.
- Integration: creating an `equipment` row with `parent_equipment_id` referencing itself raises a `CHECK` violation (recursion guard via deferred trigger).

#### 1.5 — Health, readiness, and base FastAPI app

**What**: FastAPI application skeleton with `/healthz`, `/readyz`, `/version`, `/metrics` (Prometheus), and OpenAPI generation.

**Design**:
- `mes.main` exposes `app = FastAPI(title="MES Platform", version=__version__, openapi_url="/openapi.json", docs_url="/docs")`.
- Routers mounted at `/api/v1`.
- Middleware: `RequestIDMiddleware`, `TenantContextMiddleware` (reads `X-Tenant-Slug`, sets `SET LOCAL mes.tenant_id`), `OTelMiddleware`.
- `/readyz` returns 200 only when DB, Redis, and the MQTT broker (if configured) all answer within 1s.

**Testing**:
- Unit: `TestClient(app).get("/healthz").json() == {"status":"ok"}`.
- Integration: stop the DB container; `/readyz` returns 503 with `{"db":"down"}`.
- Unit: `GET /openapi.json` returns a valid OpenAPI 3.1 document (`openapi == "3.1.0"`).

---

## Phase 2: Identity, Personnel, RBAC, and Audit Foundation

### Purpose

Stand up the security and accountability bedrock: OIDC authentication, ISA-95 personnel model, RBAC via Casbin, the immutable `event_store` and `audit_log` tables, and the 21 CFR Part 11 e-signature primitive. Every later phase depends on these to record who did what. After this phase, an administrator can create users with roles and every write to the system is attributable.

### Tasks

#### 2.1 — Personnel and RBAC schema

**What**: Migration adding the ISA-95 personnel tables, RBAC tables, and Casbin policy table.

**Design**: DDL from suggestion 1, "Personnel Model" section, plus a `casbin_rule` table managed by `casbin-sqlalchemy-adapter`. The `permission` table is seeded with a fixed list:
```
work_order:create, work_order:dispatch, work_order:complete,
inspection:perform, inspection:approve,
material:consume, material:quarantine,
equipment:configure, recipe:approve,
audit:read, esignature:apply,
admin:tenant, admin:site
```

**Testing**:
- Unit: `person_role` PK enforces uniqueness across `(person_id, role_id, site_id)`.
- Integration: revoking a role removes Casbin policies on cascade.

#### 2.2 — OIDC authentication and session management

**What**: Authentication code flow with PKCE for users; client-credentials flow for ERP/IIoT API access.

**Design**:
- `mes.auth.oidc` wraps `Authlib`'s async OAuth2 client; discovers Keycloak via the issuer URL.
- `Depends(current_user)` returns a `User` Pydantic model containing `person_id, tenant_id, roles, sites`.
- JWT validation uses Keycloak's JWKS; cached for 1h.
- `POST /api/v1/auth/login` redirects to Keycloak; callback at `/api/v1/auth/callback`; session cookie is signed with `secret_key`, httpOnly, sameSite=lax, secure in prod.
- Service-account tokens validated via the same JWKS; scope `mes.api.full` required.

**Testing**:
- Integration (Keycloak Testcontainer): valid auth-code flow returns 200 and sets cookie.
- Integration: malformed JWT → 401 with `{"error":"invalid_token"}`.
- Integration: expired JWT → 401 with `{"error":"token_expired"}`.
- Unit: `current_user` raises `InsufficientPermissions` when required role missing.

#### 2.3 — Casbin policy enforcement

**What**: Casbin RBAC + site scoping; FastAPI dependency `Requires(action, resource)`.

**Design**:
- Casbin model file at `server/src/mes/auth/casbin_model.conf`:
  ```
  [request_definition]
  r = sub, obj, act, site
  [policy_definition]
  p = sub, obj, act, site
  [role_definition]
  g = _, _
  g2 = _, _
  [policy_effect]
  e = some(where (p.eft == allow))
  [matchers]
  m = g(r.sub, p.sub) && r.obj == p.obj && r.act == p.act && (p.site == "*" || r.site == p.site)
  ```
- Policies seeded per role from a YAML at `server/config/rbac.yml`.

**Testing**:
- Unit: an operator with `work_order:complete` scoped to `site:plantA` can complete a work order at plant A but not plant B.
- Unit: `admin:tenant` implies all actions across all sites of that tenant.

#### 2.4 — Event store and audit log

**What**: TimescaleDB hypertable `event_store` (from suggestion 2) and trigger-based `audit_log` (from suggestion 1) created. Application-level `EventRecorder` writes events transactionally.

**Design**:
- Migration creates `event_store` exactly per suggestion 2 then:
  ```sql
  SELECT create_hypertable('event_store', 'recorded_at', chunk_time_interval => INTERVAL '7 days');
  CREATE INDEX ON event_store (stream_id, sequence_number DESC);
  ```
- `audit_log` is implemented as a generic PL/pgSQL trigger applied to every business table:
  ```sql
  CREATE TRIGGER trg_audit AFTER INSERT OR UPDATE OR DELETE
  ON work_order FOR EACH ROW EXECUTE FUNCTION mes_audit_capture();
  ```
- Python API:
  ```python
  class EventRecorder:
      async def append(
          self, *,
          stream_type: str, stream_id: UUID, event_type: str,
          payload: dict, actor: User, occurred_at: datetime | None = None,
          correlation_id: UUID | None = None,
      ) -> Event: ...
  ```
- Each event has a JSON Schema registered in `schemas/events/<EventType>.json` and validated before insert.

**Testing**:
- Unit: appending an event with `sequence_number` already taken raises `ConcurrencyError`.
- Unit: payload not matching its registered JSON Schema raises `EventValidationError`.
- Integration: updating a `work_order` row creates an `audit_log` entry with the diff in `old_values`/`new_values`.
- Integration: attempt to `UPDATE event_store` raises a permission error (revoked role).

#### 2.5 — Electronic signatures (21 CFR Part 11 primitive)

**What**: `POST /api/v1/esignatures` records a tamper-evident signature on any record.

**Design**:
- Table per suggestion 1 + a hash function `compute_record_hash(table, id) -> sha256` that serialises the row in canonical JSON and hashes it.
- Request body:
  ```json
  {
    "table_name": "work_order",
    "record_id": "uuid",
    "meaning": "approved",
    "reason": "End of shift release",
    "password": "..."        // re-authentication per 21 CFR 11.200
  }
  ```
- Re-authentication against Keycloak before insert; signature also emitted as `ESignatureApplied` event.

**Testing**:
- Integration: signing a record then mutating it makes the verification endpoint return `valid=false, reason="hash_mismatch"`.
- Integration: missing `password` → 400; wrong password → 401 with audit-logged failure.
- Integration: e-signature appears in the audit trail with `meaning` and `signer_full_name`.

---

## Phase 3: Production Master Data — Materials, Products, Routings, Recipes

### Purpose

Add the ISA-95 product-definition and ISA-88 recipe schemas, plus CRUD APIs and admin UI screens. After this phase, an administrator can define materials, BOMs, product definitions, routings, and master recipes — the prerequisites for creating any work order.

### Tasks

#### 3.1 — Material, BOM, and lot schema and API

**What**: Migrations for `material_class`, `material_definition`, `material_property`, `material_lot`, `bill_of_materials`. CRUD API at `/api/v1/materials`, `/api/v1/materials/{id}/bom`, `/api/v1/lots`.

**Design**: DDL per suggestion 1 "Material Model" section. Pydantic models:
```python
class MaterialDefinitionIn(BaseModel):
    part_number: str = Field(min_length=1, max_length=100)
    name: str
    material_class_id: UUID | None
    unit_of_measure: str
    is_lot_tracked: bool = True
    is_serial_tracked: bool = False
    shelf_life_days: int | None = None
    properties: dict[str, Any] = {}     # vertical-specific (allergen, hazard class)
```
- BOM endpoint accepts a nested tree and enforces no cycles (recursive CTE check).
- `POST /api/v1/lots` emits `MaterialLotReceived` event.

**Testing**:
- Unit: BOM with a cycle (`A → B → A`) → 400 `cycle_detected`.
- Unit: lot quantity must be > 0.
- Integration: serial-tracked material requires `serial_number` on lot creation.
- E2E: create material → create BOM → fetch BOM tree returns expected hierarchy.

#### 3.2 — Product definition and routing

**What**: Schemas and APIs for `product_definition`, `routing`, `routing_step`, `routing_step_material`.

**Design**: DDL per suggestion 1. Approval workflow: `POST /api/v1/products/{id}/approve` transitions status `draft → approved` and requires an e-signature with `meaning="approved"`. A `routing.status='active'` precondition is enforced at work-order creation.

**Testing**:
- Unit: routing-step sequence numbers must be strictly increasing and unique per routing.
- Integration: approving a product definition without e-signature → 400.
- Integration: creating a work order against an inactive routing → 422.

#### 3.3 — ISA-88 master recipe

**What**: Migrations for `master_recipe`, `recipe_procedure`, `recipe_unit_procedure`, `recipe_operation`, `recipe_phase`, `recipe_parameter`. API to author and approve recipes.

**Design**:
- Endpoint `POST /api/v1/recipes` accepts a nested ISA-88 procedure tree.
- `recipe_parameter.is_critical=true` parameters require an e-signature when changed.
- Recipe `status` workflow: `draft → approved → active → obsolete`; only one `active` version per `(tenant_id, name)`.
- Export endpoint `GET /api/v1/recipes/{id}/b2mml` serialises to a B2MML `MasterRecipe` XML.

**Testing**:
- Unit: changing a critical parameter on an active recipe → 400 unless e-signature attached.
- Integration: activating a new version automatically marks the prior `active` version `obsolete`.
- Fixture: round-trip a vendored B2MML sample recipe and compare to expected JSON.

#### 3.4 — Admin UI for master data

**What**: Next.js admin screens for materials, BOMs, products, routings, and recipes.

**Design**:
- `app/(admin)/materials/page.tsx` — data table (TanStack Table) with server-side filtering and the BOM tree editor.
- shadcn `Dialog`, `Form`, `Combobox`, `Tabs` used throughout.
- Real recipe authoring: split-pane editor with the procedural tree on the left and the parameter form on the right.

**Testing**:
- Vitest: BOM editor adds, edits, and deletes nodes correctly.
- Playwright: admin user logs in, creates a material, attaches a BOM, sees it in the list.

---

## Phase 4: Production Execution — Work Orders and Operator Panel

### Purpose

Deliver the heart of the MES: production orders, work orders at each routing step, operator panels with step-guided execution, error-proofing gates, material consumption, and the immutable transaction trail. After this phase, an operator can pick up a job, work through its steps, scan material lots, and complete units — with every action recorded as both a relational state change and an event.

### Tasks

#### 4.1 — Production order and work order schemas

**What**: Migrations for `production_order`, `work_order`, `work_order_material`, `shift_definition`, `labor_record`.

**Design**: DDL per suggestion 1 "Production Scheduling and Execution". Add a `work_order.state_machine` enforced via PL/pgSQL trigger so invalid transitions (e.g. `completed → in_progress`) are rejected at the DB layer as well as the API.

**Testing**:
- Unit: state machine table; only documented transitions allowed; invalid transition raises a domain error.
- Integration: `completed_quantity + scrapped_quantity` cannot exceed `planned_quantity * 1.2` (over-build tolerance) without an e-signature.

#### 4.2 — Production order creation and explosion

**What**: `POST /api/v1/production-orders` creates a `production_order` and explodes the routing into one `work_order` per routing step.

**Design**:
- Service `ProductionOrderService.create(...)` runs in a single DB transaction:
  1. Insert `production_order`.
  2. For each `routing_step` of the active routing, insert a `work_order` with `status='pending'`, `sequence_number` copied.
  3. Reserve `material_lot` rows of FIFO-selected lots for the input materials of step 1 (`status='reserved'`).
  4. Emit `ProductionOrderCreated` and one `WorkOrderCreated` per step.
- ERP import: `POST /api/v1/erp/production-orders/b2mml` accepts B2MML `ProductionSchedule` XML and creates the order.

**Testing**:
- Unit: routing with 3 steps → 3 work orders created with correct sequence and equipment.
- Integration: insufficient lot quantity for step 1 → 422 with detailed shortfall message and no rows inserted.
- Integration: round-trip a B2MML `ProductionSchedule` sample.

#### 4.3 — Operator API for execution

**What**: REST API for operator actions: claim, start, pause, resume, consume material, produce unit, scrap unit, complete step, raise issue.

**Design**:
```
POST /api/v1/work-orders/{id}/claim     -> assigns current_user as operator
POST /api/v1/work-orders/{id}/start
POST /api/v1/work-orders/{id}/pause     {reason}
POST /api/v1/work-orders/{id}/consume   {material_lot_id, quantity}
POST /api/v1/work-orders/{id}/produce   {quantity, lot_number?, serial_number?}
POST /api/v1/work-orders/{id}/scrap     {quantity, reason}
POST /api/v1/work-orders/{id}/complete
```
- Each call: validate state transition → write event → mutate relational state → broadcast WS message.
- Error-proofing: `produce` blocked if a required inspection gate has not passed.

**Testing**:
- Unit: state machine: `pending → ready → in_progress → paused → in_progress → completed`.
- Integration: consuming a lot whose `status='quarantined'` → 409.
- E2E: operator runs the full happy path and the `work_order` ends at `status='completed'` with `completed_quantity == planned_quantity`.

#### 4.4 — Operator panel UI

**What**: Touch-friendly operator panel — large fonts, big buttons, barcode-scanner input, step list, current-step detail.

**Design**:
- `app/(operator)/work-orders/page.tsx` shows assigned jobs sorted by priority.
- `app/(operator)/work-orders/[id]/page.tsx` shows current step with: instructions, required materials (scan-to-consume), required tooling, quantity tally (+ / -), pause/resume controls, complete button.
- Real-time updates via WebSocket (`/ws/work-orders/{id}`); fallback to polling.
- Offline-safe: actions queued in IndexedDB if the network drops; replayed when reconnected.

**Testing**:
- Vitest: barcode input parsed (Code-128 / GS1-128).
- Playwright: operator completes a job end-to-end in the UI and the work order is `completed` in the DB.
- Playwright: drop network during execution; reconnect; queued events flush; final state correct.

#### 4.5 — Genealogy queries

**What**: APIs for forward and reverse genealogy.

**Design**:
```
GET /api/v1/lots/{lot_id}/forward   -> all product lots downstream
GET /api/v1/lots/{lot_id}/reverse   -> all raw-material lots upstream
```
Implemented via recursive CTE over `work_order_material`. Response includes the path and time of each consumption.

**Testing**:
- Fixture: a 3-level genealogy (raw → intermediate → finished); forward and reverse return correct sets.
- Integration: recall scenario — given a raw lot, retrieve all finished lots in <100ms for 10k events.

---

## Phase 5: Machine Connectivity — OPC-UA, MQTT/Sparkplug B, MTConnect

### Purpose

Bring shop-floor machines into the system as a real-time event source. Implement an OPC-UA client, an MQTT/Sparkplug B subscriber, and an optional MTConnect client; map tags to equipment data points; ingest values into `machine_data_point` (TimescaleDB hypertable) and emit `SensorReadingReceived` events. This is the data backbone for OEE and AI.

### Tasks

#### 5.1 — Data-point definitions and ingestion schema

**What**: Migrations for `data_point_definition` and `machine_data_point`; create the TimescaleDB hypertable and compression policy.

**Design**:
- `machine_data_point` per suggestion 1, then:
  ```sql
  SELECT create_hypertable('machine_data_point', 'timestamp', chunk_time_interval => INTERVAL '1 day');
  ALTER TABLE machine_data_point SET (timescaledb.compress, timescaledb.compress_segmentby='equipment_id');
  SELECT add_compression_policy('machine_data_point', INTERVAL '7 days');
  SELECT add_retention_policy('machine_data_point', INTERVAL '180 days');  -- configurable
  ```
- Continuous aggregates for minute and hour averages.

**Testing**:
- Integration: insert 1 million rows for one equipment over 8 hours; query the hour CA returns 8 rows.
- Integration: data older than the compression policy gets compressed and is still queryable.

#### 5.2 — OPC-UA bridge

**What**: Long-running async service `mes.integrations.opcua.OpcUaBridge` that subscribes to configured tags on one or more OPC-UA servers and ingests values.

**Design**:
- For each `equipment` with `opcua_endpoint` set, open a session with `asyncua.Client` using `Basic256Sha256` (configurable).
- Load tags from `data_point_definition` where `source_protocol='opcua'`.
- Create `subscriptions` with the configured `sample_interval_ms` and deadband.
- For each notification:
  ```python
  await event_recorder.append(
      stream_type="Equipment", stream_id=eq_id,
      event_type="SensorReadingReceived",
      payload={"tag": tag, "value": value, "quality": quality, "ts": ts.isoformat()},
      actor=System.user())
  await db.execute(insert(MachineDataPoint).values(...))
  ```
- Reconnect with exponential backoff; alert on persistent disconnects.
- Run as a Celery worker pool with one worker per OPC-UA endpoint for isolation.

**Testing**:
- Integration: spin up a `python-opcua` test server with 10 tags; bridge subscribes and ingests >=9 values in 10s.
- Integration: kill the server; bridge logs `OPCUA_DISCONNECTED` event; restart server; bridge reconnects within `backoff_max` seconds and resumes.
- Unit: deadband math: change below deadband is ignored.

#### 5.3 — MQTT / Sparkplug B subscriber

**What**: `mes.integrations.mqtt.SparkplugSubscriber` consumes Sparkplug B birth/death/data messages and maps them to `data_point_definition` and `equipment_state_log` rows.

**Design**:
- Subscribes to `spBv1.0/+/+/+/+` per Sparkplug B.
- On `NBIRTH` (Node birth): mark all node tags `online`.
- On `DBIRTH` (Device birth): register device tags if not present (auto-discovery).
- On `DDATA`: append a `SensorReadingReceived` event and insert into `machine_data_point`.
- On `DDEATH` / `NDEATH`: emit `EquipmentStateChanged` with `state='unplanned_stop', source='sparkplug'`.

**Testing**:
- Integration (EMQX Testcontainer + `tahu` publisher): publish a NBIRTH then 100 DDATA messages; verify `machine_data_point` count == 100 and that auto-discovery created the data-point definitions.

#### 5.4 — MTConnect client (CNC fleets)

**What**: HTTP polling client for MTConnect agents on CNC machines.

**Design**:
- `MtConnectClient.poll(agent_url, equipment_id)` calls `/current` and `/sample?from=...` per the spec.
- Map MTConnect DataItem IDs to `data_point_definition.source_address`.
- Emit `EquipmentStateChanged` when `Availability=AVAILABLE → UNAVAILABLE` or `ControllerMode` changes.

**Testing**:
- Fixture: vendored MTConnect XML samples; parser produces expected rows.
- Integration: a mock HTTP server simulating an MTConnect agent; client polls and ingests 60 samples in a minute.

#### 5.5 — Edge agent (store-and-forward)

**What**: A lightweight Python container `edge-agent` that runs the OPC-UA and MQTT bridges locally and buffers to SQLite if the central server is unreachable.

**Design**:
- Same bridges as the central server, but instead of writing to PostgreSQL it `POST`s batched events to `https://<central>/api/v1/edge/events`.
- On HTTP failure or 5xx, write batch to `local.db` outbox; retry every 30s with exponential backoff.
- Configured via `EDGE_*` env vars; supports identity via mTLS client certs.

**Testing**:
- Integration: block the network between edge agent and server; events queue locally; restore network; outbox drains and all events ingested in order.

---

## Phase 6: OEE and Performance Analytics (ISO 22400)

### Purpose

Convert machine state and production counts into ISO 22400-aligned OEE — availability, performance, quality — at equipment, work-centre, site, and tenant levels, with shift, hour, and day rollups. Surface OEE in real-time dashboards. This unlocks the project's headline value proposition.

### Tasks

#### 6.1 — Downtime reasons and equipment state log

**What**: Schemas and APIs for `downtime_reason` (hierarchy) and `equipment_state_log`.

**Design**: DDL per suggestion 1. Reason-tree CRUD UI. Manual downtime entry endpoint:
```
POST /api/v1/equipment/{id}/downtime
{ "started_at": "...", "ended_at": "...", "reason_id": "uuid", "notes": "..." }
```
Auto-derivation: a Celery task watches `machine_data_point` and `SensorReadingReceived` events; when `availability` tag transitions, an `equipment_state_log` row is created automatically with `source='ai_detected'` or `source='opcua'`.

**Testing**:
- Unit: overlapping `equipment_state_log` ranges for the same equipment are rejected.
- Integration: a downtime started but never ended is auto-closed at next shift boundary.

#### 6.2 — OEE calculation engine

**What**: Service that produces `oee_record` rows per (equipment, shift) and per (equipment, hour).

**Design**:
- `mes.domain.oee.calculator.OeeCalculator.compute(equipment_id, period_start, period_end)` returns:
  ```python
  @dataclass
  class OeeResult:
      planned_production_time: int   # seconds, ISO 22400 PBT
      actual_run_time: int
      planned_downtime: int
      unplanned_downtime: int
      total_count: Decimal
      good_count: Decimal
      reject_count: Decimal
      ideal_cycle_time: Decimal
      availability: Decimal          # actual_run_time / (PBT - planned_downtime)
      performance: Decimal           # (ideal_cycle_time * total_count) / actual_run_time
      quality: Decimal               # good_count / total_count
      oee: Decimal                   # availability * performance * quality
  ```
- ISO 22400 time-state classifier in `mes.core.time` maps `equipment_state_log.state` to the 11 time-states (PBT, APT, APUT, ADT, etc.).
- Celery beat schedules: every 5 min for current shift; nightly for previous day rollup.
- `GET /api/v1/oee?equipment_id=...&from=...&to=...&bucket=hour|shift|day` returns time-series JSON.

**Testing**:
- Fixture: a synthetic 8-hour shift with known states/counts; OEE matches hand-calculated values within 0.0001.
- Property-based (Hypothesis): random state sequences; OEE always in [0,1]; `availability * performance * quality == oee` within tolerance.
- Integration: live equipment data → CA refresh → API returns expected hourly buckets.

#### 6.3 — Live OEE dashboards

**What**: Executive and supervisor dashboards with OEE tiles, downtime Pareto, top-loss waterfall.

**Design**:
- `app/(supervisor)/oee/page.tsx`: site overview grid; click into work-centre → equipment.
- ECharts components: gauge (current OEE), Pareto bar (downtime reasons), waterfall (losses), heatmap (24h × 30d).
- WebSocket subscription to `/ws/oee/{equipment_id}` for sub-minute updates.
- Plant-to-plant benchmarking page for multi-site tenants.

**Testing**:
- Playwright: dashboard renders with seeded data; switching sites updates tiles.
- Vitest: Pareto component aggregates by reason category correctly.

#### 6.4 — Threshold-based alerting

**What**: `alert_rule` / `alert_instance` from suggestion 1; rules engine emits notifications by email, SMS, and webhook.

**Design**:
- Rule types: `oee_below`, `availability_below`, `downtime_minutes_exceeds`, `spc_violation`.
- Evaluation: Celery beat task every 1 min iterates active rules and queries the latest aggregate.
- Notification channels:
  - Email via SMTP (MailHog in dev).
  - SMS via Twilio (provider-pluggable).
  - Webhook: `POST <url> { rule_id, target_id, severity, message, triggered_at }` with HMAC `X-MES-Signature` header.

**Testing**:
- Integration: rule `oee_below 0.6` → seed shift OEE 0.5 → an `alert_instance` row appears and MailHog receives the email.
- Integration: webhook receiver validates the HMAC signature.

---

## Phase 7: Quality Management and SPC

### Purpose

Add the quality module — inspection plans, characteristic-level measurements, SPC charting and rule violations, non-conformance reporting, and CAPA workflow. Quality gates wire into work-order execution (already supported by the Phase 4 state machine).

### Tasks

#### 7.1 — Inspection plan and characteristic schemas

**What**: DDL per suggestion 1 "Quality Management"; APIs to create plans with characteristics and link to routing steps.

**Design**: `inspection_plan.sampling_method ∈ {all, first_piece, periodic, aql}`. AQL parameters held in JSONB `sampling_config`. Plan revision via version bump; only one `status='active'` per plan name.

**Testing**:
- Unit: variable-type characteristics require numeric limits; attribute types do not.
- Integration: linking a plan to a non-existent routing step → 422.

#### 7.2 — Inspection result capture

**What**: `POST /api/v1/work-orders/{id}/inspections` records a full inspection with per-characteristic measurements.

**Design**: Each measurement compared to limits; `result` automatically derived; `overall_result` = `fail` if any required measurement fails. `InspectionPerformed` event emitted. When `overall_result='fail'`, a `non_conformance` row is auto-created with `severity` derived from `is_critical`.

**Testing**:
- Unit: a measurement outside UCL/LCL → `result='fail'`.
- Integration: failing inspection on a critical characteristic auto-creates an NC and blocks `work_order.produce`.

#### 7.3 — SPC engine

**What**: `mes.domain.quality.spc` computes control limits (X-bar/R, Individuals/Moving Range) and evaluates Western Electric rules in real time.

**Design**:
- Library functions:
  ```python
  def compute_control_limits(samples: list[float], chart: Literal["xbar_r","imr"]) -> ControlLimits: ...
  def evaluate_we_rules(series: list[float], cl: ControlLimits) -> list[Violation]: ...
  ```
- Background task subscribes to `InspectionPerformed`; if 8+ historical points exist, recompute limits and evaluate rules; emit `SpcViolationDetected` event when triggered.
- API: `GET /api/v1/spc/{characteristic_id}?from=...&to=...` returns the chart data.

**Testing**:
- Fixture: Western Electric Rule 1 (one point outside 3σ) on a known series.
- Property-based: random in-control series produces 0 violations on average (within tolerance).
- Integration: SPC violation triggers an `alert_instance` with `severity='warning'`.

#### 7.4 — Non-conformance and CAPA workflow

**What**: Full lifecycle — open → investigating → resolved → closed; CAPA with verification step.

**Design**:
- State machines enforced server-side.
- Transitions to `closed` require an e-signature with `meaning='verified'`.
- `GET /api/v1/non-conformances?status=open` for the quality engineer worklist.
- Reports: open NCs by site, by severity, by aging bucket.

**Testing**:
- Integration: closing a CAPA without verification e-signature → 400.
- Integration: aging report buckets open NCs into `<7d`, `7-30d`, `>30d` correctly.

#### 7.5 — Quality UI

**What**: Inspection capture screen, NC dashboard, SPC charts.

**Design**: Inspection screen uses the same touch-first design as the operator panel. NC dashboard is a kanban (`open / investigating / resolved`) with drag-to-transition (with confirm e-sig modal at close).

**Testing**:
- Playwright: inspector records a failing inspection; UI immediately shows the auto-created NC.

---

## Phase 8: ERP Integration and Public REST API

### Purpose

Make the MES a first-class citizen of the enterprise: publish a versioned REST API as OpenAPI 3.1, ship B2MML / ISA-95 import/export, ship reference connectors for SAP, Oracle, Epicor, and provide webhooks for downstream systems.

### Tasks

#### 8.1 — OpenAPI hardening

**What**: Audit and refine the generated OpenAPI: tags, examples, error envelopes, security schemes, deprecation policy.

**Design**:
- Every router uses an explicit `responses=` map and `examples=` for request bodies.
- Common error model:
  ```python
  class ApiError(BaseModel):
      error: str
      code: str        # machine-readable
      message: str
      details: dict[str, Any] | None = None
      correlation_id: UUID
  ```
- Security schemes: `oauth2_client_credentials`, `oauth2_authorization_code`, `bearer`.
- Versioning: URL path `/api/v1`; `Deprecation` header policy documented.
- Redocly site served at `/docs/api`.

**Testing**:
- Unit: schemathesis fuzzing against `/openapi.json` finds zero contract violations.
- CI: lint `openapi.json` with Redocly CLI.

#### 8.2 — Webhooks

**What**: Tenant-configurable outbound webhooks for selected events.

**Design**:
- `POST /api/v1/webhooks` registers `{url, event_types: [...], secret}`.
- Worker dispatches each matching event with HMAC-SHA256 signature header `X-MES-Signature: t=<ts>,v1=<hex>`.
- Retries: 5 attempts with exponential backoff (1m, 5m, 30m, 2h, 24h); after final failure, recorded in `webhook_delivery_failure`.

**Testing**:
- Integration: register a webhook; emit matching event; receiver receives within 2s and signature verifies.
- Integration: receiver returns 500; delivery retried per schedule; final failure logged.

#### 8.3 — B2MML import/export

**What**: ISA-95 B2MML messages: `ProductionSchedule`, `ProductionPerformance`, `ProductDefinition`.

**Design**:
- XSDs vendored from MESA at `schemas/b2mml/` (Apache 2.0 compatible).
- Bindings generated with `xsdata`:
  ```bash
  xsdata generate schemas/b2mml --package mes.integrations.b2mml.bindings
  ```
- Endpoints:
  ```
  POST /api/v1/b2mml/production-schedule         (import)
  GET  /api/v1/b2mml/production-performance?...  (export)
  GET  /api/v1/b2mml/product-definition?...
  ```

**Testing**:
- Fixture: vendored sample `ProductionSchedule.xml` imports cleanly.
- Round-trip: export then import yields the same orders/quantities.

#### 8.4 — ERP reference connectors

**What**: Thin client adapters for SAP S/4HANA OData, Oracle EBS REST, and Epicor REST.

**Design**:
- `mes.integrations.erp.SapClient` wraps OData V4 with OAuth 2.0 client-credentials; methods `pull_production_orders(since)`, `push_production_performance(order_id)`.
- Each connector publishes a single CLI subcommand (`mes-cli erp sap pull`, etc.).
- Configuration via `erp_config` table; secret indirection via Vault.

**Testing**:
- Integration: mocked SAP OData server returns a sample order; connector creates a `production_order` row and emits the appropriate events.

---

## Phase 9: AI-Native Features

### Purpose

Deliver the defining capability of this MES: AI-driven OEE root-cause analysis, natural-language production reporting, adaptive scheduling, and predictive first-pass yield. These are designed as pluggable services that consume the event stream and write back recommendations.

### Tasks

#### 9.1 — Event-stream consumer framework

**What**: A reusable `EventConsumer` abstraction that lets AI services subscribe to selected event types and read replays.

**Design**:
```python
class EventConsumer:
    name: str                              # checkpoint key
    event_types: list[str]
    async def handle(self, event: Event) -> None: ...

# Implementations:
class OeeRcaConsumer(EventConsumer): ...
class YieldPredictionConsumer(EventConsumer): ...
class ScheduleRebalancerConsumer(EventConsumer): ...
```
- Consumers run as Celery workers; checkpoint persisted in `projection_checkpoint`.
- Backpressure: at-most-100 unacked events per consumer; failures retried via dead-letter queue.

**Testing**:
- Integration: stop a consumer for 5 min; restart; backlog drains and checkpoint advances monotonically.

#### 9.2 — OEE root-cause analysis

**What**: When a downtime event is recorded with `reason_id IS NULL` or `category='unplanned'`, an AI service correlates the prior 5 minutes of `machine_data_point` and `equipment_state_log` data to recommend a root cause.

**Design**:
- Feature pipeline assembles a `pandas.DataFrame` of recent sensor readings, alarms, and product changeovers.
- A trained `sklearn.ensemble.RandomForestClassifier` (trained nightly on historical labelled downtime) outputs a ranked list of probable `downtime_reason` codes with confidence.
- Recommendation written to `downtime_recommendation`:
  ```sql
  CREATE TABLE downtime_recommendation (
      id UUID PRIMARY KEY,
      equipment_state_log_id UUID NOT NULL REFERENCES equipment_state_log(id),
      ranked JSONB NOT NULL,            -- [{"reason_id":..,"confidence":0.78}, ...]
      model_version VARCHAR(50) NOT NULL,
      generated_at TIMESTAMPTZ NOT NULL DEFAULT now()
  );
  ```
- Operator UI presents top 3 with "accept" / "override"; accepted recommendations become labelled training data.
- Cold-start fallback (when <500 labelled events): rule-based classifier using alarms and tag thresholds.

**Testing**:
- Unit: feature pipeline produces deterministic feature vectors for a given event window.
- Fixture: synthetic dataset with known patterns; cross-validated F1 score > 0.7 (the test is the offline eval gate, not the inference path).
- Integration: a downtime event triggers a recommendation appearing in the operator panel within 30s.

#### 9.3 — Natural-language production reporting (LLM agent + MCP)

**What**: A conversational interface and MCP server letting supervisors and external agents ask "what happened on line 3 in the last 4 hours?" or "show me top 3 downtime causes today" in plain English.

**Design**:
- MCP tools exposed at `/api/v1/mcp`:
  ```
  get_oee_summary(equipment_id?, work_center_id?, site_id?, from, to)
  query_work_orders(filters)
  get_downtime_top_n(scope, from, to, n=5)
  get_quality_trend(characteristic_id, from, to)
  log_downtime_event(equipment_id, started_at, ended_at, reason_code, notes)
  ```
- Each tool is a thin wrapper over an existing API and respects RBAC.
- LLM provider abstracted behind `LlmClient` with backends for Anthropic, OpenAI, Azure, vLLM.
- Conversational UI at `/(supervisor)/ask` with streaming responses; cites the underlying queries.
- RAG over a pgvector embedding store of: shift reports, SOPs, recipe parameter notes, recent CAPA root-cause text.
- Prompts vendored in `mes.ai.nl_reporting.prompts.system` (reviewed and version-controlled), e.g.:
  ```
  You are the MES Shift Reporting Assistant. You answer questions strictly using the
  tools provided. Never invent KPI values. When a calculation is needed, call
  get_oee_summary or get_downtime_top_n. Cite the tool calls you made in a final
  "Sources" block.
  ```

**Testing**:
- Unit (golden): a frozen set of 20 questions and gold tool-call traces; assert the model selects the expected tool with the expected parameters.
- Integration: ask "What was OEE on Line 1 yesterday?" → response contains the value from the OEE API.
- Permission test: a user lacking `oee:read` for a site cannot retrieve OEE for that site via the MCP tool.

#### 9.4 — Adaptive scheduling

**What**: A service that, on disruptive events (breakdown, rush order, material shortage), proposes a new sequence of work orders for affected equipment.

**Design**:
- Triggers: `EquipmentStateChanged → breakdown`, `MaterialLotQuarantined`, `ProductionOrderCreated priority<=2`.
- Optimisation: `mes.ai.scheduler.solver` uses Google OR-Tools CP-SAT to minimise weighted tardiness given setup-time matrices, due dates, and priorities.
- Output: `schedule_proposal` table; UI shows side-by-side current vs proposed; supervisor accepts (writes new `work_order.planned_start/end`).

**Testing**:
- Fixture: 50 jobs across 5 machines with known optimal; solver result within 5% of optimum in <30s.
- Integration: simulate a breakdown; proposal generated and visible in scheduling UI.

#### 9.5 — Predictive first-pass yield

**What**: Model that, given current process parameters at an upstream step, predicts the probability of downstream quality escape.

**Design**:
- Trained nightly on `inspection_result` × `recipe_parameter` × `machine_data_point` joins.
- Endpoint `GET /api/v1/yield/predict?work_order_id=...` returns `{probability_pass: 0.91, top_drivers:[...]}`.
- Surfaced as a banner on the operator panel when probability drops below threshold.

**Testing**:
- Fixture: synthetic dataset with one obvious driver; model identifies it as top driver.
- Integration: parameter shift in test data → endpoint response changes accordingly.

---

## Phase 10: Compliance Packs and Multi-Site Hardening

### Purpose

Ship the 21 CFR Part 11 / EU Annex 11 validation pack and the IATF 16949 traceability pack required to sell into regulated industries; add multi-site centralised monitoring and benchmarking; add IEC 62443 SL-2 hardening.

### Tasks

#### 10.1 — 21 CFR Part 11 / EU Annex 11 compliance pack

**What**: Validation artefacts (URS, FS, DS, IQ/OQ/PQ scripts), feature gating, and the validated build pipeline.

**Design**:
- `compliance/cfr-part-11/` directory containing URS, FS, DS, traceability matrix, IQ/OQ/PQ test scripts.
- Compliance configuration:
  ```yaml
  cfr_part_11:
    enabled: true
    require_esignature_on:
      - recipe.approve
      - production_order.close
      - non_conformance.close
      - capa.verify
    audit_retention_years: 15
    password_policy:
      min_length: 12
      require_mfa: true
  ```
- A validated build is one whose Docker image digest matches a signed release manifest; verified at startup.

**Testing**:
- Unit: enabling the pack disables flows that bypass e-signatures.
- Integration: complete OQ test script (machine-runnable subset) passes in CI.

#### 10.2 — IATF 16949 traceability pack

**What**: PPAP package generation and lifelong genealogy retention.

**Design**:
- `POST /api/v1/ppap/{production_order_id}/generate` produces a PDF/A package containing PSW, control plan reference, dimensional results, material certs, and process capability indices.
- Retention: configurable per material category; cold-storage export to S3-compatible object storage with WORM lock.

**Testing**:
- Fixture: a sample order with full genealogy → generated PSW PDF matches a reference template.

#### 10.3 — Multi-site rollups

**What**: Centralised monitoring of multiple sites; plant-to-plant OEE benchmarking; alert escalation across sites.

**Design**:
- All aggregate APIs accept `site_ids[]` filter.
- New endpoints under `/api/v1/multi-site/` for benchmarking matrices and ranking.
- Edge agent supports `region` tag so the central server can group by region.

**Testing**:
- Integration: seed 3 sites with varying OEE; benchmarking endpoint ranks them correctly.

#### 10.4 — IEC 62443 SL-2 hardening

**What**: Network segmentation, encryption in transit and at rest, hardened defaults, security headers, regular security scans in CI.

**Design**:
- TLS 1.3 enforced; HSTS, CSP, Referrer-Policy headers configured.
- PostgreSQL connections require TLS in prod.
- Sensitive columns (e.g., `electronic_signature.record_hash`) encrypted at rest via `pgcrypto`.
- CI: Trivy scans images; `bandit` and `pip-audit` for Python; `gosec` not applicable.
- Penetration-test checklist in `compliance/iec-62443/`.

**Testing**:
- CI: Trivy reports zero high/critical vulnerabilities in the released image.
- Integration: HTTP request without TLS → 308 → HTTPS.

---

## Phase 11: Optional Backlog — Computer Vision, Maintenance, Energy

### Purpose

Add the high-value but optional capabilities: edge computer-vision inline inspection, full maintenance management, ISO 50001 energy tracking, MCP-driven mobile experiences.

### Tasks

#### 11.1 — Computer-vision inline inspection

**What**: Edge agent runs an ONNX-Runtime inference loop on a camera feed and emits `InspectionPerformed` events.

**Design**:
- `edge/src/vision_runtime.py` loads an ONNX model and processes frames at 5–30 fps.
- Detected defects emit a `VisionInspectionResult` event with bounding boxes and a snapshot URL (S3).
- A "model registry" API on the server tracks deployed model versions per camera.

**Testing**:
- Fixture: a small image set with known-good and known-bad examples; inference latency < 200ms on the reference Jetson.

#### 11.2 — Maintenance management

**What**: Preventive maintenance plans (suggestion-1 `maintenance_plan`, `maintenance_work_order`) and a predictive failure flag based on equipment data.

**Design**: Per suggestion 1 plus a Celery job that converts `equipment.next_due` into work-order tasks 7 days ahead.

**Testing**: Integration: a plan with `interval_days=30` creates an MWO on day 23 and again on day 53.

#### 11.3 — ISO 50001 energy KPIs

**What**: Energy meter data ingestion (a special class of `data_point_definition`) and energy-per-unit KPIs in the OEE projection.

**Testing**: Fixture: a 24h kWh stream is aggregated into per-product `energy_per_unit` correctly.

---

## Phase Summary and Dependencies

```
Phase 1: Foundation (1.1-1.5)                       required by all
   |
Phase 2: Identity, RBAC, Event Store, E-Signatures  requires 1
   |
Phase 3: Master Data (materials, products, recipes) requires 2
   |
Phase 4: Production Execution                        requires 3
   |\
   | \-- Phase 8: ERP Integration & Public API       requires 4 (parallel with 5,6,7)
   |
Phase 5: Machine Connectivity (OPC-UA/MQTT/MTC)      requires 2  (parallel with 4)
   |
Phase 6: OEE Analytics                                requires 4, 5
   |
Phase 7: Quality and SPC                              requires 4 (parallel with 6)
   |
Phase 9: AI-Native Features                           requires 6, 7
   |
Phase 10: Compliance Packs & Hardening                requires 2, 4, 6, 7, 8
   |
Phase 11: Optional Backlog (CV, Maintenance, Energy)  requires 5; CV needs 7
```

**Parallelism opportunities**:
- Phase 5 (machine connectivity) can begin as soon as Phase 2 finishes; it does not need Phase 4.
- Phases 6 and 7 can be developed concurrently once Phases 4 and 5 are merged.
- Phase 8 (ERP + public API) can begin after Phase 4 and parallel Phases 5/6/7.
- Phase 10's IEC 62443 hardening tasks (10.4) can be picked up by a separate engineer at any time after Phase 2.

---

## Definition of Done (per phase)

Every phase must satisfy all of the following before being merged:

1. All tasks in the phase implemented per the design.
2. Unit tests pass in CI; coverage for new code ≥ 85% (`pytest --cov`).
3. Integration tests pass in CI (Testcontainers spin up Postgres/Redis/EMQX as needed).
4. `ruff check`, `biome check`, and `mypy --strict` pass with zero warnings.
5. `alembic upgrade head` and `alembic downgrade -1` both succeed on a fresh DB.
6. Docker images for `server`, `worker`, `web`, and (when relevant) `edge-agent` build from `main`.
7. `docker compose up` succeeds and all healthchecks pass within 90 seconds.
8. New API endpoints appear in `/openapi.json` and pass `redocly lint`.
9. New event types have a registered JSON Schema in `schemas/events/` and an entry in `event_type_registry`.
10. New configuration options documented in `docs/configuration.md` with defaults.
11. RBAC policies for new actions added to `server/config/rbac.yml`; tests prove enforcement.
12. For phases that touch regulated functionality (2, 3, 4, 7, 10), an audit-trail integration test proves every state-changing action produces an `audit_log` row and an `event_store` row.
13. CHANGELOG.md updated under `## Unreleased` with the user-visible additions.
14. A short demo recording or screenshots attached to the merge request showing the new capability end-to-end.
