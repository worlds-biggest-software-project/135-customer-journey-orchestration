# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Customer Journey Orchestration · Created: 2026-05-19

## Philosophy

This model follows a traditional normalized relational design where every distinct concept receives its own table with explicit foreign keys and junction tables for many-to-many relationships. The schema enforces referential integrity at the database level, making it impossible for orphaned records or inconsistent state to persist. Every relationship is explicit, queryable, and indexable.

This approach mirrors how enterprise CRM and CDP systems (Salesforce, HubSpot) model their data internally — discrete tables for contacts, companies, campaigns, and activities joined through well-defined foreign keys. It is the natural fit for a platform where data integrity, complex cross-entity reporting, and regulatory compliance (GDPR right-to-erasure, CCPA data subject requests) are paramount.

The normalized model excels when the schema is well-understood upfront (as it is here — journey orchestration is a mature domain with clear entity boundaries), when complex joins across entities are frequent (e.g., "show me all customers in segment X who received message Y in journey Z and converted within 7 days"), and when the team values the safety net of database-enforced constraints.

**Best for:** Teams prioritising data integrity, complex cross-entity analytics, and regulatory compliance in a well-defined domain.

**Trade-offs:**
- (+) Full referential integrity — the database prevents invalid states
- (+) Straightforward GDPR/CCPA compliance — CASCADE DELETE from customer record
- (+) Complex cross-entity queries perform well with proper indexing
- (+) Well understood by most engineering teams; extensive tooling support
- (-) Higher table count (~45-50 tables) increases migration complexity
- (-) Schema changes require migrations — adding a new channel type means ALTER TABLE
- (-) Many-to-many junction tables add JOIN overhead for common queries
- (-) Less flexible for jurisdiction-specific or customer-specific custom fields

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| CloudEvents v1.0.2 | Event table columns map directly to CloudEvents envelope: `ce_id`, `ce_source`, `ce_type`, `ce_time`, `ce_specversion` |
| Segment Spec | `customer_events` table supports identify, track, page, group, screen event types with standardised field mapping |
| Adobe XDM | Customer profile and experience event separation mirrors XDM Individual Profile / ExperienceEvent class split |
| ISO/IEC 19510 (BPMN 2.0) | Journey definition tables model BPMN concepts: events → `journey_triggers`, activities → `journey_steps`, gateways → `journey_decisions` |
| ISO 3166-1/2 | `jurisdictions` reference table uses ISO 3166 codes for geographic targeting and consent regime mapping |
| IAB TCF v2.3 | `consent_records` table stores TCF consent strings alongside purpose-level consent flags |
| GDPR / CCPA | `data_subject_requests` table tracks erasure, access, and opt-out requests with processing status |
| OAuth 2.0 / OIDC | `integrations` and `api_credentials` tables store OAuth client credentials and token metadata |

---

## Core Identity & Multi-Tenancy

```sql
-- ============================================================
-- MULTI-TENANCY & ORGANISATION
-- ============================================================

CREATE TABLE organisations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL UNIQUE,          -- URL-safe identifier
    plan_tier       TEXT NOT NULL DEFAULT 'free',  -- free, pro, enterprise
    settings        JSONB NOT NULL DEFAULT '{}',   -- org-level config
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE org_members (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL,                 -- references auth provider (OIDC)
    email           TEXT NOT NULL,
    role            TEXT NOT NULL DEFAULT 'viewer', -- owner, admin, editor, viewer
    invited_at      TIMESTAMPTZ,
    accepted_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (org_id, user_id)
);

CREATE INDEX idx_org_members_org ON org_members(org_id);
CREATE INDEX idx_org_members_user ON org_members(user_id);
```

## Customer Profiles & Identity

