# Data Model Suggestion 4: Graph-Relational Hybrid

> Project: Customer Journey Orchestration · Created: 2026-05-19

## Philosophy

This model adds a property graph layer to a relational core, recognising that customer journey orchestration is fundamentally a graph problem at two levels: (1) journey definitions are directed graphs of steps and transitions, and (2) customer-to-customer, customer-to-segment, and customer-to-journey relationships form a complex web that benefits from graph traversal queries. The relational tables handle operational CRUD (customer records, message sends, consent), while PostgreSQL's `ltree` extension and a lightweight graph edge table enable efficient traversal of journey paths, customer relationship networks, and segment overlap analysis.

Graph-relational hybrids are used by platforms that need both transactional integrity and relationship traversal. LinkedIn's economic graph, Facebook's social graph, and Neo4j-backed recommendation engines all separate graph traversal from transactional storage. In the CJO domain, graph queries answer questions that are expensive or impossible with pure relational joins: "Which journey paths lead to the highest conversion?", "Which customers overlap across three segments?", "What is the shortest path from first touch to conversion?", "Which journey steps are bottlenecks across all active journeys?"

This model uses PostgreSQL's `ltree` extension for hierarchical journey path tracking and a dedicated `graph_edges` table for arbitrary relationship traversal, avoiding the need for a separate graph database while enabling graph-style queries through recursive CTEs and ltree operators.

**Best for:** Platforms prioritising journey path analysis, segment overlap detection, customer network analysis, and complex "which path leads to conversion?" queries.

**Trade-offs:**
- (+) Journey path analysis is native — "most common path to conversion" queries are efficient
- (+) Segment overlap and customer relationship queries use graph traversal
- (+) ltree enables efficient hierarchical queries on journey paths without recursive CTEs
- (+) Single database (PostgreSQL) — no operational complexity of a separate graph database
- (+) Relational core provides ACID transactions for operational writes
- (-) PostgreSQL graph queries (recursive CTEs) are slower than native graph databases for deep traversals
- (-) ltree paths must be maintained when journey structures change
- (-) The `graph_edges` table can grow large in high-volume environments
- (-) More conceptual complexity — team needs to understand both relational and graph paradigms
- (-) GIN indexes on ltree columns add write overhead

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| CloudEvents v1.0.2 | Events table uses CloudEvents envelope columns for standards-compliant event ingestion |
| Segment Spec | Customer events conform to Segment track/identify/page/group schema |
| ISO/IEC 19510 (BPMN 2.0) | Journey graph directly models BPMN process definitions: nodes are BPMN activities/events/gateways, edges are sequence flows |
| Adobe XDM | Customer profile structure follows XDM Individual Profile field groups |
| Arazzo Specification | Journey edge conditions and step sequences map to Arazzo workflow step dependencies |
| GDPR / CCPA | Graph-based consent propagation — consent withdrawal traverses all related processing activities |

---

## Core Relational Tables

```sql
-- Required PostgreSQL extensions
CREATE EXTENSION IF NOT EXISTS ltree;
CREATE EXTENSION IF NOT EXISTS pg_trgm;

-- ============================================================
-- MULTI-TENANCY
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

CREATE TABLE org_members (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL,
    email           TEXT NOT NULL,
    role            TEXT NOT NULL DEFAULT 'viewer',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (org_id, user_id)
);

-- ============================================================
-- CUSTOMERS
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
    attributes      JSONB NOT NULL DEFAULT '{}',
    consent         JSONB NOT NULL DEFAULT '{}',
    last_seen_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (org_id, external_id)
);

CREATE INDEX idx_customers_org ON customers(org_id);
CREATE INDEX idx_customers_email ON customers(org_id, email);
CREATE INDEX idx_customers_attrs ON customers USING GIN (attributes);
```

## Graph Layer

