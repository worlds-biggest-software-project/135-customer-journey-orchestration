# Data Model Suggestion 2: Event-Sourced / Audit-First (CQRS)

> Project: Customer Journey Orchestration · Created: 2026-05-19

## Philosophy

This model treats the event store as the single source of truth. Every state change — a customer entering a journey, a message being sent, a consent grant, a segment membership change — is recorded as an immutable event in an append-only store. Current state is derived by replaying events or, more practically, by maintaining materialised read models (projections) optimised for specific query patterns. This is the Command Query Responsibility Segregation (CQRS) pattern.

Event sourcing is a natural fit for customer journey orchestration because the domain is inherently event-driven: customer actions trigger journey steps, journey steps produce messages, messages generate interactions (opens, clicks), and interactions feed back into decisioning. The entire lifecycle is a stream of events. By capturing every event immutably, the platform gains full temporal queryability ("what was true about this customer on March 15?"), complete audit trails (GDPR Article 30 compliance), and the ability to replay history for debugging, analytics, or rebuilding read models.

Real-world systems using this pattern include Kafka-backed streaming architectures (as used by Braze and Adobe Experience Platform for real-time event processing), EventStoreDB-based CQRS systems, and financial ledger systems where immutability is a regulatory requirement.

**Best for:** Platforms requiring full audit trails, temporal queries, real-time streaming, and the ability to derive multiple read-optimised views from the same event stream.

**Trade-offs:**
- (+) Complete, immutable audit trail — every state change is preserved forever
- (+) Temporal queries are native — "show me this customer's state at any point in time"
- (+) Read models can be independently optimised for different query patterns
- (+) Event replay enables debugging, analytics reprocessing, and new projection creation
- (+) Natural fit for real-time streaming architectures (Kafka, Pulsar)
- (-) Higher storage costs — events accumulate indefinitely
- (-) Eventual consistency between write (event store) and read (projections) models
- (-) More complex to implement — requires event handlers, projectors, and snapshotting
- (-) Simple queries require read model infrastructure; cannot just "SELECT * FROM customers"
- (-) Schema evolution of events requires careful versioning (upcasting)

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| CloudEvents v1.0.2 | Every event in the store conforms to the CloudEvents envelope: `specversion`, `id`, `source`, `type`, `time`, `data` |
| Segment Spec | Inbound customer events (identify, track, page, group) are stored as CloudEvents with Segment-compatible `type` and `data` fields |
| Adobe XDM ExperienceEvent | The event schema draws from XDM's timestamped fact-record pattern — each event is a point-in-time observation |
| ISO/IEC 19510 (BPMN 2.0) | Journey definition events model BPMN elements; journey execution events track process instance state transitions |
| GDPR Article 30 | The event store IS the record of processing activities — every data access, consent change, and message send is an event |
| IAB TCF v2.3 | Consent events store the full TCF consent string as event data, with the consent state derived from event replay |

---

## Event Store (Write Side)