```sql
-- ============================================================
-- CUSTOMER PROFILES & IDENTITY RESOLUTION
-- ============================================================

CREATE TABLE customers (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    external_id     TEXT,                          -- customer's ID in source system
    email           TEXT,
    phone           TEXT,
    first_name      TEXT,
    last_name       TEXT,
    timezone        TEXT,                          -- IANA timezone
    locale          TEXT,                          -- BCP 47 language tag
    country_code    CHAR(2),                       -- ISO 3166-1 alpha-2
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    last_seen_at    TIMESTAMPTZ,
    UNIQUE (org_id, external_id)
);

CREATE INDEX idx_customers_org ON customers(org_id);
CREATE INDEX idx_customers_email ON customers(org_id, email);
CREATE INDEX idx_customers_phone ON customers(org_id, phone);

CREATE TABLE customer_identities (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    customer_id     UUID NOT NULL REFERENCES customers(id) ON DELETE CASCADE,
    identity_type   TEXT NOT NULL,                 -- email, phone, device_id, anonymous_id, user_id
    identity_value  TEXT NOT NULL,
    verified        BOOLEAN NOT NULL DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (org_id, identity_type, identity_value)
);

CREATE INDEX idx_customer_identities_customer ON customer_identities(customer_id);

CREATE TABLE customer_attributes (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    customer_id     UUID NOT NULL REFERENCES customers(id) ON DELETE CASCADE,
    attribute_key   TEXT NOT NULL,
    attribute_value TEXT,
    attribute_type  TEXT NOT NULL DEFAULT 'string', -- string, number, boolean, date, json
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (org_id, customer_id, attribute_key)
);

CREATE INDEX idx_customer_attrs_lookup ON customer_attributes(org_id, attribute_key, attribute_value);
```

## Events & Tracking

```sql
-- ============================================================
-- EVENT INGESTION (CloudEvents / Segment Spec aligned)
-- ============================================================

CREATE TABLE customer_events (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    customer_id     UUID REFERENCES customers(id) ON DELETE SET NULL,
    anonymous_id    TEXT,                          -- pre-identification tracking

    -- CloudEvents envelope fields
    ce_source       TEXT NOT NULL,                 -- e.g. 'web', 'mobile-ios', 'crm-sync'
    ce_type         TEXT NOT NULL,                 -- e.g. 'track', 'identify', 'page', 'group'
    ce_specversion  TEXT NOT NULL DEFAULT '1.0',

    -- Segment Spec fields
    event_name      TEXT,                          -- e.g. 'Product Viewed', 'Order Completed'
    properties      JSONB NOT NULL DEFAULT '{}',   -- event-specific properties
    context         JSONB NOT NULL DEFAULT '{}',   -- device, locale, campaign, referrer
    timestamp       TIMESTAMPTZ NOT NULL DEFAULT now(),
    received_at     TIMESTAMPTZ NOT NULL DEFAULT now(),

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (timestamp);

CREATE INDEX idx_events_org_customer ON customer_events(org_id, customer_id, timestamp DESC);
CREATE INDEX idx_events_org_type ON customer_events(org_id, ce_type, event_name);
CREATE INDEX idx_events_properties ON customer_events USING GIN (properties);
```

## Segments & Audiences

```sql
-- ============================================================
-- AUDIENCE SEGMENTATION
-- ============================================================

CREATE TABLE segments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    name            TEXT NOT NULL,
    description     TEXT,
    segment_type    TEXT NOT NULL DEFAULT 'dynamic', -- dynamic, static, predictive
    definition      JSONB NOT NULL DEFAULT '{}',     -- segment rule definition (filter tree)
    estimated_size  BIGINT,
    last_computed   TIMESTAMPTZ,
    status          TEXT NOT NULL DEFAULT 'draft',   -- draft, active, archived
    created_by      UUID REFERENCES org_members(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_segments_org ON segments(org_id, status);

CREATE TABLE segment_memberships (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    segment_id      UUID NOT NULL REFERENCES segments(id) ON DELETE CASCADE,
    customer_id     UUID NOT NULL REFERENCES customers(id) ON DELETE CASCADE,
    entered_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    exited_at       TIMESTAMPTZ,
    UNIQUE (org_id, segment_id, customer_id)
);

CREATE INDEX idx_seg_membership_segment ON segment_memberships(segment_id) WHERE exited_at IS NULL;
CREATE INDEX idx_seg_membership_customer ON segment_memberships(customer_id);
```

