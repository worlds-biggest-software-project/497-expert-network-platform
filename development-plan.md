# Expert Network Platform — Phased Development Plan

> Project: 497-expert-network-platform · Created: 2026-05-31
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

This plan synthesises `research.md`, `features.md`, `standards.md`, `README.md`, and `data-model-suggestion-1.md` (Normalized Relational, PostgreSQL + pgvector) into an implementable, phased build. The platform is an open-source, API-first, AI-native expert network: a two-sided knowledge marketplace connecting research buyers (consulting firms, investment funds, corporate strategy, market research) with vetted domain experts, with semantic matching, automated compliance screening, call transcription/summarisation, and native MCP integration as core differentiators.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Language | Python 3.12 | The product is LLM/embedding-heavy (semantic matching, summarisation, cross-call synthesis, MCP). Python has the strongest first-class SDKs (anthropic, openai, sentence-transformers) and the official MCP Python SDK. |
| API framework | FastAPI | Async-native (needed for concurrent LLM/transcription calls), auto-generates OpenAPI 3.1 (a stated differentiator — only AlphaSense exposes a documented API in this category), and integrates Pydantic v2 for request/response validation per JSON Schema. |
| Data validation | Pydantic v2 | Single source of truth for request/response schemas, config, and JSON Schema export. |
| Database | PostgreSQL 16 + pgvector | Per `data-model-suggestion-1`: ACID transactions for compliance-critical workflows, referential integrity across the engagement lifecycle, full-text search (`tsvector`), and pgvector for sub-100ms semantic matching up to ~10M vectors — no separate vector DB needed for MVP. |
| ORM / migrations | SQLAlchemy 2.0 (async) + Alembic | Mature async ORM; Alembic gives versioned, reversible migrations for a ~30-table compliance schema. |
| Task queue | Celery + Redis | Transcription, summarisation, embedding generation, compliance screening, and calendar sync are long-running/async. Redis doubles as cache and broker. |
| Object storage | S3-compatible (MinIO for self-host, AWS S3 for cloud) | Call recordings and NDA documents are large binaries; keep them out of Postgres. |
| LLM provider | Anthropic Claude (primary) via abstraction layer | Summarisation, synthesis, AI moderation. Abstraction (`LLMClient` protocol) allows OpenAI/local fallback. Prompt caching used for repeated system prompts. |
| Embeddings | OpenAI `text-embedding-3-small` (1536-d) via same abstraction | 1536-d matches the pgvector column in the chosen data model. Swappable for a local `sentence-transformers` model in self-host mode. |
| Transcription | Provider abstraction; default Whisper (faster-whisper self-host) / Deepgram (cloud) | Self-host vs cloud parity. 99% accuracy target per proSapient benchmark; confidence scoring stored. |
| Auth | OAuth 2.0 + OpenID Connect (Authlib), JWT sessions | Standards.md mandates RFC 6749/6750 + OIDC for enterprise SSO and API access control. |
| Payments | Stripe Connect | Two-sided marketplace payouts (expert) + invoicing (client); PCI-compliant, official Python SDK. |
| Calendar | CalDAV (RFC 4791/6638) + Google/Microsoft Graph OAuth connectors | Standards-aligned scheduling with major provider coverage. |
| Video | Provider links (Zoom/Google Meet/Teams) for MVP; WebRTC roadmap | Match incumbent integration surface; avoid building media infrastructure early. |
| MCP server | Official `mcp` Python SDK | First-class, per standards.md — Guidepoint & Third Bridge use MCP; this ships it open-source. |
| Skills taxonomy | ESCO API + O*NET seed | Open, standardised skills classification (SKOS/linked-data); free ESCO public API. |
| Frontend | Next.js 14 (App Router) + TypeScript + Tailwind + shadcn/ui | Buyer/expert dashboards. Server components for fast data-dense tables. API-first means the web app is just one client of the REST API. |
| Containerisation | Docker + docker-compose | Self-hosted/hybrid deployment is a market expectation (ISO 27001 alignment). |
| Testing | pytest + pytest-asyncio + httpx + testcontainers | Async API tests; testcontainers spins real Postgres+Redis for integration tests. |
| Code quality | ruff (lint+format) + mypy (strict) | Standard, fast Python toolchain. |
| Package manager | uv | Fast, reproducible Python dependency resolution and locking. |
| CI | GitHub Actions | Lint, type-check, test matrix, Docker build, OpenAPI/JSON-Schema diff check. |

### Project Structure

