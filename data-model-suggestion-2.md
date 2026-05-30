# Data Model Suggestion 2: Event-Sourced / CQRS Model

## Approach

An event-sourcing architecture where every state change is captured as an immutable domain event in an append-only event store. The write side processes commands and emits events; the read side maintains denormalized projections optimized for specific query patterns. This is implemented using PostgreSQL as the event store with separate projection tables.

## Why This Suits an Expert Network Platform

Expert network platforms have strong audit and compliance requirements. Every engagement passes through a multi-step lifecycle (requested -> compliance-checked -> scheduled -> completed -> paid) with regulatory bodies requiring full traceability. Event sourcing provides:

- **Complete audit trail by construction** -- every state transition is an immutable event, satisfying SEC Regulation FD and CFA Institute requirements without bolted-on audit logging.
- **Temporal queries** -- reconstruct the exact state of any engagement, compliance check, or expert profile at any point in time, critical for regulatory investigations.
- **Event replay** -- rebuild read models, feed new analytics, or populate new search indexes by replaying the event stream.
- **Natural fit for the engagement lifecycle** -- the multi-step compliance and engagement workflow maps directly to a sequence of domain events.

The main trade-offs are increased complexity in the application layer, eventual consistency between write and read sides, and a steeper learning curve for developers.

---

## Event Store Schema

```sql
-- ============================================================
-- EVENT STORE (append-only)
-- ============================================================

CREATE TABLE event_store (
    event_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_id       UUID NOT NULL,           -- aggregate root ID
    stream_type     VARCHAR(50) NOT NULL,     -- e.g. 'Expert', 'Engagement', 'Call'
    event_type      VARCHAR(100) NOT NULL,    -- e.g. 'ExpertRegistered', 'ComplianceCheckPassed'
    event_version   INTEGER NOT NULL,         -- per-stream sequence number
    payload         JSONB NOT NULL,           -- event data
    metadata        JSONB NOT NULL DEFAULT '{}', -- correlation IDs, user context, IP
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (stream_id, event_version)         -- optimistic concurrency control
);

CREATE INDEX idx_events_stream ON event_store (stream_id, event_version);
CREATE INDEX idx_events_type ON event_store (event_type);
CREATE INDEX idx_events_created ON event_store (created_at);
CREATE INDEX idx_events_stream_type ON event_store (stream_type);

-- Snapshot table for aggregates with many events
CREATE TABLE event_snapshots (
    stream_id       UUID NOT NULL,
    stream_type     VARCHAR(50) NOT NULL,
    snapshot_version INTEGER NOT NULL,
    state           JSONB NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (stream_id, snapshot_version)
);
```

---

## Domain Events

### User / Expert Events

```jsonc
// ExpertRegistered
{
  "event_type": "ExpertRegistered",
  "stream_type": "Expert",
  "payload": {
    "user_id": "uuid",
    "email": "expert@example.com",
    "first_name": "Jane",
    "last_name": "Smith",
    "headline": "Former VP Engineering at Acme Corp",
    "hourly_rate_cents": 50000,
    "currency": "USD",
    "timezone": "America/New_York"
  }
}

// ExpertProfileUpdated
{
  "event_type": "ExpertProfileUpdated",
  "payload": {
    "fields_changed": ["headline", "hourly_rate_cents"],
    "headline": "CEO & Founder, Acme Consulting",
    "hourly_rate_cents": 75000
  }
}

// ExpertSkillsUpdated
{
  "event_type": "ExpertSkillsUpdated",
  "payload": {
    "skills_added": [{"skill_id": "uuid", "proficiency": "expert"}],
    "skills_removed": ["uuid"]
  }
}

// ExpertVerified
{
  "event_type": "ExpertVerified",
  "payload": {
    "verified_by": "uuid",
    "verification_method": "linkedin_cross_reference"
  }
}

// ExpertAvailabilitySet
{
  "event_type": "ExpertAvailabilitySet",
  "payload": {
    "slots": [
      {"day_of_week": 1, "start_time": "09:00", "end_time": "17:00", "timezone": "America/New_York"}
    ]
  }
}
```

### Organisation Events