## Journey Definitions

```sql
-- ============================================================
-- JOURNEY BUILDER (BPMN-aligned)
-- ============================================================

CREATE TABLE journeys (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    name            TEXT NOT NULL,
    description     TEXT,
    status          TEXT NOT NULL DEFAULT 'draft',    -- draft, active, paused, archived, completed
    version         INTEGER NOT NULL DEFAULT 1,
    parent_id       UUID REFERENCES journeys(id),     -- for versioning: points to previous version
    entry_type      TEXT NOT NULL DEFAULT 'segment',  -- segment, event, api, scheduled
    goal_type       TEXT,                             -- conversion, engagement, retention
    goal_definition JSONB,
    schedule        JSONB,                            -- cron expression, start/end dates
    frequency_cap   JSONB,                            -- max entries per customer per time window
    tags            TEXT[] DEFAULT '{}',
    created_by      UUID REFERENCES org_members(id),
    published_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_journeys_org_status ON journeys(org_id, status);

CREATE TABLE journey_triggers (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    journey_id      UUID NOT NULL REFERENCES journeys(id) ON DELETE CASCADE,
    trigger_type    TEXT NOT NULL,                    -- segment_entry, event, schedule, api_call, segment_exit
    trigger_config  JSONB NOT NULL DEFAULT '{}',      -- event name filters, segment reference, schedule
    priority        INTEGER NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_journey_triggers_journey ON journey_triggers(journey_id);

CREATE TABLE journey_steps (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    journey_id      UUID NOT NULL REFERENCES journeys(id) ON DELETE CASCADE,
    step_type       TEXT NOT NULL,                    -- action, decision, delay, split, exit
    name            TEXT NOT NULL,
    config          JSONB NOT NULL DEFAULT '{}',      -- step-type-specific configuration
    position_x      INTEGER,                          -- canvas coordinates for UI
    position_y      INTEGER,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_journey_steps_journey ON journey_steps(journey_id);

CREATE TABLE journey_edges (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    journey_id      UUID NOT NULL REFERENCES journeys(id) ON DELETE CASCADE,
    from_step_id    UUID NOT NULL REFERENCES journey_steps(id) ON DELETE CASCADE,
    to_step_id      UUID NOT NULL REFERENCES journey_steps(id) ON DELETE CASCADE,
    condition       JSONB,                            -- branch condition (null = default path)
    label           TEXT,                             -- e.g. 'Yes', 'No', '50%'
    sort_order      INTEGER NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (journey_id, from_step_id, to_step_id)
);

CREATE INDEX idx_journey_edges_from ON journey_edges(from_step_id);
CREATE INDEX idx_journey_edges_to ON journey_edges(to_step_id);
```

## Channels & Messages

```sql
-- ============================================================
-- CHANNELS & MESSAGE TEMPLATES
-- ============================================================

CREATE TABLE channels (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    channel_type    TEXT NOT NULL,                    -- email, sms, push, in_app, webhook, web
    name            TEXT NOT NULL,
    provider        TEXT NOT NULL,                    -- sendgrid, twilio, firebase, custom
    config          JSONB NOT NULL DEFAULT '{}',      -- API keys, sender IDs, etc. (encrypted at app layer)
    status          TEXT NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_channels_org ON channels(org_id, channel_type);

CREATE TABLE message_templates (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    channel_type    TEXT NOT NULL,                    -- email, sms, push, in_app, webhook
    name            TEXT NOT NULL,
    subject         TEXT,                             -- email subject line
    body            TEXT NOT NULL,                    -- template body (supports Liquid/Handlebars)
    preview_text    TEXT,                             -- email preview text
    from_name       TEXT,
    from_address    TEXT,
    reply_to        TEXT,
    metadata        JSONB NOT NULL DEFAULT '{}',      -- channel-specific metadata
    version         INTEGER NOT NULL DEFAULT 1,
    status          TEXT NOT NULL DEFAULT 'draft',    -- draft, active, archived
    created_by      UUID REFERENCES org_members(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_templates_org ON message_templates(org_id, channel_type, status);

CREATE TABLE message_variants (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    template_id     UUID NOT NULL REFERENCES message_templates(id) ON DELETE CASCADE,
    variant_name    TEXT NOT NULL,                    -- 'A', 'B', 'Control'
    subject         TEXT,
    body            TEXT NOT NULL,
    weight          INTEGER NOT NULL DEFAULT 50,      -- A/B split percentage
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_variants_template ON message_variants(template_id);
```