```
expert-network-platform/
├── pyproject.toml
├── uv.lock
├── Dockerfile
├── docker-compose.yml              # postgres+pgvector, redis, minio, api, worker, web
├── .env.example
├── alembic.ini
├── README.md
├── migrations/                     # Alembic revisions
│   └── versions/
├── src/
│   └── enp/
│       ├── __init__.py
│       ├── main.py                 # FastAPI app factory, router registration
│       ├── config.py               # Pydantic Settings (env-driven)
│       ├── db/
│       │   ├── session.py          # async engine + session factory
│       │   ├── base.py             # DeclarativeBase, mixins (TimestampMixin)
│       │   └── models/             # SQLAlchemy ORM models (one file per domain)
│       │       ├── user.py
│       │       ├── organisation.py
│       │       ├── expert.py
│       │       ├── taxonomy.py     # skills, industries
│       │       ├── project.py
│       │       ├── engagement.py
│       │       ├── compliance.py
│       │       ├── call.py
│       │       ├── transcript.py
│       │       ├── survey.py
│       │       ├── billing.py
│       │       └── audit.py
│       ├── schemas/                # Pydantic request/response models
│       ├── api/
│       │   ├── deps.py             # auth, db session, pagination deps
│       │   ├── routers/            # one router per resource
│       │   └── errors.py           # exception handlers -> RFC 7807 problem+json
│       ├── services/               # business logic (no FastAPI imports)
│       │   ├── matching.py         # semantic matching engine
│       │   ├── compliance.py       # screening orchestration
│       │   ├── scheduling.py       # calendar + availability
│       │   ├── transcription.py
│       │   ├── summarisation.py    # per-call + cross-call synthesis
│       │   ├── billing.py
│       │   └── recommendation.py
│       ├── integrations/           # external system adapters
│       │   ├── llm/                # LLMClient protocol + anthropic/openai impls
│       │   ├── embeddings/
│       │   ├── transcription/
│       │   ├── calendar/           # caldav, google, microsoft
│       │   ├── payments/           # stripe connect
│       │   └── taxonomy/           # esco client
│       ├── workers/                # Celery tasks
│       │   ├── app.py
│       │   └── tasks.py
│       ├── auth/                   # OAuth2/OIDC, JWT, RBAC, RLS context
│       ├── mcp/                    # MCP server exposing tools/resources
│       │   └── server.py
│       └── audit/                  # audit-log writer, middleware
├── web/                            # Next.js frontend
│   ├── package.json
│   └── app/
└── tests/
    ├── conftest.py                 # fixtures: db, client, factories
    ├── unit/
    ├── integration/
    ├── e2e/
    └── fixtures/                   # sample briefs, transcripts, restricted lists
```

The structure groups by concern (models, services, integrations, api) so every phase adds files without restructuring.

---

## Phase 1: Foundation & Project Skeleton

### Purpose
Establish a runnable, tested project skeleton: configuration, database connectivity, migrations, the base ORM layer, a health endpoint, structured error handling, and CI. After this phase the application boots, connects to Postgres+Redis, and runs an empty but type-checked, linted, containerised service. Everything else builds on this.

### Tasks

#### 1.1 — Project scaffolding & tooling

**What**: Initialise the repo with uv, pyproject, ruff, mypy, pytest, Docker, and CI.

**Design**:
- `pyproject.toml` declares dependencies (fastapi, uvicorn, sqlalchemy[asyncio], asyncpg, alembic, pydantic-settings, redis, celery, authlib, pgvector, stripe, mcp, anthropic, openai, httpx) and dev deps (pytest, pytest-asyncio, mypy, ruff, testcontainers).
- `ruff.toml`: line length 100, enable `E,F,I,UP,B,ASYNC` rules. `mypy` strict mode.
- `docker-compose.yml` services: `postgres` (image `pgvector/pgvector:pg16`), `redis`, `minio`, `api`, `worker`, `web`.
- `.env.example` enumerates every env var consumed by `config.py`.

**Config model**:
```python
class Settings(BaseSettings):
    environment: Literal["dev", "test", "prod"] = "dev"
    database_url: PostgresDsn
    redis_url: RedisDsn
    jwt_secret: SecretStr
    jwt_algorithm: str = "HS256"
    access_token_ttl_minutes: int = 30
    llm_provider: Literal["anthropic", "openai"] = "anthropic"
    anthropic_api_key: SecretStr | None = None
    openai_api_key: SecretStr | None = None
    embedding_model: str = "text-embedding-3-small"
    embedding_dim: int = 1536
    s3_endpoint_url: str | None = None
    s3_bucket: str = "enp-media"
    stripe_secret_key: SecretStr | None = None
    model_config = SettingsConfigDict(env_prefix="ENP_", env_file=".env")
```

**Testing**:
- `Unit: Settings loads from env vars with ENP_ prefix → correct typed values`
- `Unit: missing required DATABASE_URL → ValidationError naming the field`
- `Unit: default values applied when optional vars absent (access_token_ttl_minutes == 30)`

#### 1.2 — Database layer & base mixins

**What**: Async SQLAlchemy engine, session dependency, declarative base, and shared mixins.

**Design**:
```python
class Base(DeclarativeBase): ...

class TimestampMixin:
    created_at: Mapped[datetime] = mapped_column(server_default=func.now())
    updated_at: Mapped[datetime] = mapped_column(server_default=func.now(), onupdate=func.now())

class UUIDPKMixin:
    id: Mapped[UUID] = mapped_column(primary_key=True, server_default=func.gen_random_uuid())
```
- `db/session.py`: `create_async_engine(settings.database_url, pool_size=10)`, `async_sessionmaker`. FastAPI dependency `get_db()` yields a session and commits/rolls back.

**Testing**:
- `Integration (testcontainers Postgres): engine connects, SELECT 1 returns 1`
- `Integration: get_db() rolls back on exception, commits on success`

#### 1.3 — Alembic migrations & pgvector bootstrap

**What**: Configure Alembic for async and create the initial migration enabling extensions.