```sql
-- ============================================================
-- CORE EVENT STORE — the single source of truth
-- ============================================================

CREATE TABLE event_store (
    -- Event identity
    event_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_id       UUID NOT NULL,                    -- aggregate root ID (customer, journey, etc.)
    stream_type     TEXT NOT NULL,                     -- 'customer', 'journey', 'segment', 'message'
    sequence_num    BIGINT NOT NULL,                   -- per-stream ordering guarantee

    -- CloudEvents envelope
    ce_specversion  TEXT NOT NULL DEFAULT '1.0',
    ce_source       TEXT NOT NULL,                     -- producing service/module
    ce_type         TEXT NOT NULL,                     -- event type (see Event Catalog below)
    ce_time         TIMESTAMPTZ NOT NULL DEFAULT now(),
    ce_datacontenttype TEXT NOT NULL DEFAULT 'application/json',

    -- Tenant & causation
    org_id          UUID NOT NULL,
    correlation_id  UUID,                              -- links related events across streams
    causation_id    UUID,                              -- the event that caused this event
    actor_id        UUID,                              -- user or system that triggered it
    actor_type      TEXT NOT NULL DEFAULT 'system',    -- user, system, api, scheduler

    -- Payload
    data            JSONB NOT NULL,                    -- event-specific payload
    metadata        JSONB NOT NULL DEFAULT '{}',       -- observability: ip, user_agent, trace_id

    -- Constraints
    UNIQUE (stream_id, sequence_num)
) PARTITION BY RANGE (ce_time);

-- Primary query patterns
CREATE INDEX idx_es_stream ON event_store(stream_id, sequence_num);
CREATE INDEX idx_es_org_type ON event_store(org_id, ce_type, ce_time DESC);
CREATE INDEX idx_es_correlation ON event_store(correlation_id) WHERE correlation_id IS NOT NULL;
CREATE INDEX idx_es_org_time ON event_store(org_id, ce_time DESC);

-- ============================================================
-- SNAPSHOTS — periodic state snapshots for fast replay
-- ============================================================

CREATE TABLE event_snapshots (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_id       UUID NOT NULL,
    stream_type     TEXT NOT NULL,
    sequence_num    BIGINT NOT NULL,                   -- snapshot taken at this sequence number
    state           JSONB NOT NULL,                    -- serialised aggregate state
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (stream_id, sequence_num)
);

CREATE INDEX idx_snapshots_stream ON event_snapshots(stream_id, sequence_num DESC);
```

### Event Catalog

The `ce_type` field follows a hierarchical naming convention: `{domain}.{entity}.{action}`.

```
-- Customer domain
customer.profile.created
customer.profile.updated
customer.profile.merged
customer.profile.deleted
customer.identity.linked
customer.identity.unlinked
customer.attribute.set
customer.attribute.removed
customer.consent.granted
customer.consent.withdrawn
customer.suppression.added
customer.suppression.removed

-- Tracking domain (Segment Spec aligned)
tracking.event.received          -- raw inbound event (track, page, identify, group)

-- Journey domain
journey.definition.created
journey.definition.updated
journey.definition.published
journey.definition.paused
journey.definition.archived
journey.step.added
journey.step.updated
journey.step.removed
journey.edge.added
journey.edge.removed

-- Journey execution domain
journey.enrollment.started
journey.enrollment.step_entered
journey.enrollment.step_completed
journey.enrollment.step_failed
journey.enrollment.decision_evaluated
journey.enrollment.delay_started
journey.enrollment.delay_completed
journey.enrollment.completed
journey.enrollment.exited
journey.enrollment.goal_met

-- Messaging domain
message.send.queued
message.send.dispatched
message.send.delivered
message.send.bounced
message.send.failed
message.interaction.opened
message.interaction.clicked
message.interaction.replied
message.interaction.unsubscribed
message.interaction.complained

-- Segment domain
segment.definition.created
segment.definition.updated
segment.definition.archived
segment.membership.entered
segment.membership.exited
segment.computation.completed

-- Experiment domain
experiment.created
experiment.started
experiment.stopped
experiment.winner_declared

-- Integration domain
integration.configured
integration.sync.started
integration.sync.completed
integration.sync.failed

-- Admin domain
admin.member.invited
admin.member.accepted
admin.member.role_changed
admin.member.removed
```

## Read Models (Projections)

