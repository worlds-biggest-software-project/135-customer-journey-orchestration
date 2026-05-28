# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: Customer Journey Orchestration · Created: 2026-05-19

## Philosophy

This model combines relational tables for well-defined structural entities (organisations, journeys, customers) with JSONB columns for variable, domain-specific, or rapidly evolving data. The relational skeleton provides referential integrity and efficient joins for core queries, while JSONB columns absorb the variability that would otherwise require dozens of junction tables, EAV patterns, or frequent schema migrations.

This is the pragmatic middle ground used by many modern SaaS platforms. HubSpot, for example, models contacts and companies as relational entities but stores custom properties as flexible key-value structures. Braze stores user attributes and event properties as schemaless JSON. The pattern works especially well for customer journey orchestration because: (a) customer attributes vary wildly across organisations (an e-commerce company tracks cart value; a bank tracks account tier), (b) journey step configurations differ by step type (email steps need subject lines; delay steps need durations; decision steps need condition trees), and (c) event properties are inherently schemaless (every Track event has different properties).

The key insight is that not all data benefits equally from normalisation. Core relationships (customer-to-journey, journey-to-steps, message-to-customer) are relational. Variable payloads (event properties, step configurations, customer attributes, segment rule definitions) are JSONB. PostgreSQL's GIN indexes on JSONB columns make this approach queryable without sacrificing too much performance.

**Best for:** Teams wanting rapid iteration, MVP speed, and flexibility for multi-tenant variation without sacrificing core relational integrity.

**Trade-offs:**
- (+) Fewer tables (~20) — less migration overhead and simpler operational management
- (+) Custom attributes, event properties, and step configs need no schema changes
- (+) JSONB GIN indexes support containment queries for filtering on dynamic fields
- (+) Faster time-to-market — new features often just add keys to JSONB columns
- (+) Natural fit for per-tenant customisation (different orgs store different attributes)
- (-) JSONB fields lack database-enforced constraints — validation must happen at application layer
- (-) GIN index queries on JSONB are slower than B-tree indexes on dedicated columns
- (-) Reporting on JSONB fields requires JSON path extraction, which complicates SQL
- (-) Risk of "JSONB sprawl" — without discipline, JSONB columns become undocumented bags of data
- (-) Partial indexes on JSONB paths are possible but less ergonomic than column-level indexes

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| CloudEvents v1.0.2 | Event `envelope` JSONB column stores the full CloudEvents envelope; `data` JSONB stores payload |
| Segment Spec | `events.data` JSONB conforms to Segment identify/track/page/group schemas — no rigid column mapping needed |
| Adobe XDM | Customer profile `attributes` JSONB can be structured to follow XDM field group conventions without table proliferation |
| ISO/IEC 19510 (BPMN 2.0) | Journey steps store BPMN-aligned type semantics; step `config` JSONB varies by BPMN element type |
| ISO 3166-1/2 | Jurisdiction codes stored in customer `attributes` JSONB or dedicated columns depending on query frequency |
| IAB TCF v2.3 | Consent `details` JSONB stores TCF string and purpose-level consent alongside simple status columns |
| GDPR / CCPA | Privacy fields use relational columns for queryable status; request details in JSONB for flexibility |

---

## Schema Definition