```jsonc
// OrganisationCreated
{
  "event_type": "OrganisationCreated",
  "stream_type": "Organisation",
  "payload": {
    "name": "Apex Capital Partners",
    "org_type": "investment_fund",
    "subscription_tier": "enterprise",
    "billing_email": "billing@apexcapital.com"
  }
}

// MemberAddedToOrganisation
{
  "event_type": "MemberAddedToOrganisation",
  "payload": {
    "user_id": "uuid",
    "role": "manager"
  }
}

// RestrictedListUpdated
{
  "event_type": "RestrictedListUpdated",
  "payload": {
    "list_id": "uuid",
    "entries_added": [
      {"entity_type": "ticker", "entity_name": "ACME", "reason": "Active M&A process"}
    ],
    "entries_removed": []
  }
}
```

### Project Events

```jsonc
// ProjectCreated
{
  "event_type": "ProjectCreated",
  "stream_type": "Project",
  "payload": {
    "organisation_id": "uuid",
    "created_by": "uuid",
    "title": "EV Battery Supply Chain Analysis",
    "brief": "Seeking experts in lithium-ion battery manufacturing...",
    "budget_cents": 2500000,
    "required_skills": ["uuid1", "uuid2"]
  }
}

// ProjectBriefMatched (AI matching result)
{
  "event_type": "ProjectBriefMatched",
  "payload": {
    "matched_experts": [
      {"expert_id": "uuid", "match_score": 0.9423, "matching_skills": ["uuid"]},
      {"expert_id": "uuid", "match_score": 0.8871, "matching_skills": ["uuid"]}
    ],
    "model_version": "semantic-match-v3"
  }
}
```

### Engagement Lifecycle Events

```jsonc
// EngagementRequested
{
  "event_type": "EngagementRequested",
  "stream_type": "Engagement",
  "payload": {
    "project_id": "uuid",
    "expert_id": "uuid",
    "requested_by": "uuid",
    "hourly_rate_cents": 50000,
    "match_score": 0.9423
  }
}

// ExpertInvited
{
  "event_type": "ExpertInvited",
  "payload": {
    "invited_at": "2026-05-29T10:00:00Z",
    "invitation_message": "We'd like to schedule a call..."
  }
}

// ExpertAccepted / ExpertDeclined
{
  "event_type": "ExpertAccepted",
  "payload": {
    "accepted_at": "2026-05-29T14:30:00Z"
  }
}

// ComplianceCheckInitiated
{
  "event_type": "ComplianceCheckInitiated",
  "payload": {
    "checks_required": ["nda", "conflict_of_interest", "mnpi_acknowledgement", "restricted_list"]
  }
}

// ComplianceCheckPassed / ComplianceCheckFailed
{
  "event_type": "ComplianceCheckPassed",
  "payload": {
    "check_type": "restricted_list",
    "automated": true,
    "details": "No matches found against 3 active restricted lists"
  }
}

// EngagementComplianceCleared
{
  "event_type": "EngagementComplianceCleared",
  "payload": {
    "all_checks_passed": true,
    "cleared_at": "2026-05-29T15:00:00Z"
  }
}
```

### Call Events

```jsonc
// CallScheduled
{
  "event_type": "CallScheduled",
  "stream_type": "Call",
  "payload": {
    "engagement_id": "uuid",
    "scheduled_start": "2026-06-02T14:00:00Z",
    "scheduled_end": "2026-06-02T15:00:00Z",
    "meeting_provider": "zoom",
    "meeting_url": "https://zoom.us/j/123456"
  }
}

// CallStarted
{
  "event_type": "CallStarted",
  "payload": {
    "actual_start": "2026-06-02T14:02:00Z"
  }
}

// CallCompleted
{
  "event_type": "CallCompleted",
  "payload": {
    "actual_end": "2026-06-02T14:58:00Z",
    "duration_minutes": 56,
    "recording_url": "s3://recordings/uuid.mp4"
  }
}

// TranscriptGenerated
{
  "event_type": "TranscriptGenerated",
  "payload": {
    "transcript_id": "uuid",
    "word_count": 8420,
    "language": "en",
    "confidence_score": 0.943,
    "model_version": "whisper-v4"
  }
}

// SummaryGenerated
{
  "event_type": "SummaryGenerated",
  "payload": {
    "summary_type": "key_takeaways",
    "content": "1. Battery-grade lithium supply remains...",
    "model_version": "claude-opus-4-20250514"
  }
}

// CallRated
{
  "event_type": "CallRated",
  "payload": {
    "rated_by": "uuid",
    "is_from_expert": false,
    "rating": 5,
    "expertise_rating": 5,
    "communication_rating": 4,
    "feedback": "Exceptional depth of knowledge..."
  }
}
```

### Payment Events

