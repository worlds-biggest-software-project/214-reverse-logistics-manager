# Data Model Suggestion 4: Graph-Relational Hybrid

> Project: Reverse Logistics Manager · Created: 2026-05-20

## Philosophy

This model combines a relational core for operational CRUD with a property graph layer for relationship-heavy queries that are natural to reverse logistics: product-to-disposition path analysis, consumer fraud network detection, supply chain partner relationships, and multi-hop traceability ("which recycler processed items from which returns from which consumers?"). The graph layer uses PostgreSQL's `ltree` extension and a generic `graph_edge` table rather than requiring a separate graph database like Neo4j.

Reverse logistics is inherently a network problem. A returned item flows through a web of relationships: consumer -> order -> return -> inspection -> disposition -> partner -> recommerce channel. Fraud detection requires traversing consumer networks (shared addresses, shared payment methods, coordinated return patterns). Sustainability reporting requires tracing the full lifecycle from product manufacture through sale, return, and final disposition. These are graph traversal problems that are awkward in a normalised relational model but natural in a graph.

The key insight is that most day-to-day operations (create an RMA, record an inspection, generate a label) are standard CRUD that work perfectly in relational tables. But the high-value analytical queries — fraud network detection, disposition path optimisation, lifecycle traceability — benefit enormously from a graph layer. This model gives you both.

**Best for:** Platforms where fraud network analysis, supply chain traceability, disposition path optimisation, and circular economy lifecycle tracking are strategic priorities.

**Trade-offs:**
- (+) Fraud network detection via graph traversal (shared addresses, coordinated returns)
- (+) Full product lifecycle traceability for sustainability/DPP reporting
- (+) Disposition path analysis: which routes yield the best recovery rates?
- (+) Flexible relationship modeling — new edge types without schema changes
- (+) ltree enables hierarchical queries (product categories, org structures) without recursive CTEs
- (-) Two data layers (relational + graph) increase complexity
- (-) Graph edges must be kept in sync with relational foreign keys
- (-) Developers need familiarity with graph query patterns
- (-) PostgreSQL ltree/graph is less performant than dedicated graph databases for very large traversals
- (-) More moving parts than a pure relational or pure JSONB model

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| GS1 GTIN | Relational `product.gtin` column; graph node for product lifecycle traversal |
| GS1 GRAI | Relational column; graph edges link returnable assets to shipments and returns |
| GS1 Digital Link | Product node property for QR-code-based tracking |
| ISO 3166-1/2 | Jurisdiction hierarchy modeled via ltree: `earth.na.us.ca` |
| ISO 4217 | Currency fields on monetary columns |
| WEEE Directive | Graph edges link items to certified recyclers; compliance properties on disposition nodes |
| R2v3 / e-Stewards | Certification properties on disposition partner nodes |
| EU Digital Product Passport | Product lifecycle graph enables DPP data assembly from manufacture through disposal |
| GS1 EPCIS | Graph structure aligns with EPCIS event model: what, when, where, why for each lifecycle step |

---

## Relational Core (Operational CRUD)