**Design**:
- Initial migration `0001_extensions` runs `CREATE EXTENSION IF NOT EXISTS "pgcrypto"; CREATE EXTENSION IF NOT EXISTS vector; CREATE EXTENSION IF NOT EXISTS pg_trgm;`
- `alembic/env.py` imports `Base.metadata` and uses the async engine.

**Testing**:
- `Integration: alembic upgrade head on fresh DB succeeds; extensions present in pg_extension`
- `Integration: alembic downgrade base then upgrade head is idempotent`

#### 1.4 — App factory, health endpoint & error handling

**What**: FastAPI app with a health check and RFC 7807 problem+json error handlers.

**Design**:
- `GET /health` → `{"status": "ok", "db": "ok", "redis": "ok"}` (checks both, returns 503 if either down).
- Exception handlers map `AppError` subclasses → `application/problem+json` with `type`, `title`, `status`, `detail`, `instance`.
- `AppError` hierarchy: `NotFoundError(404)`, `ValidationError(422)`, `AuthError(401)`, `ForbiddenError(403)`, `ConflictError(409)`, `ComplianceBlockedError(409)`.

**Testing**:
- `Integration: GET /health with DB+Redis up → 200, all "ok"`
- `Integration: GET /health with Redis down → 503`
- `Unit: NotFoundError → problem+json body with status 404 and type URI`

### Definition of Done
App boots via `docker-compose up`, `GET /health` returns 200, `alembic upgrade head` succeeds, ruff + mypy + pytest all pass in CI.

---

## Phase 2: Identity, Organisations & RBAC

### Purpose
Implement authentication (OAuth 2.0 / OIDC + JWT), the user/organisation/membership data model, role-based access control, and per-organisation data isolation. Every subsequent resource is owned by an organisation or expert and gated by these primitives. This phase delivers the security backbone required by ISO 27001, OWASP ASVS, and the OAuth/OIDC standards.

### Tasks

#### 2.1 — User, organisation & membership models

**What**: ORM models and migration for `users`, `organisations`, `organisation_members` per data-model-suggestion-1.

**Design**: Implement tables `users`, `organisations`, `organisation_members` exactly as specified in the chosen schema (UUID PKs, CHECK constraints on `user_type`, `org_type`, `subscription_tier`, `role`; unique `(organisation_id, user_id)`). Passwords stored as argon2 hashes.

**Testing**:
- `Unit: creating user with duplicate email → IntegrityError`
- `Unit: user_type outside enum → CHECK violation`
- `Integration: cascade delete of organisation removes its members`

#### 2.2 — Registration, login & token issuance

**What**: `POST /auth/register`, `POST /auth/login`, `POST /auth/refresh`, OIDC discovery for SSO.

**Design**:
- `POST /auth/register` body `{email, password, first_name, last_name, user_type}` → creates user (`status='pending'`, sends verification), returns 201.
- `POST /auth/login` → validates argon2 hash, returns `{access_token, refresh_token, token_type:"bearer", expires_in}`. Access token JWT claims: `sub`, `user_type`, `org_ids`, `roles`, `exp`, `iat`.
- OIDC: Authlib client for enterprise IdPs; `GET /auth/oidc/{provider}/login` → redirect, `GET /auth/oidc/{provider}/callback` → upsert user, issue tokens.

**Testing**:
- `Integration: register then login → valid bearer token, decodes to correct sub`
- `Integration: login with wrong password → 401, no token`
- `Integration: expired access token on protected route → 401`
- `Integration (mocked IdP): OIDC callback with valid code → user upserted, tokens issued`

#### 2.3 — RBAC & organisation-scoped authorization

**What**: Dependency-based authorization and Postgres row-level security context.

**Design**:
- `require_roles(*roles)` and `require_org_member(org_id_param)` FastAPI dependencies read JWT claims.
- Role matrix: `owner` > `admin` > `manager` > `member`; `expert` and platform `admin` are distinct user types.
- Session-level RLS: set `SET LOCAL app.current_org = :org_id` per request; RLS policies on org-owned tables filter `organisation_id = current_setting('app.current_org')::uuid`.

**Testing**:
- `Unit: member role on admin-only route → ForbiddenError`
- `Integration: user A in org1 cannot read org2's projects (RLS blocks) → 404`
- `Integration: owner can add/remove members; member cannot`

### Definition of Done
Users can register/login/refresh; SSO via OIDC works against a mocked IdP; cross-org access is provably blocked by tests; OWASP ASVS auth checks (rate limit on login, no user enumeration in errors) pass.

---

## Phase 3: Expert Profiles & Skills Taxonomy

### Purpose
Build the expert side of the marketplace: profiles, work history, the SKOS-aligned skills/industry taxonomy seeded from ESCO, expert-skill/industry mappings, availability, and verification. This is the supply side of the network and the data the matching engine will search. After this phase experts can be onboarded and discovered via structured filters.

### Tasks

#### 3.1 — Expert profile & work history models + CRUD

**What**: `expert_profiles`, `expert_work_history` models and CRUD endpoints.

**Design**: Tables per suggestion-1. Endpoints:
- `POST /experts` (creates profile for the authenticated expert user), `GET /experts/{id}`, `PATCH /experts/{id}`, `GET /experts` (paginated, filterable).
- Profile response uses schema.org `Person` JSON-LD `@context` for the public profile representation (standards.md interoperability requirement).
- Work-history sub-resource: `POST/PATCH/DELETE /experts/{id}/work-history`.