```sql
-- ============================================================
-- READ MODEL: Customer Profiles (projected from customer.* events)
-- ============================================================

CREATE TABLE rm_customers (
    id              UUID PRIMARY KEY,
    org_id          UUID NOT NULL,
    external_id     TEXT,
    email           TEXT,
    phone           TEXT,
    first_name      TEXT,
    last_name       TEXT,
    timezone        TEXT,
    locale          TEXT,
    country_code    CHAR(2),
    attributes      JSONB NOT NULL DEFAULT '{}',
    identities      JSONB NOT NULL DEFAULT '[]',
    consent_state   JSONB NOT NULL DEFAULT '{}',
    -- Example consent_state:
    -- {
    --   "marketing_email": {"status": "granted", "updated_at": "2026-05-01T10:00:00Z"},
    --   "marketing_sms": {"status": "denied", "updated_at": "2026-04-15T08:30:00Z"},
    --   "tcf": {"string": "CPx...", "updated_at": "2026-05-10T12:00:00Z"}
    -- }
    last_seen_at    TIMESTAMPTZ,
    last_event_seq  BIGINT NOT NULL DEFAULT 0,        -- watermark for projection
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_rm_customers_org ON rm_customers(org_id);
CREATE INDEX idx_rm_customers_email ON rm_customers(org_id, email);
CREATE INDEX idx_rm_customers_attrs ON rm_customers USING GIN (attributes);

-- ============================================================
-- READ MODEL: Journey Definitions (projected from journey.definition.* events)
-- ============================================================

CREATE TABLE rm_journeys (
    id              UUID PRIMARY KEY,
    org_id          UUID NOT NULL,
    name            TEXT NOT NULL,
    description     TEXT,
    status          TEXT NOT NULL,
    version         INTEGER NOT NULL,
    entry_type      TEXT NOT NULL,
    goal_definition JSONB,
    steps           JSONB NOT NULL DEFAULT '[]',
    -- Example steps:
    -- [
    --   {"id": "uuid", "type": "action", "name": "Send welcome email", "config": {...}},
    --   {"id": "uuid", "type": "decision", "name": "Opened?", "config": {...}}
    -- ]
    edges           JSONB NOT NULL DEFAULT '[]',
    -- Example edges:
    -- [
    --   {"from": "uuid", "to": "uuid", "condition": {...}, "label": "Yes"},
    --   {"from": "uuid", "to": "uuid", "condition": null, "label": "No"}
    -- ]
    triggers        JSONB NOT NULL DEFAULT '[]',
    frequency_cap   JSONB,
    tags            TEXT[],
    published_at    TIMESTAMPTZ,
    last_event_seq  BIGINT NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_rm_journeys_org ON rm_journeys(org_id, status);

-- ============================================================
-- READ MODEL: Active Journey Enrollments (projected from journey.enrollment.* events)
-- ============================================================

CREATE TABLE rm_enrollments (
    id              UUID PRIMARY KEY,
    org_id          UUID NOT NULL,
    journey_id      UUID NOT NULL,
    customer_id     UUID NOT NULL,
    current_step_id UUID,
    status          TEXT NOT NULL,                     -- active, completed, exited, errored
    entered_at      TIMESTAMPTZ NOT NULL,
    exited_at       TIMESTAMPTZ,
    exit_reason     TEXT,
    steps_completed INTEGER NOT NULL DEFAULT 0,
    last_event_seq  BIGINT NOT NULL DEFAULT 0,
    updated_at      TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_rm_enrollments_journey ON rm_enrollments(journey_id, status);
CREATE INDEX idx_rm_enrollments_customer ON rm_enrollments(customer_id);
CREATE UNIQUE INDEX idx_rm_enrollments_active ON rm_enrollments(org_id, journey_id, customer_id)
    WHERE status = 'active';

-- ============================================================
-- READ MODEL: Message Delivery Status (projected from message.* events)
-- ============================================================

CREATE TABLE rm_message_sends (
    id              UUID PRIMARY KEY,
    org_id          UUID NOT NULL,
    enrollment_id   UUID,
    customer_id     UUID NOT NULL,
    channel_type    TEXT NOT NULL,
    template_id     UUID,
    variant_id      UUID,
    status          TEXT NOT NULL,                     -- queued, dispatched, delivered, bounced, failed
    sent_at         TIMESTAMPTZ,
    delivered_at    TIMESTAMPTZ,
    opened_at       TIMESTAMPTZ,                      -- first open
    clicked_at      TIMESTAMPTZ,                      -- first click
    open_count      INTEGER NOT NULL DEFAULT 0,
    click_count     INTEGER NOT NULL DEFAULT 0,
    provider_id     TEXT,
    last_event_seq  BIGINT NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_rm_sends_customer ON rm_message_sends(customer_id, created_at DESC);
CREATE INDEX idx_rm_sends_org ON rm_message_sends(org_id, channel_type, created_at DESC);

-- ============================================================
-- READ MODEL: Segment Memberships (projected from segment.membership.* events)
-- ============================================================

CREATE TABLE rm_segment_memberships (
    org_id          UUID NOT NULL,
    segment_id      UUID NOT NULL,
    customer_id     UUID NOT NULL,
    entered_at      TIMESTAMPTZ NOT NULL,
    last_event_seq  BIGINT NOT NULL DEFAULT 0,
    PRIMARY KEY (org_id, segment_id, customer_id)
);

CREATE INDEX idx_rm_segmem_customer ON rm_segment_memberships(customer_id);

-- ============================================================
-- READ MODEL: Analytics Aggregates (projected from multiple event types)
-- ============================================================

CREATE TABLE rm_journey_daily_stats (
    org_id          UUID NOT NULL,
    journey_id      UUID NOT NULL,
    stat_date       DATE NOT NULL,
    enrollments     BIGINT NOT NULL DEFAULT 0,
    completions     BIGINT NOT NULL DEFAULT 0,
    exits           BIGINT NOT NULL DEFAULT 0,
    goals_met       BIGINT NOT NULL DEFAULT 0,
    messages_sent   BIGINT NOT NULL DEFAULT 0,
    messages_delivered BIGINT NOT NULL DEFAULT 0,
    opens           BIGINT NOT NULL DEFAULT 0,
    clicks          BIGINT NOT NULL DEFAULT 0,
    unsubscribes    BIGINT NOT NULL DEFAULT 0,
    conversions     BIGINT NOT NULL DEFAULT 0,
    conversion_value NUMERIC(14,2) NOT NULL DEFAULT 0,
    last_event_seq  BIGINT NOT NULL DEFAULT 0,
    PRIMARY KEY (org_id, journey_id, stat_date)
);

-- ============================================================
-- READ MODEL: Suppression Lists (projected from customer.suppression.* events)
-- ============================================================

CREATE TABLE rm_suppressions (
    org_id          UUID NOT NULL,
    channel_type    TEXT NOT NULL,
    identifier      TEXT NOT NULL,
    reason          TEXT NOT NULL,
    suppressed_at   TIMESTAMPTZ NOT NULL,
    PRIMARY KEY (org_id, channel_type, identifier)
);
```