```sql
-- Enable ltree extension for hierarchical data
CREATE EXTENSION IF NOT EXISTS ltree;

-- Tenant
CREATE TABLE tenant (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    type            VARCHAR(50) NOT NULL,
    default_currency VARCHAR(3) NOT NULL DEFAULT 'USD',
    settings        JSONB NOT NULL DEFAULT '{}',
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE app_user (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    email           VARCHAR(255) NOT NULL,
    full_name       VARCHAR(255) NOT NULL,
    role            VARCHAR(50) NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, email)
);

-- Product with category hierarchy via ltree
CREATE TABLE product (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    sku             VARCHAR(100) NOT NULL,
    gtin            VARCHAR(14),
    name            VARCHAR(500) NOT NULL,
    category_path   LTREE,                        -- e.g., 'electronics.phones.smartphones'
    brand           VARCHAR(255),
    unit_cost       NUMERIC(12,2),
    unit_price      NUMERIC(12,2),
    currency        VARCHAR(3) DEFAULT 'USD',
    weee_category   VARCHAR(50),
    attrs           JSONB NOT NULL DEFAULT '{}',
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, sku)
);

CREATE INDEX idx_product_gtin ON product (gtin) WHERE gtin IS NOT NULL;
CREATE INDEX idx_product_category ON product USING GIST (category_path);

-- Consumer
CREATE TABLE consumer (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    external_customer_id VARCHAR(255),
    email           VARCHAR(255) NOT NULL,
    full_name       VARCHAR(255),
    phone           VARCHAR(50),
    fraud_risk_score NUMERIC(5,2),
    profile         JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, email)
);

-- Original order
CREATE TABLE original_order (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    consumer_id     UUID NOT NULL REFERENCES consumer(id),
    external_order_id VARCHAR(255) NOT NULL,
    order_date      TIMESTAMPTZ NOT NULL,
    order_total     NUMERIC(12,2) NOT NULL,
    currency        VARCHAR(3) DEFAULT 'USD',
    channel         VARCHAR(50),
    platform        VARCHAR(50),
    order_data      JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, external_order_id)
);

-- Return request
CREATE TABLE return_request (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    rma_number      VARCHAR(50) NOT NULL,
    consumer_id     UUID NOT NULL REFERENCES consumer(id),
    original_order_id UUID NOT NULL REFERENCES original_order(id),
    status          VARCHAR(50) NOT NULL DEFAULT 'requested',
    return_type     VARCHAR(50) NOT NULL DEFAULT 'return',
    resolution_type VARCHAR(50),
    initiated_via   VARCHAR(50) DEFAULT 'portal',
    fraud_risk_score NUMERIC(5,2),
    refund_amount   NUMERIC(12,2),
    store_credit_amount NUMERIC(12,2),
    currency        VARCHAR(3) DEFAULT 'USD',
    approved_at     TIMESTAMPTZ,
    received_at     TIMESTAMPTZ,
    resolved_at     TIMESTAMPTZ,
    return_data     JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, rma_number)
);

CREATE INDEX idx_return_request_status ON return_request (tenant_id, status);
CREATE INDEX idx_return_request_consumer ON return_request (consumer_id);

-- Return item
CREATE TABLE return_item (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    return_request_id UUID NOT NULL REFERENCES return_request(id),
    tenant_id       UUID NOT NULL,
    product_id      UUID REFERENCES product(id),
    sku             VARCHAR(100) NOT NULL,
    quantity        INTEGER NOT NULL DEFAULT 1,
    unit_price      NUMERIC(12,2) NOT NULL,
    reason_code     VARCHAR(100) NOT NULL,
    serial_number   VARCHAR(255),
    condition_grade VARCHAR(20),
    disposition_type VARCHAR(50),
    item_data       JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_return_item_request ON return_item (return_request_id);

-- Disposition partner
CREATE TABLE disposition_partner (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(255) NOT NULL,
    partner_type    VARCHAR(50) NOT NULL,
    certification_type VARCHAR(50),
    certification_expiry DATE,
    partner_data    JSONB NOT NULL DEFAULT '{}',
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Return shipment
CREATE TABLE return_shipment (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    return_request_id UUID NOT NULL REFERENCES return_request(id),
    carrier         VARCHAR(100) NOT NULL,
    tracking_number VARCHAR(255),
    label_url       VARCHAR(1000),
    status          VARCHAR(50) NOT NULL DEFAULT 'pending',
    shipping_cost   NUMERIC(12,2),
    shipment_data   JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Return policy
CREATE TABLE return_policy (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(255) NOT NULL,
    policy_def      JSONB NOT NULL,
    is_default      BOOLEAN NOT NULL DEFAULT FALSE,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Integration config
CREATE TABLE integration (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    platform        VARCHAR(50) NOT NULL,
    integration_type VARCHAR(50) NOT NULL,
    config          JSONB NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Graph Layer

```sql
-- ============================================================
-- GRAPH NODE: Every entity that participates in relationships
-- ============================================================
CREATE TABLE graph_node (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    node_type       VARCHAR(50) NOT NULL,          -- consumer, order, return, item, product, partner, shipment, location
    entity_id       UUID NOT NULL,                 -- FK to the relational table (not enforced for flexibility)
    label           VARCHAR(500),                  -- Human-readable label for display
    properties      JSONB NOT NULL DEFAULT '{}',   -- Denormalised properties for graph queries
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, node_type, entity_id)
);