**Testing**:
- `Integration: expert creates own profile → 201; another expert editing it → 403`
- `Unit: profile serialises with schema.org @context and Person @type`
- `Integration: list experts paginated (limit/offset) returns correct page + total count`

#### 3.2 — Skills & industry taxonomy + ESCO seeding

**What**: `skill_categories`, `skills`, `industries` models; ESCO import job.

**Design**:
- `integrations/taxonomy/esco.py`: client calls the free ESCO REST API, maps SKOS concepts → `skill_categories` (broader concepts) and `skills` (narrower), preserving `slug` and SKOS URIs.
- Management command `enp seed-taxonomy --source esco` populates the tables; O*NET CSV seed as fallback.
- `GET /taxonomy/skills?q=&category=` typeahead search using `pg_trgm` similarity.

**Testing**:
- `Integration (mocked ESCO API): seed imports N concepts → skills + categories rows with SKOS URIs`
- `Integration: GET /taxonomy/skills?q=mach → returns "Machine Learning" ranked first (trgm)`
- `Unit: duplicate skill slug on re-seed is upserted, not duplicated`

#### 3.3 — Expert-skill / expert-industry mappings & availability

**What**: `expert_skills`, `expert_industries`, `expert_availability` association management.

**Design**:
- `PUT /experts/{id}/skills` body `[{skill_id, proficiency, years}]` replaces the set transactionally.
- `PUT /experts/{id}/availability` body list of `{day_of_week, start_time, end_time, timezone}`; CHECK `end_time > start_time` enforced.
- `is_available` flag and per-slot availability both stored; matching reads both.

**Testing**:
- `Integration: set skills then GET profile → skills embedded with proficiency`
- `Unit: availability with end_time <= start_time → CHECK violation → 422`
- `Integration: replacing skills removes old associations (no orphans)`

#### 3.4 — Expert verification workflow

**What**: Verification state machine on `expert_profiles.verification_status`.

**Design**: States `unverified → pending → verified | rejected`. `POST /experts/{id}/verify` (admin only) transitions to verified/rejected with a reason; LinkedIn cross-reference is a pluggable verifier interface `Verifier.verify(profile) -> VerificationResult`.

**Testing**:
- `Unit: transition verified → pending is rejected (invalid transition)`
- `Integration: non-admin calling /verify → 403`
- `Integration: verify with method "linkedin_cross_reference" → status verified, audit entry written`

### Definition of Done
Experts can be onboarded with profiles, skills, industries, and availability; taxonomy is seeded from ESCO; structured (non-semantic) expert search works; verification workflow enforced; profiles export schema.org JSON-LD.

---

## Phase 4: Projects, Briefs & AI Semantic Matching

### Purpose
Deliver the core value proposition: AI-powered semantic matching of research briefs to expert profiles. This phase introduces projects (client research programmes), embedding generation for experts and briefs, and the matching engine that ranks experts by cosine similarity plus structured filters. This is the "heart" of the product and ships early per the plan's phase principles.

### Tasks

#### 4.1 — Project & project-skill models + CRUD

**What**: `projects`, `project_skills` models and endpoints.

**Design**: Tables per suggestion-1. `POST /projects` (org member) with `{title, description, brief, budget_cents, required_skills:[{skill_id, importance}]}`. Status state machine `draft → active → paused → completed → archived`. Org-scoped via RLS.

**Testing**:
- `Integration: create project under org → 201; appears only for that org`
- `Unit: invalid status transition (archived → active) → 422`

#### 4.2 — Embedding generation pipeline

**What**: Generate and store 1536-d embeddings for expert profiles and project briefs.

**Design**:
- `integrations/embeddings/` defines `EmbeddingClient.embed(texts: list[str]) -> list[list[float]]` with OpenAI and local impls.
- Expert embedding text = headline + bio + skills names + industries + work-history titles. Project embedding text = title + description + brief.
- `expert_embeddings`, `project_embeddings` tables (pgvector `vector(1536)`, `ivfflat` cosine index, `lists=100`).
- Celery tasks `generate_expert_embedding(expert_id)` and `generate_project_embedding(project_id)` fired on profile/project create/update (debounced).

**Testing**:
- `Integration (mocked embedding client): create expert → task enqueued → embedding row written with model_version`
- `Unit: embedding text builder concatenates fields in defined order, truncates to token budget`
- `Integration: updating profile re-generates embedding (updated_at advances)`

#### 4.3 — Semantic matching engine

**What**: `GET /projects/{id}/matches` returning ranked experts.

**Design**:
- `services/matching.py`: `match_experts(project_id, filters, top_k) -> list[MatchResult]`.
- Algorithm:
  1. Hard filters (SQL): `is_available`, `verification_status='verified'`, language, required-skill presence, exclude experts on any restricted list bound to the project's org (forward reference to Phase 6 — until then, no-op).
  2. Vector search: `ORDER BY project_embedding <=> expert_embedding LIMIT top_k*4` (cosine distance).
  3. Re-rank: blended score `0.7 * (1 - cosine_distance) + 0.2 * skill_overlap + 0.1 * normalised_rating`.
  4. Return `match_score` (0–1), `matching_skills`, `expert` summary.