## Configuration Tables (Non-Event-Sourced)

```sql
-- ============================================================
-- REFERENCE / CONFIGURATION (not event-sourced — static config)
-- ============================================================

CREATE TABLE organisations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL UNIQUE,
    plan_tier       TEXT NOT NULL DEFAULT 'free',
    settings        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE channels (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
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
    org_id          UUID NOT NULL REFERENCES organisations(id),
    channel_type    TEXT NOT NULL,
    name            TEXT NOT NULL,
    subject         TEXT,
    body            TEXT NOT NULL,
    preview_text    TEXT,
    metadata        JSONB NOT NULL DEFAULT '{}',
    version         INTEGER NOT NULL DEFAULT 1,
    status          TEXT NOT NULL DEFAULT 'draft',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE integrations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    integration_type TEXT NOT NULL,
    provider        TEXT NOT NULL,
    name            TEXT NOT NULL,
    config          JSONB NOT NULL DEFAULT '{}',
    credentials     JSONB NOT NULL DEFAULT '{}',
    status          TEXT NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- PROJECTION TRACKING — tracks which events each projector has processed
-- ============================================================

CREATE TABLE projection_checkpoints (
    projection_name TEXT PRIMARY KEY,                  -- e.g. 'rm_customers', 'rm_enrollments'
    last_event_id   UUID NOT NULL,
    last_event_time TIMESTAMPTZ NOT NULL,
    last_sequence   BIGINT NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Example: Temporal Query (Event Replay)

```sql
-- "What was this customer's consent state on March 15, 2026?"
-- Replay all consent events for the customer up to that date:

SELECT
    data->>'consent_type' AS consent_type,
    data->>'status' AS status,
    ce_time AS changed_at
FROM event_store
WHERE stream_id = '{{customer_uuid}}'
  AND ce_type IN ('customer.consent.granted', 'customer.consent.withdrawn')
  AND ce_time <= '2026-03-15T23:59:59Z'
ORDER BY ce_time DESC;

-- To get the definitive state at that point, use DISTINCT ON:

SELECT DISTINCT ON (data->>'consent_type')
    data->>'consent_type' AS consent_type,
    data->>'status' AS status,
    ce_time AS as_of
FROM event_store
WHERE stream_id = '{{customer_uuid}}'
  AND ce_type IN ('customer.consent.granted', 'customer.consent.withdrawn')
  AND ce_time <= '2026-03-15T23:59:59Z'
ORDER BY data->>'consent_type', ce_time DESC;
```

## Example: Journey Execution Replay

```sql
-- "Show me the complete history of this customer's journey enrollment"

SELECT
    ce_type AS event,
    ce_time AS occurred_at,
    data->>'step_id' AS step_id,
    data->>'step_name' AS step_name,
    data->>'status' AS status,
    data->>'result' AS result
FROM event_store
WHERE correlation_id = '{{enrollment_uuid}}'
ORDER BY ce_time ASC;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store (Write) | 2 | event_store (partitioned), event_snapshots |
| Read Model: Customers | 1 | rm_customers |
| Read Model: Journeys | 1 | rm_journeys |
| Read Model: Execution | 2 | rm_enrollments, rm_message_sends |
| Read Model: Segments | 1 | rm_segment_memberships |
| Read Model: Analytics | 1 | rm_journey_daily_stats |
| Read Model: Compliance | 1 | rm_suppressions |
| Configuration | 4 | organisations, channels, message_templates, integrations |
| Infrastructure | 1 | projection_checkpoints |
| **Total** | **14** | Plus ~50 event types in the event catalog |

---

## Key Design Decisions

1. **Single event store table** — all events across all aggregate types go into one `event_store` table, partitioned by time. This simplifies infrastructure (one table to back up, monitor, and partition) and enables cross-aggregate correlation queries. The `stream_type` column allows filtering to a specific domain when needed.

2. **CloudEvents-native envelope** — every event conforms to the CloudEvents v1.0 specification. The `ce_*` prefix columns store envelope fields, and `data` holds the event-specific payload. This means events can be published to external systems (Kafka, EventBridge) without transformation.

3. **Correlation and causation tracking** — `correlation_id` links all events in a business process (e.g., all events from one journey enrollment share a correlation ID). `causation_id` records which event triggered this event, enabling causal chain reconstruction for debugging.

4. **Projections own their read schema** — each read model table is independently designed for its query pattern. `rm_customers` denormalises identities and consent into JSONB for single-row reads. `rm_journey_daily_stats` pre-aggregates for dashboard queries. Projections can be rebuilt from scratch by replaying the event store.

5. **Snapshot table for fast aggregate loading** — rather than replaying all events from the beginning of time, the `event_snapshots` table stores periodic state snapshots. To load a customer's current state, load the latest snapshot and replay only events after it.

6. **Configuration tables are NOT event-sourced** — organisations, channels, templates, and integrations are managed via normal CRUD. Event-sourcing everything adds unnecessary complexity for entities that don't benefit from temporal queries or audit replay.

7. **Projection checkpoint tracking** — `projection_checkpoints` records the last event processed by each projector, enabling exactly-once processing semantics and allowing projectors to resume from where they left off after restarts.

8. **Event type naming convention** — the hierarchical `{domain}.{entity}.{action}` naming enables wildcard subscriptions (e.g., subscribe to `customer.*` for all customer events) and clear event catalog documentation. This aligns with CloudEvents type extension recommendations.