CREATE INDEX idx_graph_node_tenant ON graph_node (tenant_id);
CREATE INDEX idx_graph_node_type ON graph_node (tenant_id, node_type);
CREATE INDEX idx_graph_node_entity ON graph_node (entity_id);
CREATE INDEX idx_graph_node_props ON graph_node USING GIN (properties);

-- ============================================================
-- GRAPH EDGE: Typed, directed relationships between nodes
-- ============================================================
CREATE TABLE graph_edge (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    from_node_id    UUID NOT NULL REFERENCES graph_node(id),
    to_node_id      UUID NOT NULL REFERENCES graph_node(id),
    edge_type       VARCHAR(100) NOT NULL,         -- See edge type catalog below
    weight          NUMERIC(10,4) DEFAULT 1.0,     -- For weighted traversals (recovery rate, risk score)
    properties      JSONB NOT NULL DEFAULT '{}',   -- Edge-specific data
    valid_from      TIMESTAMPTZ NOT NULL DEFAULT now(),
    valid_to        TIMESTAMPTZ,                   -- NULL = currently active
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_graph_edge_from ON graph_edge (from_node_id, edge_type);
CREATE INDEX idx_graph_edge_to ON graph_edge (to_node_id, edge_type);
CREATE INDEX idx_graph_edge_tenant ON graph_edge (tenant_id, edge_type);
CREATE INDEX idx_graph_edge_type ON graph_edge (edge_type);
CREATE INDEX idx_graph_edge_props ON graph_edge USING GIN (properties);

/*
EDGE TYPE CATALOG:

-- Consumer relationships
PLACED_ORDER:       consumer -> order
INITIATED_RETURN:   consumer -> return
SHARES_ADDRESS:     consumer -> consumer       (fraud network)
SHARES_PAYMENT:     consumer -> consumer       (fraud network)
SHARES_DEVICE:      consumer -> consumer       (fraud network)

-- Order/Return lifecycle
RETURN_FOR_ORDER:   return -> order
CONTAINS_ITEM:      return -> item
ITEM_IS_PRODUCT:    item -> product

-- Inspection & Disposition flow
INSPECTED_AS:       item -> inspection_result   (properties: grade, photos)
DISPOSITIONED_TO:   item -> disposition_channel (properties: type, recovery)
PROCESSED_BY:       item -> partner             (properties: cost, certificate)
RECYCLED_BY:        item -> partner             (properties: weee_cat, cert_number)
REFURBISHED_BY:     item -> partner

-- Shipping
SHIPPED_VIA:        return -> shipment
SHIPPED_FROM:       shipment -> location
SHIPPED_TO:         shipment -> location

-- Product lifecycle (DPP)
MANUFACTURED_BY:    product -> manufacturer
SOLD_TO:            product -> consumer         (via order)
RETURNED_BY:        product -> consumer         (via return)
REPAIRED_BY:        product -> partner
RESOLD_TO:          product -> consumer         (recommerce)
RECYCLED_AT:        product -> partner          (end of life)

-- Fraud network
FLAGGED_FOR:        consumer -> fraud_signal    (properties: type, severity)
ASSOCIATED_WITH:    return -> fraud_signal
*/

-- ============================================================
-- JURISDICTION HIERARCHY via ltree
-- ============================================================
CREATE TABLE jurisdiction (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code            VARCHAR(10) NOT NULL UNIQUE,   -- ISO 3166 code
    name            VARCHAR(255) NOT NULL,
    path            LTREE NOT NULL,                -- e.g., 'earth.eu.de.by' (Earth > EU > Germany > Bavaria)
    jurisdiction_type VARCHAR(50) NOT NULL,         -- continent, region, country, state, city
    regulations     JSONB NOT NULL DEFAULT '{}',
    /*
    regulations example:
    {
        "weee": {"applicable": true, "directive": "2012/19/EU"},
        "right_to_repair": {"applicable": true, "effective_date": "2026-07-31"},
        "return_window_min_days": 14,
        "data_privacy": "gdpr"
    }
    */
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_jurisdiction_path ON jurisdiction USING GIST (path);
```

## Example Graph Queries

### Fraud network detection: find consumers sharing addresses

```sql
-- Find all consumers connected to a suspicious consumer via shared addresses
WITH RECURSIVE fraud_network AS (
    -- Start from the suspicious consumer
    SELECT
        gn.entity_id as consumer_id,
        gn.label as consumer_name,
        0 as depth,
        ARRAY[gn.entity_id] as path
    FROM graph_node gn
    WHERE gn.entity_id = '550e8400-...'  -- suspicious consumer
      AND gn.node_type = 'consumer'

    UNION ALL

    -- Traverse SHARES_ADDRESS and SHARES_PAYMENT edges
    SELECT
        target.entity_id,
        target.label,
        fn.depth + 1,
        fn.path || target.entity_id
    FROM fraud_network fn
    JOIN graph_node source ON source.entity_id = fn.consumer_id AND source.node_type = 'consumer'
    JOIN graph_edge e ON e.from_node_id = source.id
        AND e.edge_type IN ('SHARES_ADDRESS', 'SHARES_PAYMENT', 'SHARES_DEVICE')
        AND e.valid_to IS NULL
    JOIN graph_node target ON target.id = e.to_node_id AND target.node_type = 'consumer'
    WHERE fn.depth < 3  -- Max 3 hops
      AND NOT (target.entity_id = ANY(fn.path))  -- Prevent cycles
)
SELECT consumer_id, consumer_name, depth
FROM fraud_network
WHERE depth > 0
ORDER BY depth;
```

### Product lifecycle traceability (DPP)

```sql
-- Trace the full lifecycle of a product instance (by serial number)
SELECT
    e.edge_type as lifecycle_step,
    source.node_type as from_type,
    source.label as from_label,
    target.node_type as to_type,
    target.label as to_label,
    e.properties as step_details,
    e.valid_from as step_date
FROM graph_node product_node
JOIN graph_edge e ON (e.from_node_id = product_node.id OR e.to_node_id = product_node.id)
JOIN graph_node source ON source.id = e.from_node_id
JOIN graph_node target ON target.id = e.to_node_id
WHERE product_node.entity_id = '...'  -- product ID
  AND product_node.node_type = 'product'
ORDER BY e.valid_from ASC;
```

### Disposition path analysis: which routes yield best recovery?

```sql
-- Average recovery rate by disposition path (condition grade -> disposition type -> partner)
SELECT
    inspect_edge.properties->>'condition_grade' as grade,
    dispo_edge.properties->>'disposition_type' as disposition,
    partner_node.label as partner,
    COUNT(*) as item_count,
    AVG((dispo_edge.properties->>'estimated_recovery')::numeric) as avg_estimated,
    AVG((dispo_edge.properties->>'actual_recovery')::numeric) as avg_actual,
    AVG((dispo_edge.properties->>'actual_recovery')::numeric) /
        NULLIF(AVG((dispo_edge.properties->>'estimated_recovery')::numeric), 0) as accuracy
FROM graph_node item_node
JOIN graph_edge inspect_edge ON inspect_edge.from_node_id = item_node.id
    AND inspect_edge.edge_type = 'INSPECTED_AS'
JOIN graph_edge dispo_edge ON dispo_edge.from_node_id = item_node.id
    AND dispo_edge.edge_type = 'DISPOSITIONED_TO'
JOIN graph_edge partner_edge ON partner_edge.from_node_id = item_node.id
    AND partner_edge.edge_type IN ('PROCESSED_BY', 'REFURBISHED_BY', 'RECYCLED_BY')
JOIN graph_node partner_node ON partner_node.id = partner_edge.to_node_id
WHERE item_node.tenant_id = '...'
  AND item_node.node_type = 'item'
GROUP BY grade, disposition, partner_node.label
ORDER BY avg_actual DESC;
```

### Hierarchical product category query via ltree

```sql
-- Find all returns for products in the 'electronics' category tree
SELECT r.rma_number, p.name, p.category_path
FROM return_request r
JOIN return_item ri ON ri.return_request_id = r.id
JOIN product p ON p.id = ri.product_id
WHERE r.tenant_id = '...'
  AND p.category_path <@ 'electronics';  -- All descendants of 'electronics'
```

### Jurisdiction regulation lookup via ltree

```sql
-- Find all applicable regulations for a location (inherits from parent jurisdictions)
SELECT name, path, regulations
FROM jurisdiction
WHERE path @> 'earth.eu.de'  -- All ancestors of Germany (including EU-level regulations)
ORDER BY nlevel(path) ASC;   -- Most general first
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Identity & Multi-Tenancy | 2 | tenant, app_user |
| Product & Inventory | 1 | product (with ltree category) |
| Consumer & Order | 2 | consumer, original_order |
| Return Lifecycle | 3 | return_request, return_item, return_shipment |
| Policy & Config | 3 | return_policy, disposition_partner, integration |
| Graph Layer | 2 | graph_node, graph_edge |
| Reference Data | 1 | jurisdiction (with ltree hierarchy) |
| **Total** | **14** | Plus ltree extension; graph layer adds 2 tables to a hybrid relational base |

---

## Key Design Decisions

1. **PostgreSQL-native graph via graph_node/graph_edge tables** — avoids the operational overhead of a separate Neo4j deployment while supporting the relationship queries that matter most (fraud networks, lifecycle traceability, disposition paths). For platforms processing <10M returns/year, PostgreSQL graph queries perform well enough.

2. **Graph nodes reference relational entities** — `graph_node.entity_id` points to the UUID primary key in the corresponding relational table (consumer, product, return_request, etc.). The relational table is the system of record for CRUD; the graph layer is the system of record for relationships.

3. **Temporal edges with valid_from/valid_to** — graph edges have a validity window. When a consumer changes address, the old SHARES_ADDRESS edge gets a `valid_to` timestamp and a new edge is created. This enables point-in-time fraud analysis.

4. **ltree for product categories and jurisdictions** — PostgreSQL's `ltree` extension provides efficient hierarchical queries without recursive CTEs. `p.category_path <@ 'electronics'` finds all products in any electronics subcategory in a single index scan.

5. **Jurisdiction hierarchy with inherited regulations** — the `jurisdiction` table with ltree paths enables queries like "what regulations apply in Bavaria?" by traversing up the hierarchy (Bavaria -> Germany -> EU -> Earth). WEEE applies at EU level; Right to Repair at member state level.

6. **Edge type catalog is extensible** — adding new relationship types (e.g., RECOMMENDED_BY for AI disposition recommendations) requires only inserting new rows, not schema changes. The `properties` JSONB on each edge carries type-specific data.

7. **Weighted edges for analytics** — the `weight` column on graph_edge enables weighted path analysis. For disposition routing, weight can represent recovery rate; for fraud detection, weight can represent confidence.

8. **Dual-write pattern** — application code writes to both relational tables and graph tables in the same transaction. This adds some write overhead but ensures consistency. Alternatively, graph edges can be populated asynchronously from a change data capture stream.

9. **Fraud network detection as a first-class capability** — the SHARES_ADDRESS, SHARES_PAYMENT, and SHARES_DEVICE edge types enable multi-hop fraud network analysis that would require complex self-joins in a pure relational model. This is a strategic differentiator for the platform.

10. **DPP-ready lifecycle graph** — the edge types (MANUFACTURED_BY, SOLD_TO, RETURNED_BY, REPAIRED_BY, RESOLD_TO, RECYCLED_AT) map directly to EU Digital Product Passport lifecycle events. As ESPR delegated acts are published, the graph structure can absorb new lifecycle steps without schema changes.