- `MatchResult` Pydantic model: `{expert_id, match_score: float, matching_skills: list[UUID], rationale: str}`.

**Testing**:
- `Integration (seeded embeddings): brief about "EV batteries" ranks the battery expert above the unrelated finance expert`
- `Unit: blended score weights sum to 1.0 and produce values in [0,1]`
- `Integration: required-skill hard filter excludes experts lacking the skill`
- `Integration: top_k respected; results sorted descending by match_score`

#### 4.4 — Knowledge-gap hint (lightweight)

**What**: Flag required project skills with no/low matching experts.

**Design**: `GET /projects/{id}/gaps` → for each required skill, count verified experts with that skill and an embedding similarity above a threshold; return skills below a coverage floor as gaps.

**Testing**:
- `Integration: project requiring a skill no expert has → that skill returned as a gap`

### Definition of Done
A client can post a brief and receive a ranked, explainable list of matching experts in sub-second time on a seeded dataset; embeddings regenerate on changes; hard filters and blended re-ranking are tested.

---

## Phase 5: Engagements & Compliance Screening

### Purpose
Implement the regulated heart of the workflow: requesting an engagement, then driving it through automated compliance screening (NDA, conflict-of-interest, MNPI acknowledgement, restricted-list check, GDPR consent) before any call can be scheduled. Compliance is treated as a core architectural concern per SEC Reg FD, Advisers Act §204A, and CFA Institute standards. Nothing proceeds to scheduling without a clear compliance gate.

### Tasks

#### 5.1 — Engagement model & lifecycle

**What**: `engagements` model with enforced state machine.

**Design**: Table per suggestion-1. States: `requested → expert_invited → accepted → compliance_pending → compliance_cleared → scheduled → completed`, plus terminal `declined`, `cancelled`. `POST /projects/{id}/engagements` `{expert_id, hourly_rate_cents}` creates a `requested` engagement, recording `match_score` from the matcher. Transition guard function rejects illegal jumps and writes an audit entry per transition.

**Testing**:
- `Unit: requested → scheduled (skipping compliance) → rejected`
- `Integration: duplicate active engagement for same (project, expert) → 409`
- `Integration: each transition writes an audit_log row`

#### 5.2 — Restricted lists & entries

**What**: `restricted_lists`, `restricted_list_entries` models + CRUD (org-scoped).

**Design**: Tables per suggestion-1. Entries typed `company | person | ticker | topic` with `valid_from`/`valid_until`. `POST /orgs/{id}/restricted-lists`, `POST .../entries`. Matching against expert work-history companies and profile uses `pg_trgm` fuzzy + exact ticker match.

**Testing**:
- `Integration: add entry with valid_until in past → not active in screening`
- `Unit: ticker entry matches expert work-history company by exact ticker`

#### 5.3 — Compliance screening orchestration

**What**: `services/compliance.py` running all required checks, producing `compliance_checks` rows.

**Design**:
- `compliance_checks` table per suggestion-1 (`check_type`, `status`, `automated`, `details`, `expires_at`).
- `run_compliance(engagement_id) -> ComplianceResult` orchestrates in parallel:
  - `restricted_list`: automated — fuzzy-match expert against active restricted-list entries of the project's org. Match → `failed` with details.
  - `conflict_of_interest`: automated — heuristic + optional LLM check comparing expert's current employer/affiliations against the project's subject companies.
  - `mnpi_acknowledgement`: requires expert action (acknowledge MNPI policy) → starts `pending`.
  - `nda`: generates NDA doc (Phase 5.4) → `pending` until both parties sign.
  - `gdpr_consent`: verify expert consented to contact/data retention.
- When all required checks reach `passed`/`waived`, emit `EngagementComplianceCleared`, transition engagement to `compliance_cleared`.
- Endpoints: `POST /engagements/{id}/compliance/run`, `GET /engagements/{id}/compliance`, `POST /engagements/{id}/compliance/{check_id}/acknowledge|waive`.

**Testing**:
- `Integration (fixture restricted list): expert whose former employer is restricted → restricted_list check FAILS, engagement blocked`
- `Integration: clean expert → all automated checks pass, MNPI+NDA pending`
- `Integration: scheduling a call before compliance_cleared → ComplianceBlockedError 409`
- `Unit: waive on a check (admin) → status waived, recorded with performed_by`
- `Integration (mocked LLM): COI check returns conflict for matching employer → failed with rationale`

#### 5.4 — NDA generation & e-sign tracking

**What**: Generate NDA documents and track dual-party signatures.

**Design**: `nda_documents` table per suggestion-1. `generate_nda(engagement)` renders a templated PDF (jurisdiction-aware template), stores in S3, returns `document_url`. `POST /engagements/{id}/nda/sign` marks `signed_by_expert`/`signed_by_client`; when both signed, the `nda` compliance check flips to `passed`.

**Testing**:
- `Integration: generate NDA → PDF stored, url returned, nda check pending`
- `Integration: both parties sign → nda check passed; only one signs → still pending`

### Definition of Done
An engagement cannot reach `scheduled` without a cleared compliance gate; restricted-list matches block engagement; NDA/MNPI flows tracked; every transition and check is auditable (satisfying Reg FD / §204A record-keeping). The matching engine's restricted-list hard filter (Phase 4.3) is now wired to real lists.