```jsonc
// InvoiceIssued
{
  "event_type": "InvoiceIssued",
  "stream_type": "Invoice",
  "payload": {
    "organisation_id": "uuid",
    "invoice_number": "INV-2026-001234",
    "line_items": [
      {"call_id": "uuid", "description": "Expert call - 56 min", "total_cents": 46667}
    ],
    "subtotal_cents": 46667,
    "tax_cents": 0,
    "total_cents": 46667,
    "due_at": "2026-07-02T00:00:00Z"
  }
}

// PayoutInitiated
{
  "event_type": "PayoutInitiated",
  "stream_type": "Payout",
  "payload": {
    "expert_id": "uuid",
    "call_id": "uuid",
    "amount_cents": 37334,
    "currency": "USD",
    "payment_method": "bank_transfer"
  }
}
```

---

## Command Handlers

```
Command: RegisterExpert
  Validates: email uniqueness, required fields
  Emits: ExpertRegistered

Command: RequestEngagement
  Validates: project exists, expert exists, no duplicate active engagement
  Emits: EngagementRequested

Command: RunComplianceChecks
  Validates: engagement in correct state
  Orchestrates: parallel NDA, COI, MNPI, restricted-list checks
  Emits: ComplianceCheckInitiated, then ComplianceCheckPassed/Failed per check,
         then EngagementComplianceCleared or EngagementComplianceFailed

Command: ScheduleCall
  Validates: engagement compliance-cleared, expert available, no time conflict
  Emits: CallScheduled

Command: CompleteCall
  Validates: call in_progress
  Emits: CallCompleted
  Triggers: async TranscribeCall command

Command: SubmitRating
  Validates: call completed, user was participant, no duplicate rating
  Emits: CallRated
  Side-effect: updates expert avg_rating in read projection
```

---

## Read Projections (Denormalized Query Tables)

