# Data Model Suggestion 3: Hybrid Relational + JSONB Model (PostgreSQL)

## Approach

A PostgreSQL schema that keeps core transactional entities in normalized relational tables but uses JSONB columns for inherently flexible, nested, or schema-evolving data. This combines the referential integrity of a relational model with the document-store flexibility of JSONB where it adds genuine value -- expert profile metadata, AI-generated outputs, compliance rule configurations, and survey structures.

## Why This Suits an Expert Network Platform

Expert network platforms must serve two very different data patterns:

1. **Structured transactional data** -- users, organisations, engagements, calls, payments -- that demands referential integrity, ACID transactions, and consistent querying.
2. **Semi-structured, evolving data** -- expert profiles have variable fields across industries (a pharma expert's profile looks very different from a fintech expert's), AI-generated summaries have evolving schemas as models improve, compliance rules vary by jurisdiction, and survey structures are inherently dynamic.

The hybrid approach avoids the rigidity of pure 3NF (where every new profile field requires a migration) and the chaos of pure document storage (where data integrity is application-enforced). JSONB columns with GIN indexes and JSON Schema validation constraints provide a pragmatic middle ground.

---

## Schema Definition

```sql
-- ============================================================
-- EXTENSIONS
-- ============================================================

CREATE EXTENSION IF NOT EXISTS "pgcrypto";
CREATE EXTENSION IF NOT EXISTS "vector";    -- pgvector for embeddings

-- ============================================================
-- USERS
-- ============================================================

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email           VARCHAR(255) NOT NULL UNIQUE,
    password_hash   VARCHAR(255) NOT NULL,
    first_name      VARCHAR(100) NOT NULL,
    last_name       VARCHAR(100) NOT NULL,
    user_type       VARCHAR(20) NOT NULL CHECK (user_type IN ('expert', 'client', 'admin')),
    status          VARCHAR(20) NOT NULL DEFAULT 'pending'
        CHECK (status IN ('pending', 'active', 'suspended', 'deactivated')),
    -- Flexible user preferences and settings as JSONB
    preferences     JSONB NOT NULL DEFAULT '{}'::jsonb,
    -- e.g. {"notification_channels": ["email","sms"], "locale": "en-US", "theme": "dark"}
    contact_info    JSONB NOT NULL DEFAULT '{}'::jsonb,
    -- e.g. {"phone": "+1...", "secondary_email": "...", "address": {...}}
    email_verified  BOOLEAN NOT NULL DEFAULT FALSE,
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_users_email ON users (email);
CREATE INDEX idx_users_type_status ON users (user_type, status);

-- ============================================================
-- ORGANISATIONS
-- ============================================================

CREATE TABLE organisations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(255) NOT NULL UNIQUE,
    org_type        VARCHAR(30) NOT NULL CHECK (org_type IN ('consulting', 'investment_fund', 'corporate', 'research', 'other')),
    subscription_tier VARCHAR(20) NOT NULL DEFAULT 'payg'
        CHECK (subscription_tier IN ('payg', 'starter', 'professional', 'enterprise')),
    -- Flexible billing and settings
    billing_info    JSONB NOT NULL DEFAULT '{}'::jsonb,
    -- e.g. {"email": "...", "stripe_customer_id": "...", "tax_id": "...", "address": {...}}
    settings        JSONB NOT NULL DEFAULT '{}'::jsonb,
    -- e.g. {"default_currency": "USD", "auto_nda": true, "compliance_rules": [...]}
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE organisation_members (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role            VARCHAR(20) NOT NULL DEFAULT 'member'
        CHECK (role IN ('owner', 'admin', 'manager', 'member')),
    permissions     JSONB NOT NULL DEFAULT '[]'::jsonb,
    -- e.g. ["manage_projects", "view_billing", "manage_compliance"]
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organisation_id, user_id)
);

CREATE INDEX idx_org_members_org ON organisation_members (organisation_id);
CREATE INDEX idx_org_members_user ON organisation_members (user_id);

-- ============================================================
-- EXPERT PROFILES (relational core + JSONB extensions)
-- ============================================================

CREATE TABLE expert_profiles (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL UNIQUE REFERENCES users(id) ON DELETE CASCADE,

    -- Core searchable/filterable fields remain relational
    headline        VARCHAR(300),
    hourly_rate_cents INTEGER NOT NULL DEFAULT 0,
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    timezone        VARCHAR(50) NOT NULL DEFAULT 'UTC',
    is_available    BOOLEAN NOT NULL DEFAULT TRUE,
    avg_rating      NUMERIC(3,2) DEFAULT 0.00,
    total_calls     INTEGER NOT NULL DEFAULT 0,
    verification_status VARCHAR(20) NOT NULL DEFAULT 'unverified'
        CHECK (verification_status IN ('unverified', 'pending', 'verified', 'rejected')),

    -- Rich profile data as JSONB -- varies by industry, evolves over time
    bio             JSONB NOT NULL DEFAULT '{}'::jsonb,
    -- e.g. {"short": "...", "long": "...", "expertise_areas": ["...", "..."]}

    skills          JSONB NOT NULL DEFAULT '[]'::jsonb,
    -- Array of skill objects: [{"id":"uuid","name":"...","category":"...","proficiency":"expert","years":8}]

    industries      JSONB NOT NULL DEFAULT '[]'::jsonb,
    -- [{"id":"uuid","name":"Healthcare","is_primary":true,"sub_sectors":["Biotech","MedDevices"]}]

    work_history    JSONB NOT NULL DEFAULT '[]'::jsonb,
    -- [{"company":"Acme","title":"VP Eng","start":"2020-01","end":"2024-06","current":false,"description":"..."}]

    education       JSONB NOT NULL DEFAULT '[]'::jsonb,
    -- [{"institution":"MIT","degree":"PhD","field":"Materials Science","year":2015}]

    certifications  JSONB NOT NULL DEFAULT '[]'::jsonb,
    -- [{"name":"CFA Level III","issuer":"CFA Institute","year":2018}]

    publications    JSONB NOT NULL DEFAULT '[]'::jsonb,
    -- [{"title":"...","venue":"Nature","year":2023,"url":"..."}]

    languages       JSONB NOT NULL DEFAULT '[]'::jsonb,
    -- [{"code":"en","name":"English","proficiency":"native"},{"code":"fr","name":"French","proficiency":"fluent"}]

    social_links    JSONB NOT NULL DEFAULT '{}'::jsonb,
    -- {"linkedin":"https://...","twitter":"https://...","personal_website":"https://..."}

    -- Domain-specific metadata (varies by expert type)
    domain_metadata JSONB NOT NULL DEFAULT '{}'::jsonb,
    -- Pharma: {"therapeutic_areas":["oncology"],"regulatory_experience":["FDA","EMA"]}
    -- FinTech: {"markets":["US","EU"],"asset_classes":["equities","fixed_income"]}

    -- Embedding for AI semantic matching
    embedding       vector(1536),

    onboarded_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Relational indexes for core filtering
CREATE INDEX idx_expert_available ON expert_profiles (is_available) WHERE is_available = TRUE;
CREATE INDEX idx_expert_rate ON expert_profiles (hourly_rate_cents);
CREATE INDEX idx_expert_rating ON expert_profiles (avg_rating DESC);
CREATE INDEX idx_expert_verification ON expert_profiles (verification_status);

-- GIN indexes for JSONB search
CREATE INDEX idx_expert_skills_gin ON expert_profiles USING gin(skills jsonb_path_ops);
CREATE INDEX idx_expert_industries_gin ON expert_profiles USING gin(industries jsonb_path_ops);
CREATE INDEX idx_expert_languages_gin ON expert_profiles USING gin(languages jsonb_path_ops);
CREATE INDEX idx_expert_domain_gin ON expert_profiles USING gin(domain_metadata jsonb_path_ops);

-- Vector similarity index for AI matching
CREATE INDEX idx_expert_embedding ON expert_profiles USING ivfflat (embedding vector_cosine_ops) WITH (lists = 100);

-- ============================================================
-- SKILLS TAXONOMY (reference data, relational)
-- ============================================================

CREATE TABLE skill_taxonomy (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(150) NOT NULL,
    slug            VARCHAR(150) NOT NULL UNIQUE,
    parent_id       UUID REFERENCES skill_taxonomy(id),
    category        VARCHAR(100),
    aliases         JSONB DEFAULT '[]'::jsonb,  -- ["ML", "machine learning", "AI/ML"]
    metadata        JSONB DEFAULT '{}'::jsonb,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_skill_taxonomy_parent ON skill_taxonomy (parent_id);
CREATE INDEX idx_skill_taxonomy_aliases ON skill_taxonomy USING gin(aliases jsonb_path_ops);

CREATE TABLE industry_taxonomy (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(150) NOT NULL,
    parent_id       UUID REFERENCES industry_taxonomy(id),
    sic_code        VARCHAR(10),
    naics_code      VARCHAR(10),
    metadata        JSONB DEFAULT '{}'::jsonb,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- PROJECTS
-- ============================================================

CREATE TABLE projects (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    created_by      UUID NOT NULL REFERENCES users(id),
    title           VARCHAR(300) NOT NULL,
    description     TEXT,
    brief           TEXT,
    status          VARCHAR(20) NOT NULL DEFAULT 'draft'
        CHECK (status IN ('draft', 'active', 'paused', 'completed', 'archived')),
    budget_cents    BIGINT,
    currency        CHAR(3) NOT NULL DEFAULT 'USD',

    -- Flexible matching criteria as JSONB
    matching_criteria JSONB NOT NULL DEFAULT '{}'::jsonb,
    -- {
    --   "required_skills": [{"id":"uuid","importance":"required"}],
    --   "preferred_industries": ["uuid"],
    --   "min_years_experience": 10,
    --   "preferred_languages": ["en","de"],
    --   "geographic_focus": ["US","EU"],
    --   "custom_filters": {"regulatory_experience": ["FDA"]}
    -- }

    -- AI matching results stored as JSONB
    ai_match_results JSONB DEFAULT '[]'::jsonb,
    -- [{"expert_id":"uuid","score":0.94,"matched_on":["skill1","industry2"],"generated_at":"..."}]

    -- Embedding of the research brief for semantic matching
    brief_embedding vector(1536),

    start_date      DATE,
    end_date        DATE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_projects_org ON projects (organisation_id);
CREATE INDEX idx_projects_status ON projects (status);
CREATE INDEX idx_projects_matching ON projects USING gin(matching_criteria jsonb_path_ops);
CREATE INDEX idx_projects_brief_emb ON projects USING ivfflat (brief_embedding vector_cosine_ops) WITH (lists = 50);

-- ============================================================
-- ENGAGEMENTS
-- ============================================================

CREATE TABLE engagements (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES projects(id),
    expert_id       UUID NOT NULL REFERENCES expert_profiles(id),
    requested_by    UUID NOT NULL REFERENCES users(id),
    status          VARCHAR(30) NOT NULL DEFAULT 'requested'
        CHECK (status IN ('requested', 'expert_invited', 'accepted', 'compliance_pending',
                          'compliance_cleared', 'scheduled', 'completed', 'declined', 'cancelled')),
    match_score     NUMERIC(5,4),
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
-- COMPLIANCE (relational core with flexible rule config)
-- ============================================================

CREATE TABLE compliance_checks (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    engagement_id   UUID NOT NULL REFERENCES engagements(id),
    check_type      VARCHAR(30) NOT NULL
        CHECK (check_type IN ('nda', 'conflict_of_interest', 'mnpi_acknowledgement', 'restricted_list', 'gdpr_consent', 'custom')),
    status          VARCHAR(20) NOT NULL DEFAULT 'pending'
        CHECK (status IN ('pending', 'passed', 'failed', 'waived', 'expired')),
    automated       BOOLEAN NOT NULL DEFAULT TRUE,
    performed_by    UUID REFERENCES users(id),

    -- Flexible check details and evidence as JSONB
    check_config    JSONB NOT NULL DEFAULT '{}'::jsonb,
    -- e.g. {"rules_version":"v2","restricted_lists_checked":["uuid1","uuid2"]}
    result_details  JSONB NOT NULL DEFAULT '{}'::jsonb,
    -- e.g. {"matches_found":0,"lists_checked":3,"confidence":0.99}
    evidence        JSONB NOT NULL DEFAULT '{}'::jsonb,
    -- e.g. {"screenshot_url":"...","signed_document_url":"...","acknowledgement_text":"..."}

    checked_at      TIMESTAMPTZ,
    expires_at      TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_compliance_engagement ON compliance_checks (engagement_id);
CREATE INDEX idx_compliance_status ON compliance_checks (check_type, status);

CREATE TABLE restricted_lists (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    -- Entries stored as JSONB array for flexible entity types
    entries         JSONB NOT NULL DEFAULT '[]'::jsonb,
    -- [{"entity_type":"ticker","entity_name":"ACME","reason":"Active M&A","added_by":"uuid","valid_from":"2026-01-01","valid_until":null}]
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_restricted_org ON restricted_lists (organisation_id);
CREATE INDEX idx_restricted_entries ON restricted_lists USING gin(entries jsonb_path_ops);

-- ============================================================
-- CALLS
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

    -- Meeting provider details as JSONB (varies by provider)
    meeting_details JSONB NOT NULL DEFAULT '{}'::jsonb,
    -- {"provider":"zoom","url":"https://...","meeting_id":"123","passcode":"abc","dial_in":"+1..."}

    recording_url   TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_calls_engagement ON calls (engagement_id);
CREATE INDEX idx_calls_scheduled ON calls (scheduled_start);
CREATE INDEX idx_calls_status ON calls (status);

-- ============================================================
-- TRANSCRIPTS AND AI-GENERATED CONTENT
-- ============================================================

CREATE TABLE transcripts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    call_id         UUID NOT NULL UNIQUE REFERENCES calls(id),
    raw_text        TEXT NOT NULL,
    language        CHAR(5) NOT NULL DEFAULT 'en',

    -- AI-generated structured content as JSONB
    summaries       JSONB NOT NULL DEFAULT '{}'::jsonb,
    -- {
    --   "executive": "...",
    --   "expert_background": "...",
    --   "sentiment": {"overall":"positive","confidence":0.87},
    --   "key_takeaways": ["...", "..."],
    --   "topics_discussed": [{"topic":"...","sentiment":"neutral","depth":"deep"}],
    --   "follow_up_questions": ["..."]
    -- }

    -- Speaker diarisation and structured segments
    segments        JSONB NOT NULL DEFAULT '[]'::jsonb,
    -- [{"speaker":"expert","start_ms":0,"end_ms":15000,"text":"..."},...]

    -- Processing metadata
    processing_info JSONB NOT NULL DEFAULT '{}'::jsonb,
    -- {"model":"whisper-v4","confidence":0.943,"word_count":8420,"processed_at":"..."}

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Full-text search on raw transcript
CREATE INDEX idx_transcript_fts ON transcripts USING gin(to_tsvector('english', raw_text));
-- GIN index on structured summaries for topic search
CREATE INDEX idx_transcript_summaries ON transcripts USING gin(summaries jsonb_path_ops);

-- ============================================================
-- CROSS-CALL SYNTHESIS
-- ============================================================

CREATE TABLE synthesis_reports (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES projects(id),
    title           VARCHAR(300) NOT NULL,
    call_ids        UUID[] NOT NULL,     -- array of call IDs included
    content         JSONB NOT NULL,
    -- {
    --   "themes": [{"theme":"...","supporting_calls":["uuid"],"summary":"..."}],
    --   "consensus_views": ["..."],
    --   "divergent_views": [{"topic":"...","views":[{"expert":"...","position":"..."}]}],
    --   "knowledge_gaps": ["..."],
    --   "recommendations": ["..."]
    -- }
    model_version   VARCHAR(50),
    created_by      UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_synthesis_project ON synthesis_reports (project_id);

-- ============================================================
-- SURVEYS (fully JSONB-driven structure)
-- ============================================================

CREATE TABLE surveys (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES projects(id),
    created_by      UUID NOT NULL REFERENCES users(id),
    title           VARCHAR(300) NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'draft'
        CHECK (status IN ('draft', 'active', 'closed', 'archived')),

    -- Survey structure as JSONB (flexible question types, branching logic)
    structure       JSONB NOT NULL DEFAULT '{"questions":[]}'::jsonb,
    -- {
    --   "questions": [
    --     {"id":"q1","type":"single_choice","text":"...","required":true,"options":["A","B","C"]},
    --     {"id":"q2","type":"rating","text":"...","required":true,"scale":{"min":1,"max":10}},
    --     {"id":"q3","type":"text","text":"...","required":false,"max_length":2000},
    --     {"id":"q4","type":"multi_choice","text":"...","show_if":{"q1":"A"},"options":[...]}
    --   ],
    --   "settings": {"anonymous":false,"allow_save_draft":true}
    -- }

    opens_at        TIMESTAMPTZ,
    closes_at       TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE survey_responses (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    survey_id       UUID NOT NULL REFERENCES surveys(id),
    expert_id       UUID NOT NULL REFERENCES expert_profiles(id),

    -- Answers as JSONB (mirrors survey structure)
    answers         JSONB NOT NULL DEFAULT '{}'::jsonb,
    -- {"q1":"A","q2":8,"q3":"Detailed text response...","q4":["opt1","opt3"]}

    status          VARCHAR(20) NOT NULL DEFAULT 'in_progress'
        CHECK (status IN ('in_progress', 'submitted')),
    submitted_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (survey_id, expert_id)
);

-- ============================================================
-- RATINGS AND FEEDBACK
-- ============================================================

CREATE TABLE call_ratings (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    call_id         UUID NOT NULL REFERENCES calls(id),
    rated_by        UUID NOT NULL REFERENCES users(id),
    rating          INTEGER NOT NULL CHECK (rating BETWEEN 1 AND 5),
    is_from_expert  BOOLEAN NOT NULL DEFAULT FALSE,
    -- Detailed breakdown as JSONB (extensible dimensions)
    dimensions      JSONB NOT NULL DEFAULT '{}'::jsonb,
    -- {"expertise":5,"communication":4,"preparedness":5,"responsiveness":4}
    feedback        TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (call_id, rated_by)
);

-- ============================================================
-- PAYMENTS
-- ============================================================

CREATE TABLE invoices (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    invoice_number  VARCHAR(50) NOT NULL UNIQUE,
    status          VARCHAR(20) NOT NULL DEFAULT 'draft'
        CHECK (status IN ('draft', 'issued', 'paid', 'overdue', 'cancelled', 'refunded')),
    -- Line items as JSONB (avoids separate table for simple invoice data)
    line_items      JSONB NOT NULL DEFAULT '[]'::jsonb,
    -- [{"call_id":"uuid","description":"Expert call - 56 min","quantity":1,"unit_price_cents":50000,"total_cents":46667}]
    subtotal_cents  BIGINT NOT NULL,
    tax_cents       BIGINT NOT NULL DEFAULT 0,
    total_cents     BIGINT NOT NULL,
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    -- Payment provider details
    payment_info    JSONB NOT NULL DEFAULT '{}'::jsonb,
    -- {"stripe_invoice_id":"...","payment_intent":"...","receipt_url":"..."}
    issued_at       TIMESTAMPTZ,
    due_at          TIMESTAMPTZ,
    paid_at         TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE expert_payouts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    expert_id       UUID NOT NULL REFERENCES expert_profiles(id),
    call_id         UUID REFERENCES calls(id),
    amount_cents    BIGINT NOT NULL,
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    status          VARCHAR(20) NOT NULL DEFAULT 'pending'
        CHECK (status IN ('pending', 'processing', 'paid', 'failed')),
    payment_details JSONB NOT NULL DEFAULT '{}'::jsonb,
    -- {"method":"bank_transfer","stripe_transfer_id":"...","bank_last4":"1234"}
    paid_at         TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_payouts_expert ON expert_payouts (expert_id);
CREATE INDEX idx_payouts_status ON expert_payouts (status);

-- ============================================================
-- EXPERT AVAILABILITY
-- ============================================================

CREATE TABLE expert_availability (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    expert_id       UUID NOT NULL REFERENCES expert_profiles(id) ON DELETE CASCADE,
    -- Recurring availability as JSONB
    recurring_slots JSONB NOT NULL DEFAULT '[]'::jsonb,
    -- [{"day":1,"start":"09:00","end":"17:00","timezone":"America/New_York"}]
    -- Override slots (specific dates)
    overrides       JSONB NOT NULL DEFAULT '[]'::jsonb,
    -- [{"date":"2026-06-15","available":false,"reason":"vacation"}]
    calendar_sync   JSONB NOT NULL DEFAULT '{}'::jsonb,
    -- {"provider":"google","synced_at":"...","calendar_id":"..."}
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (expert_id)
);

-- ============================================================
-- AUDIT LOG
-- ============================================================

CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID REFERENCES users(id),
    action          VARCHAR(100) NOT NULL,
    entity_type     VARCHAR(50) NOT NULL,
    entity_id       UUID NOT NULL,
    changes         JSONB NOT NULL DEFAULT '{}'::jsonb,
    -- {"old":{"status":"pending"},"new":{"status":"active"},"fields_changed":["status"]}
    context         JSONB NOT NULL DEFAULT '{}'::jsonb,
    -- {"ip":"1.2.3.4","user_agent":"...","correlation_id":"uuid"}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_entity ON audit_log (entity_type, entity_id);
CREATE INDEX idx_audit_user ON audit_log (user_id);
CREATE INDEX idx_audit_created ON audit_log (created_at DESC);
-- Partition by month for large-scale deployments
-- CREATE TABLE audit_log_y2026m06 PARTITION OF audit_log FOR VALUES FROM ('2026-06-01') TO ('2026-07-01');
```