```sql
-- ============================================================
-- UNIVERSAL GRAPH EDGE TABLE
-- ============================================================
-- This table models all relationships in the system as a property graph.
-- Nodes are rows in other tables (identified by node_type + node_id).
-- Edges carry typed relationships with optional properties.

CREATE TABLE graph_edges (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,

    -- Source node
    from_type       TEXT NOT NULL,                     -- customer, segment, journey, step, message, channel
    from_id         UUID NOT NULL,

    -- Target node
    to_type         TEXT NOT NULL,
    to_id           UUID NOT NULL,

    -- Edge metadata
    edge_type       TEXT NOT NULL,                     -- see Edge Type Catalog below
    properties      JSONB NOT NULL DEFAULT '{}',
    weight          REAL,                              -- optional numeric weight for scoring
    valid_from      TIMESTAMPTZ NOT NULL DEFAULT now(),
    valid_to        TIMESTAMPTZ,                       -- NULL = currently valid

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (org_id, from_type, from_id, to_type, to_id, edge_type, valid_from)
);

-- Query patterns: forward traversal, reverse traversal, by type
CREATE INDEX idx_ge_forward ON graph_edges(org_id, from_type, from_id, edge_type)
    WHERE valid_to IS NULL;
CREATE INDEX idx_ge_reverse ON graph_edges(org_id, to_type, to_id, edge_type)
    WHERE valid_to IS NULL;
CREATE INDEX idx_ge_edge_type ON graph_edges(org_id, edge_type)
    WHERE valid_to IS NULL;
CREATE INDEX idx_ge_temporal ON graph_edges(org_id, from_id, valid_from, valid_to);
```

### Edge Type Catalog

```
-- Customer relationships
MEMBER_OF_SEGMENT       customer → segment       (properties: {"entered_at": "..."})
ENROLLED_IN_JOURNEY     customer → journey        (properties: {"status": "active"})
AT_STEP                 customer → step           (properties: {"entered_at": "..."})
RECEIVED_MESSAGE        customer → message_send   (properties: {"channel": "email"})
BELONGS_TO_GROUP        customer → group          (properties: {"role": "admin"})
RELATED_TO_CUSTOMER     customer → customer       (properties: {"relationship": "same_company"})

-- Journey structure (graph representation of the journey canvas)
JOURNEY_CONTAINS_STEP   journey → step            (properties: {"position": {"x": 100, "y": 200}})
STEP_TRANSITIONS_TO     step → step               (properties: {"condition": {...}, "label": "Yes"})
STEP_USES_TEMPLATE      step → message_template   (properties: {"variant_weights": [50, 50]})
STEP_TARGETS_SEGMENT    step → segment            (properties: {})

-- Channel relationships
CUSTOMER_REACHABLE_VIA  customer → channel        (properties: {"identifier": "email@...", "verified": true})
TEMPLATE_FOR_CHANNEL    message_template → channel (properties: {})

-- Segment relationships
SEGMENT_OVERLAPS        segment → segment         (properties: {"overlap_count": 1250, "computed_at": "..."})
SEGMENT_SUBSET_OF       segment → segment         (properties: {"confidence": 0.95})
```

## Journey Definitions with ltree Paths

```sql
-- ============================================================
-- JOURNEYS
-- ============================================================

CREATE TABLE journeys (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    name            TEXT NOT NULL,
    description     TEXT,
    status          TEXT NOT NULL DEFAULT 'draft',
    version         INTEGER NOT NULL DEFAULT 1,
    parent_id       UUID REFERENCES journeys(id),
    entry_config    JSONB NOT NULL DEFAULT '{}',
    goal_config     JSONB,
    frequency_cap   JSONB,
    tags            TEXT[] DEFAULT '{}',
    created_by      UUID REFERENCES org_members(id),
    published_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_journeys_org ON journeys(org_id, status);

CREATE TABLE journey_steps (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    journey_id      UUID NOT NULL REFERENCES journeys(id) ON DELETE CASCADE,
    step_type       TEXT NOT NULL,
    name            TEXT NOT NULL,
    config          JSONB NOT NULL DEFAULT '{}',
    position_x      INTEGER,
    position_y      INTEGER,

    -- ltree path for hierarchical queries (e.g., "all steps after the decision node")
    tree_path       ltree,
    -- Example: 'root.welcome_email.wait_3d.check_opened.yes_path.send_followup'
    -- This enables queries like:
    --   SELECT * FROM journey_steps WHERE tree_path <@ 'root.welcome_email.wait_3d';
    --   (finds all steps under the wait_3d node)

    depth           INTEGER NOT NULL DEFAULT 0,

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_steps_journey ON journey_steps(journey_id);
CREATE INDEX idx_steps_tree ON journey_steps USING GIST (tree_path);

-- Journey edges are ALSO stored in graph_edges as STEP_TRANSITIONS_TO edges,
-- but we keep a denormalised table for fast journey rendering.
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
```