---

## Phase 6: Scheduling, Calendar Integration & Calls

### Purpose
Turn a compliance-cleared engagement into a scheduled, conducted call. Implements availability-aware scheduling, CalDAV/Google/Microsoft calendar sync, video-provider link generation, automated reminders, and the call lifecycle through completion. After this phase the platform supports the full live-consultation workflow.

### Tasks

#### 6.1 — Call model & scheduling

**What**: `calls` model and scheduling endpoint with conflict detection.

**Design**: Table per suggestion-1. `POST /engagements/{id}/calls` `{scheduled_start, scheduled_end, meeting_provider}` validates: engagement `compliance_cleared`; slot within expert availability; no overlap with expert's other scheduled calls. State machine `scheduled → in_progress → completed`, plus `cancelled`, `no_show`. Generates a provider meeting link via the chosen provider connector.

**Testing**:
- `Integration: schedule within availability → 201; outside availability → 422`
- `Integration: overlapping call for same expert → 409 conflict`
- `Integration: schedule on non-cleared engagement → ComplianceBlockedError 409`

#### 6.2 — Calendar sync connectors

**What**: `calendar_sync` model + CalDAV/Google/Microsoft adapters.

**Design**: `calendar_sync` table per suggestion-1 with encrypted token columns. `integrations/calendar/` defines `CalendarProvider.create_event / update_event / delete_event`. CalDAV impl uses RFC 4791/6638; Google/Microsoft via Graph OAuth. On call schedule, an event is pushed to both participants' connected calendars (best-effort, async).

**Testing**:
- `Integration (mocked Google API): scheduling a call creates calendar events for both connected users`
- `Unit: tokens stored encrypted, never returned in API responses`
- `Integration: calendar push failure does not roll back the call (logged, retried)`

#### 6.3 — Automated reminders & no-show handling

**What**: Celery beat reminders and no-show detection.