```sql
-- ============================================================
-- MULTI-TENANCY
-- ============================================================

CREATE TABLE organisations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL UNIQUE,
    plan_tier       TEXT NOT NULL DEFAULT 'free',
    settings        JSONB NOT NULL DEFAULT '{}',
    -- Example settings:
    -- {
    --   "default_timezone": "America/New_York",
    --   "default_locale": "en-US",
    --   "features": {"ai_journey_design": true, "predictive_send_time": false},
    --   "branding": {"primary_color": "#4F46E5", "logo_url": "..."}
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE org_members (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL,
    email           TEXT NOT NULL,
    role            TEXT NOT NULL DEFAULT 'viewer',
    permissions     JSONB NOT NULL DEFAULT '[]',
    -- Example permissions override:
    -- ["journey:publish", "segment:create", "template:edit"]
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (org_id, user_id)
);

CREATE INDEX idx_org_members_org ON org_members(org_id);

-- ============================================================
-- CUSTOMERS — relational core + JSONB attributes
-- ============================================================

CREATE TABLE customers (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    external_id     TEXT,
    email           TEXT,
    phone           TEXT,
    first_name      TEXT,
    last_name       TEXT,
    timezone        TEXT,
    locale          TEXT,
    country_code    CHAR(2),

    -- Flexible attributes (custom properties per org)
    attributes      JSONB NOT NULL DEFAULT '{}',
    -- Example attributes for e-commerce org:
    -- {
    --   "plan": "premium",
    --   "ltv": 1250.00,
    --   "signup_source": "google_ads",
    --   "preferred_channel": "email",
    --   "last_purchase_date": "2026-04-22",
    --   "cart_value": 89.99,
    --   "tags": ["vip", "early_adopter"]
    -- }

    -- Identity resolution
    identities      JSONB NOT NULL DEFAULT '[]',
    -- Example identities:
    -- [
    --   {"type": "email", "value": "jane@example.com", "verified": true},
    --   {"type": "device_id", "value": "abc123", "verified": false},
    --   {"type": "anonymous_id", "value": "anon-xyz", "verified": false}
    -- ]

    -- Consent state (denormalised for fast lookup)
    consent         JSONB NOT NULL DEFAULT '{}',
    -- Example consent:
    -- {
    --   "marketing_email": {"status": "granted", "at": "2026-05-01T10:00:00Z", "source": "preference_center"},
    --   "marketing_sms": {"status": "denied", "at": "2026-04-15T08:30:00Z", "source": "signup_form"},
    --   "tcf": {"string": "CPx...", "at": "2026-05-10T12:00:00Z"}
    -- }

    last_seen_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (org_id, external_id)
);

CREATE INDEX idx_customers_org ON customers(org_id);
CREATE INDEX idx_customers_email ON customers(org_id, email);
CREATE INDEX idx_customers_phone ON customers(org_id, phone);
CREATE INDEX idx_customers_attrs ON customers USING GIN (attributes);
CREATE INDEX idx_customers_consent ON customers USING GIN (consent);

-- ============================================================
-- EVENTS — lightweight relational envelope + JSONB payload
-- ============================================================

CREATE TABLE events (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    customer_id     UUID REFERENCES customers(id) ON DELETE SET NULL,
    anonymous_id    TEXT,

    -- Typed columns for high-frequency filters
    event_type      TEXT NOT NULL,                     -- identify, track, page, group, screen
    event_name      TEXT,                              -- e.g. 'Product Viewed' (for track events)
    source          TEXT NOT NULL,                     -- web, ios, android, server, import
    channel         TEXT,                              -- email, sms, push, in_app, web

    -- Full CloudEvents + Segment payload
    data            JSONB NOT NULL DEFAULT '{}',
    -- Example data for a 'track' event:
    -- {
    --   "properties": {
    --     "product_id": "SKU-123",
    --     "product_name": "Running Shoes",
    --     "price": 129.99,
    --     "currency": "USD",
    --     "category": "Footwear"
    --   },
    --   "context": {
    --     "page": {"url": "https://shop.example.com/shoes/123", "title": "Running Shoes"},
    --     "campaign": {"source": "google", "medium": "cpc", "name": "spring_sale"},
    --     "device": {"type": "mobile", "manufacturer": "Apple"},
    --     "locale": "en-US"
    --   }
    -- }

    timestamp       TIMESTAMPTZ NOT NULL DEFAULT now(),
    received_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (timestamp);

CREATE INDEX idx_events_org_customer ON events(org_id, customer_id, timestamp DESC);
CREATE INDEX idx_events_org_type ON events(org_id, event_type, event_name, timestamp DESC);
CREATE INDEX idx_events_data ON events USING GIN (data);

-- ============================================================
-- SEGMENTS — rule definition as JSONB filter tree
-- ============================================================

CREATE TABLE segments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    name            TEXT NOT NULL,
    description     TEXT,
    segment_type    TEXT NOT NULL DEFAULT 'dynamic',

    -- Filter tree (rules engine definition)
    definition      JSONB NOT NULL DEFAULT '{}',
    -- Example definition:
    -- {
    --   "operator": "AND",
    --   "conditions": [
    --     {"field": "attributes.plan", "op": "eq", "value": "premium"},
    --     {"field": "attributes.ltv", "op": "gte", "value": 500},
    --     {
    --       "operator": "OR",
    --       "conditions": [
    --         {"field": "events.Product Viewed", "op": "count_gte", "value": 3, "window": "30d"},
    --         {"field": "attributes.tags", "op": "contains", "value": "vip"}
    --       ]
    --     }
    --   ]
    -- }

    estimated_size  BIGINT,
    last_computed   TIMESTAMPTZ,
    status          TEXT NOT NULL DEFAULT 'draft',
    created_by      UUID REFERENCES org_members(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_segments_org ON segments(org_id, status);

CREATE TABLE segment_memberships (
    org_id          UUID NOT NULL,
    segment_id      UUID NOT NULL REFERENCES segments(id) ON DELETE CASCADE,
    customer_id     UUID NOT NULL REFERENCES customers(id) ON DELETE CASCADE,
    entered_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    exited_at       TIMESTAMPTZ,
    PRIMARY KEY (org_id, segment_id, customer_id)
);

-- ============================================================
-- JOURNEYS — relational structure, JSONB step configs
-- ============================================================

CREATE TABLE journeys (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    name            TEXT NOT NULL,
    description     TEXT,
    status          TEXT NOT NULL DEFAULT 'draft',
    version         INTEGER NOT NULL DEFAULT 1,
    parent_id       UUID REFERENCES journeys(id),

    -- Entry and goal configuration
    entry_config    JSONB NOT NULL DEFAULT '{}',
    -- Example entry_config:
    -- {
    --   "type": "segment_entry",
    --   "segment_id": "uuid",
    --   "re_entry": false,
    --   "rate_limit": {"max": 1000, "per": "hour"}
    -- }

    goal_config     JSONB,
    -- Example goal_config:
    -- {
    --   "event": "Order Completed",
    --   "window": "7d",
    --   "value_field": "properties.revenue"
    -- }

    schedule        JSONB,
    frequency_cap   JSONB,
    tags            TEXT[] DEFAULT '{}',

    -- Canvas layout (entire visual state)
    canvas          JSONB NOT NULL DEFAULT '{}',
    -- Example canvas:
    -- {
    --   "viewport": {"x": 0, "y": 0, "zoom": 1.0},
    --   "grid_snap": true
    -- }

    created_by      UUID REFERENCES org_members(id),
    published_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_journeys_org ON journeys(org_id, status);

CREATE TABLE journey_steps (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    journey_id      UUID NOT NULL REFERENCES journeys(id) ON DELETE CASCADE,
    step_type       TEXT NOT NULL,                     -- action_email, action_sms, action_push,
                                                       -- action_webhook, decision, delay, ab_split, exit
    name            TEXT NOT NULL,

    -- Type-specific configuration in JSONB
    config          JSONB NOT NULL DEFAULT '{}',
    -- Example config for action_email:
    -- {
    --   "template_id": "uuid",
    --   "variant_ids": ["uuid-a", "uuid-b"],
    --   "send_time_optimization": true,
    --   "suppress_if_sent_within": "24h"
    -- }
    --
    -- Example config for decision:
    -- {
    --   "condition": {"field": "events.Email Opened", "op": "occurred", "window": "3d"},
    --   "timeout": "72h",
    --   "timeout_path": "no"
    -- }
    --
    -- Example config for delay:
    -- {
    --   "duration": "2d",
    --   "until_time": "09:00",
    --   "timezone_source": "customer"
    -- }

    position_x      INTEGER,
    position_y      INTEGER,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_steps_journey ON journey_steps(journey_id);

CREATE TABLE journey_edges (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    journey_id      UUID NOT NULL REFERENCES journeys(id) ON DELETE CASCADE,
    from_step_id    UUID NOT NULL REFERENCES journey_steps(id) ON DELETE CASCADE,
    to_step_id      UUID NOT NULL REFERENCES journey_steps(id) ON DELETE CASCADE,
    condition       JSONB,
    label           TEXT,
    sort_order      INTEGER NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (journey_id, from_step_id, to_step_id)
);

CREATE INDEX idx_edges_from ON journey_edges(from_step_id);

-- ============================================================
-- CHANNELS & TEMPLATES
-- ============================================================

CREATE TABLE channels (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    channel_type    TEXT NOT NULL,
    name            TEXT NOT NULL,
    provider        TEXT NOT NULL,
    config          JSONB NOT NULL DEFAULT '{}',
    status          TEXT NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE message_templates (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    channel_type    TEXT NOT NULL,
    name            TEXT NOT NULL,
    content         JSONB NOT NULL DEFAULT '{}',
    -- Example content for email:
    -- {
    --   "subject": "Welcome to {{org.name}}, {{customer.first_name}}!",
    --   "body_html": "<html>...",
    --   "body_text": "Welcome...",
    --   "preview_text": "Your journey starts here",
    --   "from_name": "{{org.name}}",
    --   "from_address": "hello@example.com",
    --   "reply_to": "support@example.com"
    -- }
    --
    -- Example content for push:
    -- {
    --   "title": "New offer for you!",
    --   "body": "Check out our spring collection",
    --   "image_url": "https://...",
    --   "deep_link": "app://products/spring",
    --   "badge_count": 1
    -- }
    variants        JSONB NOT NULL DEFAULT '[]',
    -- Example variants:
    -- [
    --   {"id": "uuid-a", "name": "A", "weight": 50, "content": {"subject": "Alt subject..."}},
    --   {"id": "uuid-b", "name": "B", "weight": 50, "content": {"subject": "Another subject..."}}
    -- ]
    version         INTEGER NOT NULL DEFAULT 1,
    status          TEXT NOT NULL DEFAULT 'draft',
    created_by      UUID REFERENCES org_members(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_templates_org ON message_templates(org_id, channel_type, status);

-- ============================================================
-- JOURNEY EXECUTION
-- ============================================================

CREATE TABLE journey_enrollments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    journey_id      UUID NOT NULL REFERENCES journeys(id) ON DELETE CASCADE,
    customer_id     UUID NOT NULL REFERENCES customers(id) ON DELETE CASCADE,
    current_step_id UUID REFERENCES journey_steps(id),
    status          TEXT NOT NULL DEFAULT 'active',
    context         JSONB NOT NULL DEFAULT '{}',
    -- Example context (accumulated during journey):
    -- {
    --   "trigger_event": {"name": "Signup Completed", "properties": {...}},
    --   "steps_completed": ["uuid-1", "uuid-2"],
    --   "messages_sent": 2,
    --   "last_decision": {"step": "uuid-3", "result": "yes"}
    -- }
    entered_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    exited_at       TIMESTAMPTZ,
    exit_reason     TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (org_id, journey_id, customer_id)
);

CREATE INDEX idx_enrollments_journey ON journey_enrollments(journey_id, status);
CREATE INDEX idx_enrollments_customer ON journey_enrollments(customer_id);

CREATE TABLE message_sends (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    customer_id     UUID NOT NULL REFERENCES customers(id) ON DELETE CASCADE,
    enrollment_id   UUID REFERENCES journey_enrollments(id),
    template_id     UUID REFERENCES message_templates(id),
    channel_type    TEXT NOT NULL,
    status          TEXT NOT NULL DEFAULT 'queued',

    -- Delivery and engagement tracking
    delivery        JSONB NOT NULL DEFAULT '{}',
    -- Example delivery:
    -- {
    --   "provider": "sendgrid",
    --   "provider_id": "msg-abc123",
    --   "sent_at": "2026-05-19T10:00:00Z",
    --   "delivered_at": "2026-05-19T10:00:02Z",
    --   "opened_at": "2026-05-19T14:30:00Z",
    --   "clicked_at": "2026-05-19T14:30:15Z",
    --   "open_count": 3,
    --   "click_count": 1,
    --   "clicked_urls": ["https://shop.example.com/spring"],
    --   "bounced": false
    -- }

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_sends_customer ON message_sends(customer_id, created_at DESC);
CREATE INDEX idx_sends_org_status ON message_sends(org_id, status) WHERE status IN ('queued', 'sent');

-- ============================================================
-- CONSENT & COMPLIANCE
-- ============================================================

CREATE TABLE consent_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    customer_id     UUID NOT NULL REFERENCES customers(id) ON DELETE CASCADE,
    consent_type    TEXT NOT NULL,
    action          TEXT NOT NULL,                     -- granted, withdrawn
    details         JSONB NOT NULL DEFAULT '{}',
    -- Example details:
    -- {
    --   "tcf_string": "CPx...",
    --   "purpose": "Marketing communications via email",
    --   "source": "preference_center",
    --   "ip_address": "203.0.113.42",
    --   "user_agent": "Mozilla/5.0..."
    -- }
    occurred_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_consent_log_customer ON consent_log(customer_id, consent_type);

CREATE TABLE suppression_list (
    org_id          UUID NOT NULL,
    channel_type    TEXT NOT NULL,
    identifier      TEXT NOT NULL,
    reason          TEXT NOT NULL,
    suppressed_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (org_id, channel_type, identifier)
);

-- ============================================================
-- INTEGRATIONS & WEBHOOKS
-- ============================================================

CREATE TABLE integrations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    provider        TEXT NOT NULL,
    name            TEXT NOT NULL,
    config          JSONB NOT NULL DEFAULT '{}',
    credentials     JSONB NOT NULL DEFAULT '{}',
    sync_state      JSONB NOT NULL DEFAULT '{}',
    -- Example sync_state:
    -- {
    --   "last_sync": "2026-05-19T06:00:00Z",
    --   "records_synced": 15420,
    --   "errors": [],
    --   "cursor": "abc123"
    -- }
    status          TEXT NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- ANALYTICS AGGREGATES
-- ============================================================

CREATE TABLE journey_stats (
    org_id          UUID NOT NULL,
    journey_id      UUID NOT NULL REFERENCES journeys(id) ON DELETE CASCADE,
    stat_date       DATE NOT NULL,
    metrics         JSONB NOT NULL DEFAULT '{}',
    -- Example metrics:
    -- {
    --   "enrollments": 1250,
    --   "completions": 890,
    --   "exits": 120,
    --   "goals_met": 340,
    --   "messages": {"sent": 3200, "delivered": 3150, "opened": 1800, "clicked": 450},
    --   "conversions": {"count": 340, "value": 42500.00, "currency": "USD"},
    --   "by_channel": {
    --     "email": {"sent": 2000, "opened": 1200, "clicked": 300},
    --     "push": {"sent": 1200, "opened": 600, "clicked": 150}
    --   }
    -- }
    PRIMARY KEY (org_id, journey_id, stat_date)
);

-- ============================================================
-- AUDIT LOG
-- ============================================================

CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL,
    actor_id        UUID,
    actor_type      TEXT NOT NULL DEFAULT 'user',
    action          TEXT NOT NULL,
    resource_type   TEXT NOT NULL,
    resource_id     UUID NOT NULL,
    details         JSONB NOT NULL DEFAULT '{}',
    -- Example details:
    -- {
    --   "changes": {"status": {"from": "draft", "to": "active"}},
    --   "ip": "203.0.113.42",
    --   "user_agent": "Mozilla/5.0..."
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

CREATE INDEX idx_audit_org ON audit_log(org_id, created_at DESC);
CREATE INDEX idx_audit_resource ON audit_log(resource_type, resource_id);
```