## Events & Tracking

```sql
-- ============================================================
-- EVENTS
-- ============================================================

CREATE TABLE events (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    customer_id     UUID REFERENCES customers(id) ON DELETE SET NULL,
    anonymous_id    TEXT,
    ce_source       TEXT NOT NULL,
    ce_type         TEXT NOT NULL,
    event_name      TEXT,
    properties      JSONB NOT NULL DEFAULT '{}',
    context         JSONB NOT NULL DEFAULT '{}',
    timestamp       TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (timestamp);

CREATE INDEX idx_events_org_customer ON events(org_id, customer_id, timestamp DESC);
CREATE INDEX idx_events_org_type ON events(org_id, ce_type, event_name);
```

## Journey Execution with Path Tracking

```sql
-- ============================================================
-- JOURNEY EXECUTION — with graph path tracking
-- ============================================================

CREATE TABLE journey_enrollments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    journey_id      UUID NOT NULL REFERENCES journeys(id) ON DELETE CASCADE,
    customer_id     UUID NOT NULL REFERENCES customers(id) ON DELETE CASCADE,
    current_step_id UUID REFERENCES journey_steps(id),
    status          TEXT NOT NULL DEFAULT 'active',

    -- Path taken through the journey (ordered list of step IDs)
    path_taken      UUID[] NOT NULL DEFAULT '{}',
    -- Example: ['uuid-step1', 'uuid-step2', 'uuid-step4']
    -- This enables path analysis: which routes through the journey lead to conversion?

    -- ltree representation of path for hierarchical queries
    path_ltree      ltree,
    -- Example: 'entry.welcome.wait.opened.followup'
    -- Enables: SELECT * FROM journey_enrollments WHERE path_ltree ~ '*.opened.*';

    entered_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    exited_at       TIMESTAMPTZ,
    exit_reason     TEXT,
    goal_met        BOOLEAN NOT NULL DEFAULT FALSE,
    conversion_value NUMERIC(12,2),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (org_id, journey_id, customer_id)
);

CREATE INDEX idx_enrollments_journey ON journey_enrollments(journey_id, status);
CREATE INDEX idx_enrollments_customer ON journey_enrollments(customer_id);
CREATE INDEX idx_enrollments_path ON journey_enrollments USING GIST (path_ltree);

-- ============================================================
-- MESSAGE SENDS
-- ============================================================

CREATE TABLE message_templates (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    channel_type    TEXT NOT NULL,
    name            TEXT NOT NULL,
    content         JSONB NOT NULL DEFAULT '{}',
    variants        JSONB NOT NULL DEFAULT '[]',
    version         INTEGER NOT NULL DEFAULT 1,
    status          TEXT NOT NULL DEFAULT 'draft',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE message_sends (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    customer_id     UUID NOT NULL REFERENCES customers(id) ON DELETE CASCADE,
    enrollment_id   UUID REFERENCES journey_enrollments(id),
    step_id         UUID REFERENCES journey_steps(id),
    template_id     UUID REFERENCES message_templates(id),
    channel_type    TEXT NOT NULL,
    status          TEXT NOT NULL DEFAULT 'queued',
    sent_at         TIMESTAMPTZ,
    delivered_at    TIMESTAMPTZ,
    opened_at       TIMESTAMPTZ,
    clicked_at      TIMESTAMPTZ,
    provider_id     TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_sends_customer ON message_sends(customer_id, created_at DESC);
CREATE INDEX idx_sends_enrollment ON message_sends(enrollment_id);

-- ============================================================
-- SEGMENTS
-- ============================================================

CREATE TABLE segments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    name            TEXT NOT NULL,
    description     TEXT,
    segment_type    TEXT NOT NULL DEFAULT 'dynamic',
    definition      JSONB NOT NULL DEFAULT '{}',
    estimated_size  BIGINT,
    status          TEXT NOT NULL DEFAULT 'draft',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Segment memberships are stored in BOTH the segment_memberships table
-- AND as MEMBER_OF_SEGMENT edges in graph_edges (for overlap analysis)
CREATE TABLE segment_memberships (
    org_id          UUID NOT NULL,
    segment_id      UUID NOT NULL REFERENCES segments(id) ON DELETE CASCADE,
    customer_id     UUID NOT NULL REFERENCES customers(id) ON DELETE CASCADE,
    entered_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (org_id, segment_id, customer_id)
);
```

