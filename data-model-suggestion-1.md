# Data Model Suggestion 1: Normalized Relational Model (PostgreSQL)

## Approach

A traditional third-normal-form (3NF+) relational schema in PostgreSQL. Every entity occupies its own table with strict foreign-key constraints, composite indexes for common query patterns, and CHECK constraints for domain invariants. This is the most battle-tested approach for transactional systems with complex compliance requirements.

## Why This Suits an Expert Network Platform

Expert network platforms are fundamentally transactional: clients create projects, request expert engagements, schedule calls, record compliance acknowledgements, and process payments. Each of these operations must be atomic, auditable, and consistent. A normalized relational model provides:

- **Referential integrity** across the full lifecycle (expert profile -> compliance check -> engagement -> call -> transcript -> payment).
- **ACID transactions** for compliance-critical operations like MNPI acknowledgement and restricted-list screening.
- **Mature tooling** for audit trails, row-level security, and regulatory reporting.
- **Straightforward migration paths** as the schema evolves.

The main trade-off is that complex many-to-many relationships (skills taxonomies, cross-call synthesis) require join-heavy queries, and deeply nested profile data can be awkward in flat relational tables.

---

## Schema Definition

```sql
-- ============================================================
-- USERS AND AUTHENTICATION
-- ============================================================

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email           VARCHAR(255) NOT NULL UNIQUE,
    password_hash   VARCHAR(255) NOT NULL,
    first_name      VARCHAR(100) NOT NULL,
    last_name       VARCHAR(100) NOT NULL,
    phone           VARCHAR(30),
    avatar_url      TEXT,
    user_type       VARCHAR(20) NOT NULL CHECK (user_type IN ('expert', 'client', 'admin')),
    status          VARCHAR(20) NOT NULL DEFAULT 'pending' CHECK (status IN ('pending', 'active', 'suspended', 'deactivated')),
    email_verified  BOOLEAN NOT NULL DEFAULT FALSE,
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_users_email ON users (email);
CREATE INDEX idx_users_type_status ON users (user_type, status);

-- ============================================================
-- ORGANISATIONS (client companies, consulting firms, funds)
-- ============================================================

CREATE TABLE organisations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(255) NOT NULL UNIQUE,
    domain          VARCHAR(255),
    org_type        VARCHAR(30) NOT NULL CHECK (org_type IN ('consulting', 'investment_fund', 'corporate', 'research', 'other')),
    subscription_tier VARCHAR(20) NOT NULL DEFAULT 'payg' CHECK (subscription_tier IN ('payg', 'starter', 'professional', 'enterprise')),
    billing_email   VARCHAR(255),
    logo_url        TEXT,
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE organisation_members (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role            VARCHAR(20) NOT NULL DEFAULT 'member' CHECK (role IN ('owner', 'admin', 'manager', 'member')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organisation_id, user_id)
);

CREATE INDEX idx_org_members_org ON organisation_members (organisation_id);
CREATE INDEX idx_org_members_user ON organisation_members (user_id);

-- ============================================================
-- EXPERT PROFILES
-- ============================================================

CREATE TABLE expert_profiles (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL UNIQUE REFERENCES users(id) ON DELETE CASCADE,
    headline        VARCHAR(300),
    bio             TEXT,
    hourly_rate_cents INTEGER NOT NULL DEFAULT 0,
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    years_experience INTEGER,
    linkedin_url    TEXT,
    timezone        VARCHAR(50) NOT NULL DEFAULT 'UTC',
    languages       TEXT[], -- e.g. {'en','fr','de'}
    is_available     BOOLEAN NOT NULL DEFAULT TRUE,
    avg_rating      NUMERIC(3,2) DEFAULT 0.00,
    total_calls     INTEGER NOT NULL DEFAULT 0,
    total_earnings_cents BIGINT NOT NULL DEFAULT 0,
    verification_status VARCHAR(20) NOT NULL DEFAULT 'unverified' CHECK (verification_status IN ('unverified', 'pending', 'verified', 'rejected')),
    onboarded_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_expert_profiles_rate ON expert_profiles (hourly_rate_cents);
CREATE INDEX idx_expert_profiles_rating ON expert_profiles (avg_rating DESC);
CREATE INDEX idx_expert_profiles_available ON expert_profiles (is_available) WHERE is_available = TRUE;

-- ============================================================
-- SKILLS TAXONOMY
-- ============================================================

CREATE TABLE skill_categories (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(100) NOT NULL UNIQUE,
    parent_id       UUID REFERENCES skill_categories(id),
    description     TEXT,
    sort_order      INTEGER NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE skills (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    category_id     UUID NOT NULL REFERENCES skill_categories(id),
    name            VARCHAR(150) NOT NULL,
    slug            VARCHAR(150) NOT NULL UNIQUE,
    description     TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_skills_category ON skills (category_id);

CREATE TABLE expert_skills (
    expert_id       UUID NOT NULL REFERENCES expert_profiles(id) ON DELETE CASCADE,
    skill_id        UUID NOT NULL REFERENCES skills(id) ON DELETE CASCADE,
    proficiency     VARCHAR(20) DEFAULT 'intermediate' CHECK (proficiency IN ('beginner', 'intermediate', 'advanced', 'expert')),
    years           INTEGER,
    PRIMARY KEY (expert_id, skill_id)
);

-- ============================================================
-- INDUSTRIES AND COVERAGE
-- ============================================================

CREATE TABLE industries (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(150) NOT NULL UNIQUE,
    parent_id       UUID REFERENCES industries(id),
    sic_code        VARCHAR(10),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE expert_industries (
    expert_id       UUID NOT NULL REFERENCES expert_profiles(id) ON DELETE CASCADE,
    industry_id     UUID NOT NULL REFERENCES industries(id) ON DELETE CASCADE,
    is_primary      BOOLEAN NOT NULL DEFAULT FALSE,
    PRIMARY KEY (expert_id, industry_id)
);

-- ============================================================
-- WORK HISTORY (for expert profiles)
-- ============================================================

CREATE TABLE expert_work_history (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    expert_id       UUID NOT NULL REFERENCES expert_profiles(id) ON DELETE CASCADE,
    company_name    VARCHAR(255) NOT NULL,
    title           VARCHAR(255) NOT NULL,
    start_date      DATE NOT NULL,
    end_date        DATE,
    is_current      BOOLEAN NOT NULL DEFAULT FALSE,
    description     TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_work_history_expert ON expert_work_history (expert_id);

-- ============================================================
-- PROJECTS (client research programmes)
-- ============================================================

CREATE TABLE projects (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    created_by      UUID NOT NULL REFERENCES users(id),
    title           VARCHAR(300) NOT NULL,
    description     TEXT,
    brief           TEXT, -- the research brief for AI matching
    status          VARCHAR(20) NOT NULL DEFAULT 'draft' CHECK (status IN ('draft', 'active', 'paused', 'completed', 'archived')),
    budget_cents    BIGINT,
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    start_date      DATE,
    end_date        DATE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_projects_org ON projects (organisation_id);
CREATE INDEX idx_projects_status ON projects (status);

CREATE TABLE project_skills (
    project_id      UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    skill_id        UUID NOT NULL REFERENCES skills(id) ON DELETE CASCADE,
    importance      VARCHAR(10) DEFAULT 'required' CHECK (importance IN ('required', 'preferred', 'nice_to_have')),
    PRIMARY KEY (project_id, skill_id)
);

-- ============================================================
-- ENGAGEMENTS (expert <-> project matchings)
-- ============================================================

CREATE TABLE engagements (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES projects(id),
    expert_id       UUID NOT NULL REFERENCES expert_profiles(id),
    requested_by    UUID NOT NULL REFERENCES users(id),
    status          VARCHAR(30) NOT NULL DEFAULT 'requested'
        CHECK (status IN ('requested', 'expert_invited', 'accepted', 'compliance_pending',
                          'compliance_cleared', 'scheduled', 'completed', 'declined', 'cancelled')),
    match_score     NUMERIC(5,4), -- AI-generated semantic match score
    hourly_rate_cents INTEGER NOT NULL,
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_engagements_project ON engagements (project_id);
CREATE INDEX idx_engagements_expert ON engagements (expert_id);
CREATE INDEX idx_engagements_status ON engagements (status);

-- ============================================================
-- COMPLIANCE
-- ============================================================

CREATE TABLE compliance_checks (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    engagement_id   UUID NOT NULL REFERENCES engagements(id),
    check_type      VARCHAR(30) NOT NULL CHECK (check_type IN ('nda', 'conflict_of_interest', 'mnpi_acknowledgement', 'restricted_list', 'gdpr_consent')),
    status          VARCHAR(20) NOT NULL DEFAULT 'pending' CHECK (status IN ('pending', 'passed', 'failed', 'waived', 'expired')),
    performed_by    UUID REFERENCES users(id), -- NULL if automated
    automated       BOOLEAN NOT NULL DEFAULT TRUE,
    details         TEXT,
    checked_at      TIMESTAMPTZ,
    expires_at      TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_compliance_engagement ON compliance_checks (engagement_id);
CREATE INDEX idx_compliance_status ON compliance_checks (status);

CREATE TABLE restricted_lists (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE restricted_list_entries (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    restricted_list_id UUID NOT NULL REFERENCES restricted_lists(id) ON DELETE CASCADE,
    entity_type     VARCHAR(20) NOT NULL CHECK (entity_type IN ('company', 'person', 'ticker', 'topic')),
    entity_name     VARCHAR(255) NOT NULL,
    reason          TEXT,
    added_by        UUID REFERENCES users(id),
    valid_from      DATE NOT NULL DEFAULT CURRENT_DATE,
    valid_until     DATE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_restricted_entries_list ON restricted_list_entries (restricted_list_id);
CREATE INDEX idx_restricted_entries_entity ON restricted_list_entries (entity_type, entity_name);

CREATE TABLE nda_documents (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    engagement_id   UUID NOT NULL REFERENCES engagements(id),
    document_url    TEXT NOT NULL,
    signed_by_expert BOOLEAN NOT NULL DEFAULT FALSE,
    signed_by_client BOOLEAN NOT NULL DEFAULT FALSE,
    expert_signed_at TIMESTAMPTZ,
    client_signed_at TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- CALLS / CONSULTATIONS
-- ============================================================

CREATE TABLE calls (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    engagement_id   UUID NOT NULL REFERENCES engagements(id),
    scheduled_start TIMESTAMPTZ NOT NULL,
    scheduled_end   TIMESTAMPTZ NOT NULL,
    actual_start    TIMESTAMPTZ,
    actual_end      TIMESTAMPTZ,
    duration_minutes INTEGER,
    status          VARCHAR(20) NOT NULL DEFAULT 'scheduled'
        CHECK (status IN ('scheduled', 'in_progress', 'completed', 'cancelled', 'no_show')),
    meeting_provider VARCHAR(20) CHECK (meeting_provider IN ('zoom', 'google_meet', 'teams', 'phone')),
    meeting_url     TEXT,
    meeting_id      VARCHAR(255),
    recording_url   TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_calls_engagement ON calls (engagement_id);
CREATE INDEX idx_calls_scheduled ON calls (scheduled_start);
CREATE INDEX idx_calls_status ON calls (status);

-- ============================================================
-- TRANSCRIPTS AND SUMMARIES
-- ============================================================

CREATE TABLE transcripts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    call_id         UUID NOT NULL UNIQUE REFERENCES calls(id),
    raw_text        TEXT NOT NULL,
    language        CHAR(5) NOT NULL DEFAULT 'en',
    word_count      INTEGER,
    confidence_score NUMERIC(4,3),
    processing_status VARCHAR(20) NOT NULL DEFAULT 'pending'
        CHECK (processing_status IN ('pending', 'processing', 'completed', 'failed')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE transcript_summaries (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    transcript_id   UUID NOT NULL REFERENCES transcripts(id) ON DELETE CASCADE,
    summary_type    VARCHAR(30) NOT NULL CHECK (summary_type IN ('executive', 'expert_background', 'sentiment', 'key_takeaways', 'thematic')),
    content         TEXT NOT NULL,
    model_version   VARCHAR(50), -- which AI model produced this
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_summaries_transcript ON transcript_summaries (transcript_id);

-- Full-text search index on transcripts
CREATE INDEX idx_transcripts_fts ON transcripts USING gin(to_tsvector('english', raw_text));

-- ============================================================
-- CROSS-CALL SYNTHESIS (thematic analysis across calls)
-- ============================================================

CREATE TABLE synthesis_reports (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES projects(id),
    title           VARCHAR(300) NOT NULL,
    content         TEXT NOT NULL,
    model_version   VARCHAR(50),
    created_by      UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE synthesis_report_calls (
    synthesis_id    UUID NOT NULL REFERENCES synthesis_reports(id) ON DELETE CASCADE,
    call_id         UUID NOT NULL REFERENCES calls(id) ON DELETE CASCADE,
    PRIMARY KEY (synthesis_id, call_id)
);

-- ============================================================
-- SURVEYS
-- ============================================================

CREATE TABLE surveys (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES projects(id),
    created_by      UUID NOT NULL REFERENCES users(id),
    title           VARCHAR(300) NOT NULL,
    description     TEXT,
    status          VARCHAR(20) NOT NULL DEFAULT 'draft' CHECK (status IN ('draft', 'active', 'closed', 'archived')),
    opens_at        TIMESTAMPTZ,
    closes_at       TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE survey_questions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    survey_id       UUID NOT NULL REFERENCES surveys(id) ON DELETE CASCADE,
    question_text   TEXT NOT NULL,
    question_type   VARCHAR(20) NOT NULL CHECK (question_type IN ('text', 'single_choice', 'multi_choice', 'rating', 'ranking')),
    sort_order      INTEGER NOT NULL DEFAULT 0,
    is_required     BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE survey_question_options (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    question_id     UUID NOT NULL REFERENCES survey_questions(id) ON DELETE CASCADE,
    option_text     VARCHAR(500) NOT NULL,
    sort_order      INTEGER NOT NULL DEFAULT 0
);

CREATE TABLE survey_responses (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    survey_id       UUID NOT NULL REFERENCES surveys(id),
    expert_id       UUID NOT NULL REFERENCES expert_profiles(id),
    submitted_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (survey_id, expert_id)
);

CREATE TABLE survey_answers (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    response_id     UUID NOT NULL REFERENCES survey_responses(id) ON DELETE CASCADE,
    question_id     UUID NOT NULL REFERENCES survey_questions(id),
    answer_text     TEXT,
    selected_options UUID[], -- references survey_question_options
    rating_value    INTEGER,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- RATINGS AND FEEDBACK
-- ============================================================

CREATE TABLE call_ratings (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    call_id         UUID NOT NULL REFERENCES calls(id),
    rated_by        UUID NOT NULL REFERENCES users(id),
    rating          INTEGER NOT NULL CHECK (rating BETWEEN 1 AND 5),
    expertise_rating INTEGER CHECK (expertise_rating BETWEEN 1 AND 5),
    communication_rating INTEGER CHECK (communication_rating BETWEEN 1 AND 5),
    feedback        TEXT,
    is_from_expert  BOOLEAN NOT NULL DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (call_id, rated_by)
);

CREATE INDEX idx_ratings_call ON call_ratings (call_id);

-- ============================================================
-- PAYMENTS AND BILLING
-- ============================================================

CREATE TABLE invoices (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    invoice_number  VARCHAR(50) NOT NULL UNIQUE,
    status          VARCHAR(20) NOT NULL DEFAULT 'draft' CHECK (status IN ('draft', 'issued', 'paid', 'overdue', 'cancelled', 'refunded')),
    subtotal_cents  BIGINT NOT NULL,
    tax_cents       BIGINT NOT NULL DEFAULT 0,
    total_cents     BIGINT NOT NULL,
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    issued_at       TIMESTAMPTZ,
    due_at          TIMESTAMPTZ,
    paid_at         TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE invoice_line_items (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    invoice_id      UUID NOT NULL REFERENCES invoices(id) ON DELETE CASCADE,
    call_id         UUID REFERENCES calls(id),
    description     VARCHAR(500) NOT NULL,
    quantity        NUMERIC(10,2) NOT NULL DEFAULT 1,
    unit_price_cents INTEGER NOT NULL,
    total_cents     BIGINT NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE expert_payouts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    expert_id       UUID NOT NULL REFERENCES expert_profiles(id),
    call_id         UUID REFERENCES calls(id),
    amount_cents    BIGINT NOT NULL,
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    status          VARCHAR(20) NOT NULL DEFAULT 'pending' CHECK (status IN ('pending', 'processing', 'paid', 'failed')),
    payment_method  VARCHAR(30),
    payment_ref     VARCHAR(255),
    paid_at         TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_payouts_expert ON expert_payouts (expert_id);
CREATE INDEX idx_payouts_status ON expert_payouts (status);

-- ============================================================
-- AUDIT LOG
-- ============================================================

CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID REFERENCES users(id),
    action          VARCHAR(100) NOT NULL,
    entity_type     VARCHAR(50) NOT NULL,
    entity_id       UUID NOT NULL,
    old_values      TEXT, -- serialised JSON
    new_values      TEXT, -- serialised JSON
    ip_address      INET,
    user_agent      TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_entity ON audit_log (entity_type, entity_id);
CREATE INDEX idx_audit_user ON audit_log (user_id);
CREATE INDEX idx_audit_created ON audit_log (created_at);

-- ============================================================
-- AVAILABILITY / SCHEDULING
-- ============================================================

CREATE TABLE expert_availability (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    expert_id       UUID NOT NULL REFERENCES expert_profiles(id) ON DELETE CASCADE,
    day_of_week     SMALLINT NOT NULL CHECK (day_of_week BETWEEN 0 AND 6),
    start_time      TIME NOT NULL,
    end_time        TIME NOT NULL,
    timezone        VARCHAR(50) NOT NULL DEFAULT 'UTC',
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    CHECK (end_time > start_time)
);

CREATE INDEX idx_availability_expert ON expert_availability (expert_id);

CREATE TABLE calendar_sync (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    provider        VARCHAR(20) NOT NULL CHECK (provider IN ('google', 'outlook', 'apple')),
    external_calendar_id VARCHAR(255),
    access_token_enc TEXT, -- encrypted
    refresh_token_enc TEXT, -- encrypted
    synced_at       TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- AI MATCHING VECTORS (for semantic search)
-- ============================================================

CREATE TABLE expert_embeddings (
    expert_id       UUID PRIMARY KEY REFERENCES expert_profiles(id) ON DELETE CASCADE,
    embedding       vector(1536) NOT NULL, -- pgvector extension
    model_version   VARCHAR(50) NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE project_embeddings (
    project_id      UUID PRIMARY KEY REFERENCES projects(id) ON DELETE CASCADE,
    embedding       vector(1536) NOT NULL,
    model_version   VARCHAR(50) NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Approximate nearest-neighbour index for semantic matching
CREATE INDEX idx_expert_emb_ivfflat ON expert_embeddings USING ivfflat (embedding vector_cosine_ops) WITH (lists = 100);
```