## Journey Execution

```sql
-- ============================================================
-- JOURNEY EXECUTION & CUSTOMER STATE
-- ============================================================

CREATE TABLE journey_enrollments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    journey_id      UUID NOT NULL REFERENCES journeys(id) ON DELETE CASCADE,
    customer_id     UUID NOT NULL REFERENCES customers(id) ON DELETE CASCADE,
    current_step_id UUID REFERENCES journey_steps(id),
    status          TEXT NOT NULL DEFAULT 'active',   -- active, completed, exited, errored
    entered_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    exited_at       TIMESTAMPTZ,
    exit_reason     TEXT,                             -- completed, goal_met, suppressed, error, manual
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (org_id, journey_id, customer_id)          -- one enrollment per journey per customer
);

CREATE INDEX idx_enrollments_journey ON journey_enrollments(journey_id, status);
CREATE INDEX idx_enrollments_customer ON journey_enrollments(customer_id);
CREATE INDEX idx_enrollments_step ON journey_enrollments(current_step_id) WHERE status = 'active';

CREATE TABLE journey_step_executions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    enrollment_id   UUID NOT NULL REFERENCES journey_enrollments(id) ON DELETE CASCADE,
    step_id         UUID NOT NULL REFERENCES journey_steps(id) ON DELETE CASCADE,
    status          TEXT NOT NULL DEFAULT 'pending',  -- pending, executing, completed, failed, skipped
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    result          JSONB,                            -- step execution output
    error_message   TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_step_exec_enrollment ON journey_step_executions(enrollment_id);
CREATE INDEX idx_step_exec_step ON journey_step_executions(step_id);

CREATE TABLE message_sends (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    enrollment_id   UUID REFERENCES journey_enrollments(id),
    step_execution_id UUID REFERENCES journey_step_executions(id),
    customer_id     UUID NOT NULL REFERENCES customers(id) ON DELETE CASCADE,
    template_id     UUID REFERENCES message_templates(id),
    variant_id      UUID REFERENCES message_variants(id),
    channel_id      UUID REFERENCES channels(id),
    channel_type    TEXT NOT NULL,
    status          TEXT NOT NULL DEFAULT 'queued',   -- queued, sent, delivered, bounced, failed
    sent_at         TIMESTAMPTZ,
    delivered_at    TIMESTAMPTZ,
    provider_id     TEXT,                             -- external message ID from provider
    provider_response JSONB,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_sends_customer ON message_sends(customer_id, created_at DESC);
CREATE INDEX idx_sends_enrollment ON message_sends(enrollment_id);
CREATE INDEX idx_sends_status ON message_sends(org_id, status) WHERE status IN ('queued', 'sent');
```

## Consent & Compliance