---

## Example JSONB Queries

```sql
-- Find experts with a specific skill at 'expert' proficiency
SELECT id, user_id, headline, hourly_rate_cents
FROM expert_profiles
WHERE skills @> '[{"name": "lithium-ion batteries", "proficiency": "expert"}]'
  AND is_available = TRUE
ORDER BY avg_rating DESC;

-- Find experts in a specific therapeutic area (domain metadata)
SELECT id, headline, domain_metadata->'therapeutic_areas' AS areas
FROM expert_profiles
WHERE domain_metadata @> '{"therapeutic_areas": ["oncology"]}'
  AND verification_status = 'verified';

-- Search transcript summaries for a specific topic
SELECT t.id, t.summaries->'executive' AS executive_summary
FROM transcripts t
WHERE t.summaries->'key_takeaways' @> '["supply chain disruption"]';

-- Get survey responses with specific answers
SELECT sr.expert_id, sr.answers->>'q1' AS market_outlook
FROM survey_responses sr
WHERE sr.survey_id = 'uuid'
  AND sr.answers->>'q1' = 'Bullish'
  AND sr.status = 'submitted';
```

---

## Trade-offs

**Strengths:**
- **Best of both worlds**: relational integrity for core business logic, document flexibility for variable data.
- **Fewer tables**: JSONB columns eliminate many join tables (expert_skills, expert_industries, survey_questions, survey_options, invoice_line_items) without losing queryability.
- **Schema evolution**: adding new fields to expert profiles, compliance checks, or AI outputs requires no migration -- just update the application code.
- **Single database**: no polyglot persistence complexity; PostgreSQL handles relational, document, full-text, and vector workloads.
- **GIN indexes**: JSONB queries with containment operators (@>, ?) are fast with proper indexing.