```sql
-- ============================================================
-- PROJECTION: Expert Directory (search & browse)
-- ============================================================

CREATE TABLE proj_expert_directory (
    expert_id       UUID PRIMARY KEY,
    user_id         UUID NOT NULL,
    email           VARCHAR(255) NOT NULL,
    full_name       VARCHAR(200) NOT NULL,
    headline        VARCHAR(300),
    hourly_rate_cents INTEGER,
    currency        CHAR(3),
    timezone        VARCHAR(50),
    languages       TEXT[],
    is_available    BOOLEAN NOT NULL DEFAULT TRUE,
    avg_rating      NUMERIC(3,2),
    total_calls     INTEGER NOT NULL DEFAULT 0,
    verification_status VARCHAR(20),
    skills          JSONB DEFAULT '[]',       -- [{id, name, category, proficiency}]
    industries      JSONB DEFAULT '[]',       -- [{id, name, is_primary}]
    work_history    JSONB DEFAULT '[]',       -- [{company, title, start, end}]
    last_updated    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_proj_expert_available ON proj_expert_directory (is_available, avg_rating DESC);
CREATE INDEX idx_proj_expert_skills ON proj_expert_directory USING gin(skills);

-- ============================================================
-- PROJECTION: Engagement Dashboard
-- ============================================================

CREATE TABLE proj_engagement_dashboard (
    engagement_id   UUID PRIMARY KEY,
    project_id      UUID NOT NULL,
    project_title   VARCHAR(300),
    organisation_id UUID NOT NULL,
    organisation_name VARCHAR(255),
    expert_id       UUID NOT NULL,
    expert_name     VARCHAR(200),
    status          VARCHAR(30) NOT NULL,
    match_score     NUMERIC(5,4),
    hourly_rate_cents INTEGER,
    compliance_status VARCHAR(30), -- 'pending', 'all_passed', 'has_failures'
    compliance_checks JSONB DEFAULT '[]', -- [{type, status, checked_at}]
    call_count      INTEGER NOT NULL DEFAULT 0,
    total_cost_cents BIGINT NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL,
    last_updated    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_proj_eng_project ON proj_engagement_dashboard (project_id);
CREATE INDEX idx_proj_eng_org ON proj_engagement_dashboard (organisation_id);
CREATE INDEX idx_proj_eng_status ON proj_engagement_dashboard (status);

-- ============================================================
-- PROJECTION: Call History & Transcripts
-- ============================================================

CREATE TABLE proj_call_history (
    call_id         UUID PRIMARY KEY,
    engagement_id   UUID NOT NULL,
    project_id      UUID NOT NULL,
    project_title   VARCHAR(300),
    expert_id       UUID NOT NULL,
    expert_name     VARCHAR(200),
    scheduled_start TIMESTAMPTZ NOT NULL,
    duration_minutes INTEGER,
    status          VARCHAR(20),
    has_transcript  BOOLEAN NOT NULL DEFAULT FALSE,
    has_summary     BOOLEAN NOT NULL DEFAULT FALSE,
    summary_preview TEXT,     -- first 500 chars of executive summary
    rating          INTEGER,
    recording_url   TEXT,
    last_updated    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_proj_calls_project ON proj_call_history (project_id);
CREATE INDEX idx_proj_calls_expert ON proj_call_history (expert_id);
CREATE INDEX idx_proj_calls_date ON proj_call_history (scheduled_start DESC);

-- ============================================================
-- PROJECTION: Compliance Audit View
-- ============================================================

CREATE TABLE proj_compliance_audit (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    engagement_id   UUID NOT NULL,
    expert_name     VARCHAR(200),
    organisation_name VARCHAR(255),
    project_title   VARCHAR(300),
    check_type      VARCHAR(30) NOT NULL,
    check_status    VARCHAR(20) NOT NULL,
    automated       BOOLEAN NOT NULL,
    details         TEXT,
    checked_at      TIMESTAMPTZ,
    event_id        UUID NOT NULL,   -- back-reference to source event
    event_timestamp TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_proj_audit_engagement ON proj_compliance_audit (engagement_id);
CREATE INDEX idx_proj_audit_type_status ON proj_compliance_audit (check_type, check_status);
CREATE INDEX idx_proj_audit_date ON proj_compliance_audit (event_timestamp DESC);

-- ============================================================
-- PROJECTION: Expert Earnings
-- ============================================================

CREATE TABLE proj_expert_earnings (
    expert_id       UUID NOT NULL,
    month           DATE NOT NULL,  -- first day of month
    calls_completed INTEGER NOT NULL DEFAULT 0,
    total_earned_cents BIGINT NOT NULL DEFAULT 0,
    total_paid_cents BIGINT NOT NULL DEFAULT 0,
    pending_cents   BIGINT NOT NULL DEFAULT 0,
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    PRIMARY KEY (expert_id, month)
);

-- ============================================================
-- PROJECTION: Organisation Spend
-- ============================================================

CREATE TABLE proj_org_spend (
    organisation_id UUID NOT NULL,
    month           DATE NOT NULL,
    calls_completed INTEGER NOT NULL DEFAULT 0,
    total_spent_cents BIGINT NOT NULL DEFAULT 0,
    invoiced_cents  BIGINT NOT NULL DEFAULT 0,
    paid_cents      BIGINT NOT NULL DEFAULT 0,
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    PRIMARY KEY (organisation_id, month)
);
```

---

## Trade-offs

**Strengths:**
- Complete audit trail is inherent -- no separate audit_log table needed; the event store IS the audit log.
- Temporal queries are trivial: replay events up to any timestamp to reconstruct past state.
- New read models (e.g. a knowledge-gap-analysis dashboard) can be built by replaying existing events without schema changes.
- Natural alignment with the multi-step compliance workflow: each step is a distinct event.
- Event streams can feed real-time notifications, analytics pipelines, and AI model training.

**Weaknesses:**
- **Eventual consistency**: read projections lag behind writes by milliseconds to seconds; UI must handle stale reads.
- **Application complexity**: developers must understand event sourcing, projection rebuilds, and idempotency.
- **Storage growth**: the event store grows indefinitely; snapshotting is required for aggregates with many events.
- **Schema evolution**: changing event schemas requires versioning and upcasting strategies.
- **Query flexibility**: ad-hoc queries against raw events are expensive; every new query pattern may need a new projection.

## Scalability Considerations

- **Event store partitioning**: partition by stream_type or date range to keep individual partition sizes manageable.
- **Projection rebuilds**: design projections to be fully rebuildable from events; use checkpoints for incremental rebuilds.
- **Event streaming**: events can be published to Kafka/NATS for real-time consumers (notification service, analytics, AI pipeline).
- **CQRS scaling**: read and write workloads scale independently; add read replicas for projections.

## Migration Path

If event sourcing proves too complex for certain bounded contexts (e.g. survey management), those can be implemented as traditional CRUD with events published as integration events. The compliance and engagement domains, where auditability is paramount, are the strongest candidates to retain full event sourcing. Projections can be progressively denormalized or replaced with search indexes (Elasticsearch) as query patterns evolve.