**Design**: Periodic task enqueues reminders at T-24h and T-1h (email/notification). If `actual_start` is null 15 min past `scheduled_start`, mark `no_show` and emit event; record against expert no-show metric (addresses the industry's persistent no-show complaint).

**Testing**:
- `Integration: call 1h away → reminder task scheduled`
- `Integration: no actual_start 15min after start → status no_show, metric incremented`

#### 6.4 — Call lifecycle endpoints & recording handoff

**What**: Start/complete endpoints and recording ingestion.

**Design**: `POST /calls/{id}/start` sets `actual_start`, status `in_progress`. `POST /calls/{id}/complete` sets `actual_end`, `duration_minutes`, accepts `recording_url`, transitions to `completed`, and triggers async transcription (Phase 7). `POST /calls/{id}/recording` accepts an uploaded recording to S3.

**Testing**:
- `Integration: complete call → status completed, transcription task enqueued`
- `Unit: duration_minutes computed from actual_start/end`

### Definition of Done
Compliance-cleared engagements can be scheduled with conflict-free, availability-aware slots; calendar events sync to Google/Microsoft/CalDAV; reminders fire; no-shows are detected; completing a call hands off to transcription.

---

## Phase 7: Transcription, Summarisation & Transcript Library

### Purpose
Convert every completed call into searchable, citable research artefacts — the value-add that eliminates re-sourcing costs. Implements transcription, structured AI summaries (Expert Background / Sentiment / Key Takeaways), and a full-text-searchable transcript library. This realises the "every call becomes a research artefact" promise.

### Tasks

#### 7.1 — Transcription pipeline

**What**: `transcripts` model + async transcription task.

**Design**: Table per suggestion-1 (`processing_status: pending → processing → completed | failed`). `integrations/transcription/` defines `Transcriber.transcribe(audio_url) -> TranscriptResult{text, language, confidence}`. Celery task `transcribe_call(call_id)` downloads recording from S3, transcribes, stores `raw_text`, `word_count`, `confidence_score`, builds the `tsvector` FTS index.

**Testing**:
- `Integration (mocked transcriber, fixture audio): completed call → transcript row, status completed, word_count > 0`
- `Integration: transcriber failure → status failed, retried up to N times`

#### 7.2 — Structured AI summarisation

**What**: `transcript_summaries` model + per-call summary generation.

**Design**: Table per suggestion-1 (`summary_type: executive | expert_background | sentiment | key_takeaways | thematic`). Celery task `summarise_transcript(transcript_id)` calls the LLM once per summary type using cached system prompts.

System prompt template (per type), e.g. key_takeaways:
```
You are a research analyst producing decision-ready summaries of expert consultation
transcripts. Output ONLY the requested section. Be concise, cite specifics, do not invent.

Section: KEY TAKEAWAYS — 3 to 7 bullet points capturing the most decision-relevant
insights, each ≤ 30 words. Preserve figures, company names, and timeframes verbatim.
```
User message = transcript `raw_text`. `model_version` recorded for provenance.

**Testing**:
- `Integration (mocked LLM): transcript → 4 summary rows (executive, expert_background, sentiment, key_takeaways) with model_version`
- `Unit: summary prompt builder selects correct system prompt per summary_type`
- `Integration: summary generation idempotent — re-run replaces prior summary of same type`

#### 7.3 — Transcript library search

**What**: Full-text + filtered search over the transcript library.

**Design**: `GET /transcripts?q=&project_id=&expert_id=&from=&to=` uses Postgres `to_tsvector`/`websearch_to_tsquery`, returns ranked snippets with `ts_headline` highlighting, scoped to the requesting org by RLS. Each hit links to call, expert, project, and structured summaries.

**Testing**:
- `Integration: search "lithium supply" → returns the battery transcript with highlighted snippet`
- `Integration: org isolation — user cannot find another org's transcripts`
- `Integration: filter by date range narrows results correctly`

#### 7.4 — Call ratings feeding reputation

**What**: `call_ratings` model + endpoints; update expert `avg_rating`.

**Design**: Table per suggestion-1 (unique `(call_id, rated_by)`, ratings 1–5, plus expertise/communication sub-scores). `POST /calls/{id}/rating` (participant only, completed call). On insert, recompute `expert_profiles.avg_rating` and `total_calls` in a transaction.

**Testing**:
- `Integration: rate completed call → avg_rating recomputed`
- `Integration: rating an uncompleted call or as non-participant → 403/409`
- `Unit: duplicate rating by same user → 409`

### Definition of Done
Completed calls auto-transcribe and auto-summarise into structured sections; the org's transcript library is full-text searchable with highlighting and isolation; ratings update expert reputation used by the matcher.

---

## Phase 8: Billing, Payouts & Invoicing

### Purpose
Close the marketplace loop with transparent pay-as-you-go billing for clients and payouts for experts via Stripe Connect, plus invoicing. Transparent dollar pricing is a stated differentiator versus opaque credit systems.

### Tasks

#### 8.1 — Stripe Connect onboarding

**What**: Connect expert and organisation accounts to Stripe.

**Design**: `integrations/payments/stripe_connect.py`. Experts onboard as connected accounts (`POST /experts/{id}/payments/onboard` → Stripe onboarding link). Organisations attach a payment method/customer. Webhook endpoint `POST /webhooks/stripe` verifies signatures and updates account status.

**Testing**:
- `Integration (mocked Stripe): onboard expert → connected account id stored`
- `Integration: webhook with invalid signature → 401, ignored`
- `Integration: webhook account.updated → expert payout-eligible flag set`

#### 8.2 — Invoices & line items

**What**: `invoices`, `invoice_line_items` models + generation from completed calls.

**Design**: Tables per suggestion-1. On call completion (or monthly batch), generate line items priced at engagement `hourly_rate_cents * duration`. `invoice_number` sequential per org. Status `draft → issued → paid → overdue | cancelled | refunded`.

**Testing**:
- `Integration: completed 60-min call at 500/hr → line item 50000 cents`
- `Unit: invoice total = sum(line items) + tax`
- `Integration: invoice_number unique and monotonic per org`

#### 8.3 — Expert payouts

**What**: `expert_payouts` model + payout execution.

**Design**: Table per suggestion-1. Payout = call charge minus platform fee. Celery task `process_payout(payout_id)` calls Stripe transfer to connected account; status `pending → processing → paid | failed`.

**Testing**:
- `Integration (mocked Stripe): process payout → transfer created, status paid`
- `Integration: payout to non-onboarded expert → failed with reason`
- `Unit: payout amount = charge - platform_fee_pct`

### Definition of Done
Completed calls generate invoices and expert payouts; Stripe Connect onboarding and webhooks work against mocked Stripe; amounts and fees are correct and tested; pricing is transparent (dollar amounts, no credits).

---

## Phase 9: Cross-Call Synthesis & Recommendation Engine

### Purpose
Deliver the higher-order AI differentiators: synthesising themes/contradictions/consensus across multiple calls on a project, and a recommendation engine that learns from past ratings and engagement patterns to surface the best experts for new briefs without repeating search.

### Tasks

#### 9.1 — Cross-call synthesis

**What**: `synthesis_reports`, `synthesis_report_calls` models + generation.

**Design**: Tables per suggestion-1. `POST /projects/{id}/synthesis` selects N completed calls' transcripts/summaries, calls the LLM to produce a thematic report (consensus, contradictions, open questions), stores it with `model_version` and the source call set.

Synthesis prompt structure:
```
SYSTEM: You synthesise multiple expert consultation summaries for one research project.
Identify: (1) points of consensus, (2) contradictions between experts, (3) open questions /
knowledge gaps. Attribute claims to expert pseudonyms (Expert A/B/...). Do not fabricate.
USER: <project brief> + <ordered list of per-call key_takeaways + sentiment summaries>
```

**Testing**:
- `Integration (mocked LLM, 3 fixture transcripts): synthesis report created, links all 3 calls`
- `Unit: synthesis input builder includes only completed calls with summaries`

#### 9.2 — Recommendation engine

**What**: `services/recommendation.py` learning from history.

**Design**: `recommend_experts(brief_or_project, org_id, top_k)` blends: semantic match (Phase 4) + collaborative signal (experts highly rated by this org or on similar past projects) + recency/availability. Similar-project similarity via project embeddings. Returns ranked experts with rationale distinguishing "semantic" vs "proven performer for you".

**Testing**:
- `Integration: expert rated 5 on a prior similar project ranks above an equally-semantic-matched but unrated expert`
- `Unit: blend never recommends experts on the org's restricted list`

### Definition of Done
Multi-call projects produce synthesis reports identifying consensus/contradiction/gaps; recommendations measurably prefer proven, highly-rated experts for similar briefs while respecting compliance filters.

---

## Phase 10: MCP Server & OpenAPI/SDK Surface

### Purpose
Ship the open, AI-native integration surface that defines the project's market position: a Model Context Protocol server (matching Guidepoint/Third Bridge) plus a polished, documented OpenAPI 3.1 spec and a generated client SDK — the API-first differentiator no incumbent except AlphaSense offers.

### Tasks

#### 10.1 — MCP server

**What**: MCP server exposing expert search, transcript retrieval, and synthesis as tools/resources.

**Design**: `mcp/server.py` using the official `mcp` SDK. Tools: `search_experts(brief, filters)`, `get_transcript(call_id)`, `search_transcripts(query)`, `get_synthesis(project_id)`. Resources expose transcript summaries. Auth via OAuth 2.0 bearer per MCP 2026 roadmap (SSO-integrated, audit-logged); every tool call writes an `audit_log` entry. All access is org-scoped.

**Testing**:
- `Integration (MCP client harness): search_experts tool returns matches for a brief`
- `Integration: get_transcript for another org's call → denied`
- `Integration: every MCP tool invocation produces an audit_log row`

#### 10.2 — OpenAPI 3.1 spec & client SDK

**What**: Curate the auto-generated OpenAPI 3.1 document and generate a Python/TypeScript client.

**Design**: Tag/organise routers, add examples and security schemes (OAuth2 + bearer) to the FastAPI app so `/openapi.json` is publication-quality. CI step generates clients via `openapi-generator` and fails if the committed spec drifts from the live app.

**Testing**:
- `Integration: GET /openapi.json validates against OpenAPI 3.1 meta-schema`
- `CI: generated client compiles; spec drift check fails on uncommitted change`

#### 10.3 — Surveys to expert panels

**What**: `surveys`, `survey_questions`, `survey_question_options`, `survey_responses`, `survey_answers` models + endpoints.

**Design**: Tables per suggestion-1. Create/deploy surveys to matched expert panels; experts submit responses; results aggregate per question. (Grouped here as the remaining v1.1 feature; independent of MCP/SDK and can parallel them.)

**Testing**:
- `Integration: create survey, deploy to 3 experts, collect responses → aggregation correct`
- `Unit: required question unanswered → submission rejected 422`

### Definition of Done
External AI agents (Claude/ChatGPT) can query the platform via MCP with org-scoped, audited access; the OpenAPI 3.1 spec is validated and published; a generated SDK compiles; surveys can be deployed and aggregated.

---

## Phase Summary & Dependencies

```
Phase 1: Foundation & Skeleton        ─── required by everything
    │
Phase 2: Identity, Orgs & RBAC        ─── requires 1
    │
Phase 3: Expert Profiles & Taxonomy   ─── requires 2
    │
Phase 4: Projects & Semantic Matching ─── requires 3   (CORE VALUE PROP)
    │
Phase 5: Engagements & Compliance     ─── requires 4   (wires restricted-list filter back into 4.3)
    │
Phase 6: Scheduling & Calls           ─── requires 5
    │
Phase 7: Transcription & Summaries    ─── requires 6
    ├── Phase 8: Billing & Payouts     ─── requires 6 (call completion); can parallel with 7
    │
Phase 9: Synthesis & Recommendation   ─── requires 7 (synthesis) + 4 (recommendation)
    │
Phase 10: MCP / OpenAPI / Surveys     ─── MCP+SDK require 4/7; Surveys require 3
                                           10.1, 10.2, 10.3 can all parallel each other
```

Parallelism opportunities:
- **Phases 7 and 8** can be developed concurrently after Phase 6 (both depend only on call completion).
- Within **Phase 10**, the MCP server (10.1), OpenAPI/SDK (10.2), and Surveys (10.3) are independent.
- The **Next.js frontend** (`web/`) can be built incrementally against the OpenAPI spec starting after Phase 2, in parallel with backend phases.
- ESCO taxonomy seeding (3.2) is independent of profile CRUD (3.1) once models exist.

---

## Definition of Done (per phase)

Every phase must satisfy all of the following before it is considered complete:

1. All tasks in the phase implemented.
2. All unit and integration tests pass (`pytest`), including the phase's named scenarios.
3. Linting and formatting pass (`ruff check`, `ruff format --check`).
4. Type checking passes (`mypy --strict`).
5. Alembic migration(s) created, and `upgrade head` / `downgrade` verified on a fresh DB.
6. Docker build succeeds and `docker-compose up` boots the affected services.
7. The phase's user-facing capability works end-to-end (demonstrated by an e2e or integration test).
8. New config options added to `.env.example` and documented.
9. New API endpoints appear in `/openapi.json` with correct security schemes and examples.
10. Compliance- and security-relevant changes write `audit_log` entries and respect org-scoped RLS (OWASP ASVS, Reg FD / §204A record-keeping).
11. External integrations (LLM, embeddings, transcription, Stripe, calendar, ESCO, MCP) are exercised via mocks in CI; real-dependency tests exist but are marked optional.
```