## Consent, Compliance & Audit

```sql
-- ============================================================
-- CONSENT & COMPLIANCE
-- ============================================================

CREATE TABLE consent_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    customer_id     UUID NOT NULL REFERENCES customers(id) ON DELETE CASCADE,
    consent_type    TEXT NOT NULL,
    action          TEXT NOT NULL,
    details         JSONB NOT NULL DEFAULT '{}',
    occurred_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_consent_customer ON consent_log(customer_id, consent_type);

CREATE TABLE suppression_list (
    org_id          UUID NOT NULL,
    channel_type    TEXT NOT NULL,
    identifier      TEXT NOT NULL,
    reason          TEXT NOT NULL,
    suppressed_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (org_id, channel_type, identifier)
);

-- ============================================================
-- INTEGRATIONS
-- ============================================================

CREATE TABLE integrations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    provider        TEXT NOT NULL,
    name            TEXT NOT NULL,
    config          JSONB NOT NULL DEFAULT '{}',
    credentials     JSONB NOT NULL DEFAULT '{}',
    status          TEXT NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- AUDIT LOG
-- ============================================================

CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL,
    actor_id        UUID,
    action          TEXT NOT NULL,
    resource_type   TEXT NOT NULL,
    resource_id     UUID NOT NULL,
    details         JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

CREATE INDEX idx_audit_org ON audit_log(org_id, created_at DESC);
```

## Example: Graph Traversal Queries

### Find all segments a customer belongs to
```sql
SELECT s.id, s.name, ge.properties->>'entered_at' AS entered_at
FROM graph_edges ge
JOIN segments s ON s.id = ge.to_id
WHERE ge.org_id = '{{org_uuid}}'
  AND ge.from_type = 'customer'
  AND ge.from_id = '{{customer_uuid}}'
  AND ge.edge_type = 'MEMBER_OF_SEGMENT'
  AND ge.valid_to IS NULL;
```

### Find segment overlap (customers in both segment A and segment B)
```sql
SELECT ge1.from_id AS customer_id
FROM graph_edges ge1
JOIN graph_edges ge2
    ON ge1.from_id = ge2.from_id
    AND ge2.org_id = ge1.org_id
WHERE ge1.org_id = '{{org_uuid}}'
  AND ge1.edge_type = 'MEMBER_OF_SEGMENT'
  AND ge1.to_id = '{{segment_a_uuid}}'
  AND ge2.edge_type = 'MEMBER_OF_SEGMENT'
  AND ge2.to_id = '{{segment_b_uuid}}'
  AND ge1.valid_to IS NULL
  AND ge2.valid_to IS NULL;
```

### Find the most common journey paths leading to conversion
```sql
SELECT
    path_ltree::TEXT AS path,
    COUNT(*) AS conversions,
    AVG(conversion_value) AS avg_value
FROM journey_enrollments
WHERE org_id = '{{org_uuid}}'
  AND journey_id = '{{journey_uuid}}'
  AND goal_met = TRUE
GROUP BY path_ltree
ORDER BY conversions DESC
LIMIT 10;
```