## Example: JSONB Containment Query for Segment Evaluation

```sql
-- Find all premium customers with LTV >= 500 who have email consent
SELECT id, email, first_name, last_name, attributes
FROM customers
WHERE org_id = '{{org_uuid}}'
  AND attributes @> '{"plan": "premium"}'
  AND (attributes->>'ltv')::numeric >= 500
  AND consent @> '{"marketing_email": {"status": "granted"}}';
```

## Example: Flexible Event Property Query

```sql
-- Find all 'Product Viewed' events for a specific product category in the last 30 days
SELECT e.id, e.customer_id, e.timestamp,
       e.data->'properties'->>'product_name' AS product,
       (e.data->'properties'->>'price')::numeric AS price
FROM events e
WHERE e.org_id = '{{org_uuid}}'
  AND e.event_type = 'track'
  AND e.event_name = 'Product Viewed'
  AND e.data @> '{"properties": {"category": "Footwear"}}'
  AND e.timestamp > now() - INTERVAL '30 days'
ORDER BY e.timestamp DESC;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Multi-Tenancy | 2 | organisations, org_members |
| Customers | 1 | Single table with JSONB attributes, identities, consent |
| Events | 1 | Partitioned, with JSONB data payload |
| Segments | 2 | segments (JSONB definition), segment_memberships |
| Journeys | 3 | journeys, journey_steps (JSONB config), journey_edges |
| Channels & Templates | 2 | channels, message_templates (JSONB content + variants) |
| Execution | 2 | journey_enrollments (JSONB context), message_sends (JSONB delivery) |
| Consent & Compliance | 2 | consent_log, suppression_list |
| Integrations | 1 | integrations (JSONB config, credentials, sync_state) |
| Analytics | 1 | journey_stats (JSONB metrics) |
| Audit | 1 | audit_log (partitioned, JSONB details) |
| **Total** | **18** | |

---

## Key Design Decisions

1. **Customer attributes, identities, and consent in JSONB columns** — instead of three separate tables (customer_attributes EAV, customer_identities, consent_records), all three are JSONB columns on the `customers` table. This trades normalisation for single-row customer reads — the most common query pattern in journey execution (load customer, check consent, evaluate attributes).

2. **Message template content as JSONB** — different channel types (email, SMS, push, in-app) have fundamentally different content fields. Rather than separate tables per channel or nullable columns, a single `content` JSONB column holds channel-specific fields. Variants are embedded as a JSONB array, eliminating a separate table.

3. **Journey step config as JSONB** — step types (email action, SMS action, decision, delay, A/B split) each require different configuration. The `config` JSONB column absorbs this variability. The `step_type` column tells the application layer which schema to expect in `config`.

4. **Analytics metrics as JSONB** — `journey_stats.metrics` stores a rich nested structure of daily metrics. This allows adding new metric types (e.g., a new channel) without schema migration. Pre-aggregated daily stats avoid expensive real-time aggregation queries.

5. **Consent log for audit + denormalised consent on customer** — the `consent_log` table provides the full audit trail of consent changes (required by GDPR). The `consent` JSONB column on `customers` is the denormalised current state for fast lookup during message dispatch. Both are maintained in sync by the application layer.

6. **Delivery tracking as JSONB on message_sends** — rather than a separate `message_interactions` table, delivery and engagement events (sent, delivered, opened, clicked) are aggregated into a `delivery` JSONB column. This works well for typical dashboard queries ("show me open rate") but sacrifices per-interaction granularity (individual click timestamps with URLs).

7. **GIN indexes on all JSONB columns** — PostgreSQL GIN indexes support the `@>` containment operator, enabling efficient queries like `attributes @> '{"plan": "premium"}'`. For high-frequency filter paths, the application layer can create targeted partial indexes.

8. **18 tables total** — roughly 40% fewer tables than the fully normalised model. This reduces migration complexity, ORM configuration, and the cognitive load of understanding the schema. The trade-off is that JSONB validation must be enforced at the application layer.