---

## Trade-offs

**Strengths:**
- Full referential integrity protects data consistency across the compliance-heavy workflow.
- Mature ecosystem: pgvector for semantic search, pg_trgm for fuzzy text, built-in full-text search.
- Row-level security (RLS) in PostgreSQL enables per-organisation data isolation without separate databases.
- Straightforward backup, replication, and disaster recovery.

**Weaknesses:**
- Skills taxonomy queries (hierarchical categories, multi-hop skill relationships) require recursive CTEs, which are slower than graph traversals.
- The schema contains ~30 tables; adding new entity types means new migrations.
- Transcript full-text search at scale (millions of transcripts) may require a dedicated search engine (Elasticsearch/Meilisearch) alongside PostgreSQL.

## Scalability Considerations

- **Read replicas** handle reporting and search workloads separately from transactional writes.
- **Partitioning** the audit_log and calls tables by date keeps query performance stable as data grows.
- **pgvector** supports HNSW and IVFFlat indexes for sub-100ms semantic matching up to ~10M vectors.
- Beyond ~50M transcripts, consider offloading full-text search to a dedicated engine.

## Migration Path

This schema serves as a solid foundation. If flexibility needs grow (e.g. expert profiles gain many optional domain-specific fields), individual tables can adopt JSONB columns (see Suggestion 3) without restructuring the core relational model. If graph queries become dominant for expert discovery, a read-side graph projection can be added alongside this schema.