```sql
-- ============================================================
-- CONSENT & PRIVACY COMPLIANCE
-- ============================================================

CREATE TABLE consent_records (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    customer_id     UUID NOT NULL REFERENCES customers(id) ON DELETE CASCADE,
    consent_type    TEXT NOT NULL,                    -- marketing_email, marketing_sms, marketing_push, tcf
    status          TEXT NOT NULL,                    -- granted, denied, withdrawn
    purpose         TEXT,                             -- GDPR purpose description
    tcf_string      TEXT,                            -- IAB TCF v2.3 consent string if applicable
    source          TEXT NOT NULL,                    -- web_form, api, import, preference_center
    ip_address      INET,
    user_agent      TEXT,
    granted_at      TIMESTAMPTZ,
    withdrawn_at    TIMESTAMPTZ,
    expires_at      TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_consent_customer ON consent_records(customer_id, consent_type);
CREATE INDEX idx_consent_org ON consent_records(org_id, consent_type, status);

CREATE TABLE suppression_lists (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    channel_type    TEXT NOT NULL,                    -- email, sms, push
    identifier      TEXT NOT NULL,                    -- email address, phone number, device token
    reason          TEXT NOT NULL,                    -- unsubscribed, bounced, complained, manual
    suppressed_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (org_id, channel_type, identifier)
);

CREATE INDEX idx_suppression_lookup ON suppression_lists(org_id, channel_type, identifier);

CREATE TABLE data_subject_requests (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    customer_id     UUID REFERENCES customers(id),
    request_type    TEXT NOT NULL,                    -- access, erasure, portability, opt_out_sale
    regulation      TEXT NOT NULL,                    -- gdpr, ccpa, pdpa
    status          TEXT NOT NULL DEFAULT 'pending',  -- pending, processing, completed, rejected
    requested_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at    TIMESTAMPTZ,
    response        JSONB,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_dsr_org ON data_subject_requests(org_id, status);
```

## Analytics & A/B Testing

```sql
-- ============================================================
-- ANALYTICS, GOALS & A/B TESTING
-- ============================================================

CREATE TABLE message_interactions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    message_send_id UUID NOT NULL REFERENCES message_sends(id) ON DELETE CASCADE,
    customer_id     UUID NOT NULL REFERENCES customers(id) ON DELETE CASCADE,
    interaction_type TEXT NOT NULL,                   -- open, click, reply, unsubscribe, complaint
    url             TEXT,                             -- clicked URL
    metadata        JSONB NOT NULL DEFAULT '{}',
    occurred_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (occurred_at);

CREATE INDEX idx_interactions_send ON message_interactions(message_send_id);
CREATE INDEX idx_interactions_customer ON message_interactions(customer_id, occurred_at DESC);

CREATE TABLE journey_conversions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    journey_id      UUID NOT NULL REFERENCES journeys(id) ON DELETE CASCADE,
    enrollment_id   UUID NOT NULL REFERENCES journey_enrollments(id) ON DELETE CASCADE,
    customer_id     UUID NOT NULL REFERENCES customers(id) ON DELETE CASCADE,
    conversion_event TEXT NOT NULL,                   -- event name that triggered conversion
    conversion_value NUMERIC(12,2),                   -- monetary value if applicable
    currency        CHAR(3),                          -- ISO 4217
    attributed_step_id UUID REFERENCES journey_steps(id),
    converted_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_conversions_journey ON journey_conversions(journey_id, converted_at DESC);

CREATE TABLE ab_experiments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    journey_id      UUID REFERENCES journeys(id),
    name            TEXT NOT NULL,
    experiment_type TEXT NOT NULL,                    -- message_variant, journey_path, send_time
    status          TEXT NOT NULL DEFAULT 'draft',    -- draft, running, completed, stopped
    config          JSONB NOT NULL DEFAULT '{}',      -- variant weights, confidence threshold
    winner_variant  TEXT,
    started_at      TIMESTAMPTZ,
    ended_at        TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_experiments_org ON ab_experiments(org_id, status);
```

## Integrations

```sql
-- ============================================================
-- EXTERNAL INTEGRATIONS
-- ============================================================

CREATE TABLE integrations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    integration_type TEXT NOT NULL,                   -- crm, cdp, data_warehouse, channel_provider
    provider        TEXT NOT NULL,                    -- salesforce, segment, snowflake, sendgrid
    name            TEXT NOT NULL,
    status          TEXT NOT NULL DEFAULT 'active',
    config          JSONB NOT NULL DEFAULT '{}',      -- non-sensitive config
    credentials     JSONB NOT NULL DEFAULT '{}',      -- encrypted at app layer
    last_sync_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_integrations_org ON integrations(org_id, integration_type);

CREATE TABLE webhooks (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    name            TEXT NOT NULL,
    url             TEXT NOT NULL,
    secret          TEXT,                             -- HMAC signing secret
    events          TEXT[] NOT NULL,                  -- event types to subscribe to
    status          TEXT NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_webhooks_org ON webhooks(org_id, status);
```