**Weaknesses:**
- **No referential integrity inside JSONB**: skill IDs in the expert_profiles.skills JSONB column are not foreign-key enforced. Application logic or triggers must validate references.
- **Harder to enforce constraints**: CHECK constraints on JSONB are limited; complex validation must live in the application layer.
- **Query complexity**: JSONB path queries are less readable than simple JOINs; developers need to learn the JSONB operator syntax.
- **Reporting**: BI tools may struggle with JSONB columns; extraction views or materialized views may be needed for analytics.

## Scalability Considerations

- **GIN index size**: large JSONB columns with GIN indexes consume significant disk space; monitor and use jsonb_path_ops for containment-only queries.
- **Materialized views**: pre-compute common JSONB extractions for dashboards and reporting.
- **Partial indexes**: combine relational WHERE clauses with JSONB conditions for selective indexing.
- **TOAST compression**: PostgreSQL automatically compresses large JSONB values, but very large documents (>8KB) incur TOAST overhead.

## Migration Path

This schema can evolve in either direction. If a JSONB column's structure stabilizes and needs stronger integrity guarantees, extract it into relational tables. If the system grows to need a dedicated search engine, the JSONB columns make it straightforward to sync documents to Elasticsearch or Meilisearch. The pgvector embeddings can later be moved to a dedicated vector database (Pinecone, Weaviate) if matching workloads outgrow PostgreSQL's capacity.