### Recursive CTE: Find all customers reachable through 2-hop relationships
```sql
-- "Find customers related to customer X through shared segments"
WITH RECURSIVE related AS (
    -- Direct relationships from customer X
    SELECT
        ge.to_type, ge.to_id, ge.edge_type,
        1 AS depth,
        ARRAY[ge.from_id, ge.to_id] AS path
    FROM graph_edges ge
    WHERE ge.org_id = '{{org_uuid}}'
      AND ge.from_type = 'customer'
      AND ge.from_id = '{{customer_uuid}}'
      AND ge.valid_to IS NULL

    UNION ALL

    -- Second hop: from segments back to other customers
    SELECT
        ge2.to_type, ge2.to_id, ge2.edge_type,
        r.depth + 1,
        r.path || ge2.to_id
    FROM related r
    JOIN graph_edges ge2
        ON ge2.from_type = r.to_type
        AND ge2.from_id = r.to_id
        AND ge2.org_id = '{{org_uuid}}'
        AND ge2.valid_to IS NULL
    WHERE r.depth < 2
      AND NOT ge2.to_id = ANY(r.path)   -- prevent cycles
)
SELECT DISTINCT to_id AS related_customer_id
FROM related
WHERE to_type = 'customer'
  AND to_id != '{{customer_uuid}}';
```

### ltree: Find all journey steps downstream of a decision node
```sql
SELECT id, step_type, name, tree_path::TEXT
FROM journey_steps
WHERE journey_id = '{{journey_uuid}}'
  AND tree_path <@ (
      SELECT tree_path FROM journey_steps WHERE id = '{{decision_step_uuid}}'
  )
ORDER BY depth;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Multi-Tenancy | 2 | organisations, org_members |
| Customers | 1 | customers with JSONB attributes |
| Graph Layer | 1 | graph_edges (universal relationship table) |
| Events | 1 | events (partitioned) |
| Journeys | 3 | journeys, journey_steps (with ltree), journey_edges |
| Segments | 2 | segments, segment_memberships |
| Messages | 2 | message_templates, message_sends |
| Consent & Compliance | 2 | consent_log, suppression_list |
| Integrations | 1 | integrations |
| Audit | 1 | audit_log (partitioned) |
| **Total** | **16** | Plus graph_edges which represents ~15 edge types |

---

## Key Design Decisions

1. **Universal graph edge table** — rather than a separate table for every relationship type (segment_memberships, journey_enrollments_to_steps, customer_channels), a single `graph_edges` table models all relationships. The `edge_type` column distinguishes relationship semantics. This enables cross-cutting graph queries (e.g., "show me everything connected to this customer") that would require N separate JOINs in a pure relational model.

2. **Dual storage for hot paths** — segment memberships and journey edges are stored in BOTH dedicated relational tables (for fast operational queries) AND in `graph_edges` (for analytical graph traversal). This is intentional denormalisation: the relational tables serve the execution engine, while graph_edges serves analytics.

3. **ltree for journey path tracking** — PostgreSQL's `ltree` extension enables hierarchical queries on journey step trees and enrollment paths. The path `entry.welcome.wait.opened.followup` can be queried with operators like `<@` (is descendant of), `@>` (is ancestor of), and `~` (matches pattern). This is more efficient than recursive CTEs for common path analysis queries.

4. **path_taken array on enrollments** — each enrollment records the ordered array of step UUIDs the customer traversed. Combined with `path_ltree`, this enables both precise step-by-step replay (using the UUID array) and pattern-matching path analysis (using ltree).

5. **Temporal validity on graph edges** — `valid_from` and `valid_to` columns on `graph_edges` enable bi-temporal queries. When a customer exits a segment, the edge's `valid_to` is set rather than deleting the row. This preserves historical relationship data for analytics ("which segments was this customer in last quarter?").

6. **PostgreSQL-only graph** — rather than introducing Neo4j or a separate graph database, all graph queries use PostgreSQL recursive CTEs and ltree operators. This keeps the deployment to a single database, simplifying operations, transactions, and backup/restore. The trade-off is that deep graph traversals (>3 hops) will be slower than a native graph database.

7. **Edge properties as JSONB** — each graph edge carries a `properties` JSONB column for edge-specific data (e.g., the condition on a journey transition, the timestamp a customer entered a segment, the weight of an A/B split). This avoids needing separate edge tables for each relationship type.

8. **Weight column for scoring** — the optional `weight` column on `graph_edges` enables weighted graph algorithms (e.g., scoring the strength of customer-to-segment affinity, ranking journey paths by conversion probability). This is a foundation for AI-driven next-best-action recommendations.