## Audit Log

```sql
-- ============================================================
-- AUDIT LOG
-- ============================================================

CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    actor_id        UUID,                             -- org_member who performed action
    actor_type      TEXT NOT NULL DEFAULT 'user',     -- user, system, api, webhook
    action          TEXT NOT NULL,                     -- created, updated, deleted, published, activated
    resource_type   TEXT NOT NULL,                     -- journey, segment, template, customer
    resource_id     UUID NOT NULL,
    changes         JSONB,                            -- {field: {old: x, new: y}}
    ip_address      INET,
    user_agent      TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

CREATE INDEX idx_audit_org ON audit_log(org_id, created_at DESC);
CREATE INDEX idx_audit_resource ON audit_log(resource_type, resource_id);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Multi-Tenancy & Organisation | 2 | organisations, org_members |
| Customer Profiles & Identity | 3 | customers, customer_identities, customer_attributes |
| Events & Tracking | 1 | customer_events (partitioned) |
| Segments & Audiences | 2 | segments, segment_memberships |
| Journey Definitions | 4 | journeys, journey_triggers, journey_steps, journey_edges |
| Channels & Messages | 3 | channels, message_templates, message_variants |
| Journey Execution | 3 | journey_enrollments, journey_step_executions, message_sends |
| Consent & Compliance | 3 | consent_records, suppression_lists, data_subject_requests |
| Analytics & A/B Testing | 3 | message_interactions, journey_conversions, ab_experiments |
| Integrations | 2 | integrations, webhooks |
| Audit | 1 | audit_log (partitioned) |
| **Total** | **27** | |

---

## Key Design Decisions

1. **Tenant isolation via `org_id` column** — every tenant-scoped table includes `org_id` as the leading column in composite indexes, enabling PostgreSQL Row-Level Security (RLS) policies and efficient tenant-scoped queries without schema-per-tenant complexity.

2. **Journey graph as adjacency list** — `journey_steps` + `journey_edges` models the journey canvas as a directed graph. This maps directly to BPMN 2.0 concepts (activities + sequence flows) and supports arbitrary branching, loops, and parallel paths. The adjacency list is the simplest graph representation and works well for journeys with dozens (not millions) of nodes.

3. **Events table partitioned by timestamp** — `customer_events` is range-partitioned by `timestamp` to support efficient time-range queries and automated partition management (drop old partitions for retention). High-volume event ingestion benefits from partition pruning.

4. **CloudEvents envelope fields on event table** — rather than wrapping events in a separate envelope table, the CloudEvents required fields (`ce_source`, `ce_type`, `ce_specversion`) are columns on `customer_events`, enabling direct CloudEvents-compatible event ingestion without transformation.

5. **Separate consent table with temporal records** — each consent grant/withdrawal is a new row (not an update), providing a complete consent audit trail as required by GDPR Article 7(1) and IAB TCF v2.3. The latest consent state is derived by querying the most recent record per customer per consent type.

6. **EAV pattern for custom customer attributes** — `customer_attributes` uses an entity-attribute-value pattern to support arbitrary per-customer properties without schema changes. This trades query performance (requires JOIN to filter on custom attributes) for flexibility.

7. **Message send tracking separated from interactions** — `message_sends` tracks delivery state (queued → sent → delivered → bounced) while `message_interactions` tracks engagement events (open, click, unsubscribe). This separation supports different retention policies and query patterns.

8. **Audit log as append-only partitioned table** — the `audit_log` captures all administrative changes with before/after state in JSONB, partitioned by time for efficient retention management. This supports GDPR audit requirements and SOC 2 compliance evidence.
