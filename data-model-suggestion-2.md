# Data Model Suggestion 2: Event-Sourced / Audit-First (CQRS)

> Project: Reverse Logistics Manager · Created: 2026-05-20

## Philosophy

This model uses an append-only event store as the single source of truth for the entire return lifecycle. Every state change — from RMA creation through inspection, disposition, repair, and resolution — is captured as an immutable event. Current state is derived by replaying events or reading from materialised projections (read models) that are rebuilt from the event stream. This is the CQRS (Command Query Responsibility Segregation) pattern: commands write events, queries read projections.

Event sourcing is the gold standard for domains where a complete audit trail is mandatory, temporal queries are valuable ("what was the condition grade of this item before it was re-inspected?"), and AI analytics benefit from rich historical data. Financial systems, healthcare records, and compliance-heavy logistics platforms use this pattern because you can never lose data — events are immutable and the full history is always available.

The trade-off is increased infrastructure complexity. You need an event store, a projection builder (event processor), and separate read-model tables. Simple queries that would be a single SELECT in a relational model require either pre-built projections or event replay. However, for a reverse logistics platform where regulatory audits, fraud investigation, and ML training data are core requirements, the benefits outweigh the cost.

**Best for:** Platforms where regulatory audit trails, temporal queries, fraud forensics, and ML training on historical state transitions are primary requirements.

**Trade-offs:**
- (+) Complete, immutable audit trail by design — not bolted on
- (+) Temporal queries are natural: "what was true at time T?"
- (+) Rich ML training data from event sequences (return patterns, fraud trajectories)
- (+) Easy to add new read models without changing the write path
- (+) Event replay enables bug fixes on historical data and schema evolution
- (-) Higher infrastructure complexity (event store + projection builder + read models)
- (-) Eventual consistency between event store and projections
- (-) Simple CRUD queries require pre-built projections
- (-) Event schema evolution requires careful versioning
- (-) Steeper learning curve for developers unfamiliar with CQRS

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| GS1 GTIN | Product identification in `ReturnItemAdded` events and product projection |
| GS1 GRAI | Returnable asset identification in logistics events |
| ISO 3166-1/2 | Jurisdiction codes in address-related events and shipment projections |
| ISO 4217 | Currency codes in all monetary event fields |
| WEEE Directive | `DispositionDecided` events carry WEEE category; recycling events generate compliance certificates |
| R2v3 / e-Stewards | Certification references in `DispositionPartnerAssigned` events |
| OCSF (Open Cybersecurity Schema Framework) | Event structure influenced by OCSF's approach to structured, typed, categorised events |
| CloudEvents 1.0 | Event envelope follows CloudEvents specification for interoperability |
| GDPR / CCPA | `ConsumerDataDeletionRequested` event triggers PII scrubbing from projections while preserving anonymised events |

---

## Event Store

```sql
-- The single source of truth: an append-only event store
CREATE TABLE event_store (
    event_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_id       UUID NOT NULL,                -- Aggregate root ID (e.g., return_request ID)
    stream_type     VARCHAR(100) NOT NULL,         -- return, inspection, repair, shipment, consumer
    tenant_id       UUID NOT NULL,
    sequence_number BIGINT NOT NULL,               -- Monotonically increasing within a stream
    event_type      VARCHAR(200) NOT NULL,         -- e.g., ReturnRequested, ItemInspected, DispositionDecided
    event_version   INTEGER NOT NULL DEFAULT 1,    -- Schema version for this event type
    payload         JSONB NOT NULL,                -- Event-specific data
    metadata        JSONB NOT NULL DEFAULT '{}',   -- Correlation ID, causation ID, user agent, IP
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (stream_id, sequence_number)
);

-- Primary query pattern: replay a stream
CREATE INDEX idx_event_store_stream ON event_store (stream_id, sequence_number);

-- Query all events for a tenant (e.g., for tenant-wide projections)
CREATE INDEX idx_event_store_tenant ON event_store (tenant_id, created_at);

-- Query by event type (e.g., rebuild a specific projection)
CREATE INDEX idx_event_store_type ON event_store (event_type, created_at);

-- Global sequence for ordered processing
CREATE SEQUENCE event_global_sequence;

-- Partition by month for performance at scale
-- CREATE TABLE event_store_2026_05 PARTITION OF event_store
--     FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');
```

### Event Type Catalog

```sql
/*
Event types and their payload structures:

-- RETURN LIFECYCLE EVENTS --

ReturnRequested:
{
    "rma_number": "RMA-2026-00142",
    "consumer_id": "uuid",
    "original_order_id": "uuid",
    "external_order_id": "ORD-12345",
    "return_type": "return",
    "initiated_via": "portal",
    "items": [
        {
            "item_id": "uuid",
            "product_id": "uuid",
            "sku": "WIDGET-BLU-M",
            "gtin": "00012345678905",
            "quantity": 1,
            "reason_code": "damaged",
            "reason_detail": "Screen cracked on arrival",
            "unit_price": 299.99,
            "serial_number": "SN123456"
        }
    ],
    "currency": "USD"
}

ReturnApproved:
{
    "rma_number": "RMA-2026-00142",
    "approved_by": "uuid",
    "fraud_risk_score": 12.5,
    "policy_id": "uuid",
    "resolution_type": "refund"
}

ReturnRejected:
{
    "rma_number": "RMA-2026-00142",
    "rejected_by": "uuid",
    "rejection_reason": "outside_return_window",
    "fraud_risk_score": 85.0
}

FraudSignalDetected:
{
    "rma_number": "RMA-2026-00142",
    "signal_type": "wardrobing",
    "severity": "high",
    "confidence": 0.87,
    "model_version": "fraud-v2.3",
    "details": "Consumer returned 12 items in 30 days across 4 orders"
}

-- SHIPPING EVENTS --

ReturnLabelGenerated:
{
    "rma_number": "RMA-2026-00142",
    "carrier": "ups",
    "tracking_number": "1Z999AA10123456784",
    "label_url": "https://...",
    "qr_code_url": "https://...",
    "shipping_cost": 8.50,
    "currency": "USD"
}

ReturnShipmentInTransit:
{
    "tracking_number": "1Z999AA10123456784",
    "carrier": "ups",
    "estimated_delivery": "2026-05-25T14:00:00Z"
}

ReturnShipmentDelivered:
{
    "tracking_number": "1Z999AA10123456784",
    "delivered_at": "2026-05-24T10:30:00Z",
    "received_by": "uuid"
}

-- INSPECTION EVENTS --

ItemInspected:
{
    "item_id": "uuid",
    "inspector_id": "uuid",
    "condition_grade": "fair",
    "is_authentic": true,
    "notes": "Minor cosmetic damage, fully functional",
    "photos": [
        {"url": "https://...", "type": "overview"},
        {"url": "https://...", "type": "damage_detail"}
    ]
}

AIConditionGraded:
{
    "item_id": "uuid",
    "ai_condition_grade": "fair",
    "ai_confidence": 0.92,
    "model_version": "cv-grading-v1.4",
    "damage_regions": [{"x": 120, "y": 45, "w": 80, "h": 60, "label": "scratch"}]
}

-- DISPOSITION EVENTS --

DispositionDecided:
{
    "item_id": "uuid",
    "disposition_type": "refurbish",
    "decided_by": "ml_model",
    "ml_confidence": 0.88,
    "estimated_recovery": 180.00,
    "currency": "USD",
    "partner_id": "uuid",
    "weee_category": "consumer_electronics"
}

DispositionCompleted:
{
    "item_id": "uuid",
    "disposition_type": "refurbish",
    "actual_recovery": 195.00,
    "currency": "USD",
    "recycling_certificate_number": "CERT-2026-0042"
}

-- REPAIR EVENTS --

RepairJobCreated:
{
    "repair_id": "uuid",
    "item_id": "uuid",
    "warranty_id": "uuid",
    "right_to_repair_eligible": true
}

RepairDiagnosed:
{
    "repair_id": "uuid",
    "diagnosis": "Display assembly replacement required",
    "estimated_cost": 45.00,
    "parts_needed": [
        {"part_sku": "DISP-ASMB-42", "quantity": 1, "unit_cost": 28.00, "source": "new"}
    ]
}

RepairCompleted:
{
    "repair_id": "uuid",
    "actual_cost": 42.50,
    "parts_used": [
        {"part_sku": "DISP-ASMB-42", "quantity": 1, "source": "new"}
    ]
}

-- FINANCIAL EVENTS --

RefundIssued:
{
    "rma_number": "RMA-2026-00142",
    "amount": 299.99,
    "currency": "USD",
    "method": "original_payment",
    "gateway_ref": "ref_abc123"
}

StoreCreditIssued:
{
    "rma_number": "RMA-2026-00142",
    "amount": 299.99,
    "currency": "USD",
    "credit_code": "SC-2026-00042"
}

ExchangeShipmentCreated:
{
    "rma_number": "RMA-2026-00142",
    "exchange_product_id": "uuid",
    "exchange_sku": "WIDGET-RED-M",
    "tracking_number": "1Z999AA10123456799"
}

-- SUSTAINABILITY EVENTS --

CarbonFootprintCalculated:
{
    "rma_number": "RMA-2026-00142",
    "transport_kg_co2": 2.34,
    "processing_kg_co2": 0.15,
    "packaging_kg_co2": 0.08,
    "total_kg_co2": 2.57,
    "method": "carrier_reported"
}

-- RETURN RESOLUTION --

ReturnResolved:
{
    "rma_number": "RMA-2026-00142",
    "resolution_type": "refund",
    "total_refund": 299.99,
    "total_store_credit": 0,
    "currency": "USD"
}
*/
```

## Read Model Projections

These tables are rebuilt from the event store and serve as optimised query targets. They can be dropped and rebuilt at any time.

```sql
-- ============================================================
-- PROJECTION: Current return state (denormalised for fast reads)
-- ============================================================
CREATE TABLE proj_return (
    id              UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    rma_number      VARCHAR(50) NOT NULL,
    consumer_id     UUID NOT NULL,
    consumer_email  VARCHAR(255),
    consumer_name   VARCHAR(255),
    original_order_id UUID NOT NULL,
    external_order_id VARCHAR(255),
    status          VARCHAR(50) NOT NULL,
    return_type     VARCHAR(50) NOT NULL,
    resolution_type VARCHAR(50),
    initiated_via   VARCHAR(50),
    fraud_risk_score NUMERIC(5,2),
    item_count      INTEGER NOT NULL DEFAULT 0,
    total_value     NUMERIC(12,2) NOT NULL DEFAULT 0,
    currency        VARCHAR(3) NOT NULL DEFAULT 'USD',
    refund_amount   NUMERIC(12,2),
    store_credit_amount NUMERIC(12,2),
    carrier         VARCHAR(100),
    tracking_number VARCHAR(255),
    shipping_status VARCHAR(50),
    approved_at     TIMESTAMPTZ,
    received_at     TIMESTAMPTZ,
    resolved_at     TIMESTAMPTZ,
    total_kg_co2    NUMERIC(10,4),
    last_event_id   UUID NOT NULL,                -- Last processed event
    last_event_at   TIMESTAMPTZ NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL,
    UNIQUE (tenant_id, rma_number)
);

CREATE INDEX idx_proj_return_tenant_status ON proj_return (tenant_id, status);
CREATE INDEX idx_proj_return_consumer ON proj_return (consumer_id);
CREATE INDEX idx_proj_return_created ON proj_return (tenant_id, created_at DESC);

-- ============================================================
-- PROJECTION: Return items with inspection and disposition
-- ============================================================
CREATE TABLE proj_return_item (
    id              UUID PRIMARY KEY,
    return_id       UUID NOT NULL REFERENCES proj_return(id),
    tenant_id       UUID NOT NULL,
    product_id      UUID,
    sku             VARCHAR(100),
    gtin            VARCHAR(14),
    product_name    VARCHAR(500),
    quantity        INTEGER NOT NULL DEFAULT 1,
    unit_price      NUMERIC(12,2),
    reason_code     VARCHAR(100),
    reason_detail   TEXT,
    serial_number   VARCHAR(255),
    -- Inspection fields (denormalised)
    condition_grade VARCHAR(20),
    ai_condition_grade VARCHAR(20),
    ai_confidence   NUMERIC(5,4),
    is_authentic    BOOLEAN,
    inspected_at    TIMESTAMPTZ,
    inspector_id    UUID,
    -- Disposition fields (denormalised)
    disposition_type VARCHAR(50),
    disposition_decided_by VARCHAR(50),
    disposition_partner_id UUID,
    estimated_recovery NUMERIC(12,2),
    actual_recovery NUMERIC(12,2),
    disposition_completed_at TIMESTAMPTZ,
    -- Repair fields (denormalised)
    repair_status   VARCHAR(50),
    repair_cost     NUMERIC(12,2),
    last_event_id   UUID NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_proj_return_item_return ON proj_return_item (return_id);
CREATE INDEX idx_proj_return_item_tenant ON proj_return_item (tenant_id);

-- ============================================================
-- PROJECTION: Consumer risk profile (aggregated from events)
-- ============================================================
CREATE TABLE proj_consumer_profile (
    consumer_id     UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    email           VARCHAR(255),
    full_name       VARCHAR(255),
    customer_tier   VARCHAR(50),
    total_returns   INTEGER NOT NULL DEFAULT 0,
    total_returns_value NUMERIC(12,2) NOT NULL DEFAULT 0,
    returns_last_30_days INTEGER NOT NULL DEFAULT 0,
    returns_last_90_days INTEGER NOT NULL DEFAULT 0,
    fraud_signals_count INTEGER NOT NULL DEFAULT 0,
    highest_fraud_severity VARCHAR(20),
    computed_risk_score NUMERIC(5,2),
    last_return_at  TIMESTAMPTZ,
    last_event_id   UUID NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_proj_consumer_tenant ON proj_consumer_profile (tenant_id);
CREATE INDEX idx_proj_consumer_risk ON proj_consumer_profile (tenant_id, computed_risk_score DESC);

-- ============================================================
-- PROJECTION: Disposition analytics (for dashboard and ML)
-- ============================================================
CREATE TABLE proj_disposition_analytics (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    product_id      UUID,
    sku             VARCHAR(100),
    category        VARCHAR(255),
    disposition_type VARCHAR(50) NOT NULL,
    condition_grade VARCHAR(20),
    decided_by      VARCHAR(50),
    estimated_recovery NUMERIC(12,2),
    actual_recovery NUMERIC(12,2),
    currency        VARCHAR(3),
    decided_at      TIMESTAMPTZ NOT NULL,
    completed_at    TIMESTAMPTZ,
    partner_id      UUID,
    event_id        UUID NOT NULL
);

CREATE INDEX idx_proj_dispo_tenant ON proj_disposition_analytics (tenant_id, decided_at DESC);
CREATE INDEX idx_proj_dispo_product ON proj_disposition_analytics (product_id);
CREATE INDEX idx_proj_dispo_type ON proj_disposition_analytics (tenant_id, disposition_type);

-- ============================================================
-- PROJECTION: Operational metrics (time-series for dashboards)
-- ============================================================
CREATE TABLE proj_daily_metrics (
    tenant_id       UUID NOT NULL,
    metric_date     DATE NOT NULL,
    returns_created INTEGER NOT NULL DEFAULT 0,
    returns_resolved INTEGER NOT NULL DEFAULT 0,
    returns_approved INTEGER NOT NULL DEFAULT 0,
    returns_rejected INTEGER NOT NULL DEFAULT 0,
    total_refund_amount NUMERIC(12,2) NOT NULL DEFAULT 0,
    total_store_credit NUMERIC(12,2) NOT NULL DEFAULT 0,
    total_recovery_amount NUMERIC(12,2) NOT NULL DEFAULT 0,
    avg_resolution_hours NUMERIC(10,2),
    fraud_signals_raised INTEGER NOT NULL DEFAULT 0,
    total_kg_co2    NUMERIC(10,4),
    items_restocked INTEGER NOT NULL DEFAULT 0,
    items_refurbished INTEGER NOT NULL DEFAULT 0,
    items_recycled  INTEGER NOT NULL DEFAULT 0,
    items_liquidated INTEGER NOT NULL DEFAULT 0,
    last_event_id   UUID NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL,
    PRIMARY KEY (tenant_id, metric_date)
);
```

## Reference Data (Non-Event-Sourced)

Some data is configuration, not event-driven. These tables are managed via standard CRUD.

```sql
-- Tenant configuration
CREATE TABLE tenant (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    type            VARCHAR(50) NOT NULL,
    default_currency VARCHAR(3) NOT NULL DEFAULT 'USD',
    timezone        VARCHAR(50) NOT NULL DEFAULT 'UTC',
    settings        JSONB NOT NULL DEFAULT '{}',
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Product catalog (synced from external systems)
CREATE TABLE product (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    sku             VARCHAR(100) NOT NULL,
    gtin            VARCHAR(14),
    name            VARCHAR(500) NOT NULL,
    category        VARCHAR(255),
    brand           VARCHAR(255),
    unit_cost       NUMERIC(12,2),
    unit_price      NUMERIC(12,2),
    currency        VARCHAR(3) DEFAULT 'USD',
    weee_category   VARCHAR(50),
    right_to_repair_eligible BOOLEAN NOT NULL DEFAULT FALSE,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, sku)
);

-- Return policies and rules
CREATE TABLE return_policy (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(255) NOT NULL,
    return_window_days INTEGER NOT NULL DEFAULT 30,
    rules           JSONB NOT NULL DEFAULT '[]',  -- Array of rule objects
    is_default      BOOLEAN NOT NULL DEFAULT FALSE,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Disposition partners
CREATE TABLE disposition_partner (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(255) NOT NULL,
    partner_type    VARCHAR(50) NOT NULL,
    certification_type VARCHAR(50),
    certification_expiry DATE,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Integration configurations
CREATE TABLE integration (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    platform        VARCHAR(50) NOT NULL,
    integration_type VARCHAR(50) NOT NULL,
    credentials     JSONB NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Event store checkpoint for projection processors
CREATE TABLE projection_checkpoint (
    projection_name VARCHAR(100) PRIMARY KEY,
    last_event_id   UUID NOT NULL,
    last_event_at   TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Example Queries

### Replay a return's full history

```sql
-- Get the complete timeline of a return
SELECT event_type, payload, created_at
FROM event_store
WHERE stream_id = '550e8400-e29b-41d4-a716-446655440000'
  AND stream_type = 'return'
ORDER BY sequence_number ASC;
```

### Point-in-time query: "What was this return's state at a specific date?"

```sql
-- Replay events up to a specific point in time
SELECT event_type, payload, created_at
FROM event_store
WHERE stream_id = '550e8400-e29b-41d4-a716-446655440000'
  AND created_at <= '2026-05-15T12:00:00Z'
ORDER BY sequence_number ASC;
-- Application code replays these events to reconstruct state at that moment
```

### Find all returns where disposition changed after initial decision

```sql
-- Identify returns with multiple disposition events (re-routes)
SELECT stream_id, COUNT(*) as disposition_changes
FROM event_store
WHERE event_type = 'DispositionDecided'
  AND tenant_id = '...'
GROUP BY stream_id
HAVING COUNT(*) > 1
ORDER BY disposition_changes DESC;
```

### Dashboard query using materialised projection

```sql
-- Fast dashboard query — no event replay needed
SELECT status, COUNT(*) as count, SUM(total_value) as value
FROM proj_return
WHERE tenant_id = '...'
  AND created_at >= now() - INTERVAL '30 days'
GROUP BY status;
```

### ML training data: extract return-to-disposition sequences

```sql
-- Extract event sequences for ML training
SELECT
    e.stream_id,
    array_agg(e.event_type ORDER BY e.sequence_number) as event_sequence,
    array_agg(e.payload ORDER BY e.sequence_number) as payloads
FROM event_store e
WHERE e.tenant_id = '...'
  AND e.stream_type = 'return'
  AND e.created_at >= '2026-01-01'
GROUP BY e.stream_id;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store | 1 | Single append-only event_store table (may be partitioned by month) |
| Projections — Returns | 2 | proj_return, proj_return_item |
| Projections — Analytics | 3 | proj_consumer_profile, proj_disposition_analytics, proj_daily_metrics |
| Reference Data | 5 | tenant, product, return_policy, disposition_partner, integration |
| Infrastructure | 1 | projection_checkpoint |
| **Total** | **12** | Plus 1 sequence; projections are rebuildable from events |

---

## Key Design Decisions

1. **Single event_store table as source of truth** — all business events flow into one append-only table. This simplifies the write path to a single INSERT and guarantees that no state change is ever lost.

2. **Stream-based event grouping** — events are grouped by `stream_id` (the aggregate root, typically a return_request) with a `sequence_number` ensuring ordering within a stream. This maps directly to DDD aggregate patterns.

3. **Event versioning** — the `event_version` field on each event allows schema evolution. When the payload structure for `ItemInspected` changes, the version increments and the projection builder handles both old and new versions.

4. **Projections are disposable** — every proj_* table can be dropped and rebuilt by replaying events from the event_store. The `projection_checkpoint` table tracks how far each projection has processed, enabling incremental updates.

5. **Reference data stays relational** — tenant configuration, product catalog, policies, and partner data are not event-sourced because they are externally managed configuration, not domain events. This avoids over-engineering.

6. **CloudEvents-influenced envelope** — the event structure (event_id, stream_type, event_type, metadata with correlation/causation IDs) follows the CloudEvents 1.0 specification pattern for interoperability with external event systems.

7. **Denormalised projections for read performance** — proj_return includes consumer name, tracking number, and carbon footprint directly, avoiding JOINs for the most common dashboard queries.

8. **GDPR compliance via projection scrubbing** — when a `ConsumerDataDeletionRequested` event is processed, PII is removed from projections but the anonymised event record remains in the event store for audit integrity. The event itself can store a hash instead of PII.

9. **Built-in ML training pipeline** — the event store is inherently a sequence database. Extracting return-to-disposition event sequences for ML training requires no additional ETL — just a GROUP BY on stream_id.

10. **Monthly partitioning readiness** — the event_store table can be partitioned by `created_at` month for performance at scale. Commented-out partition DDL is included as a starting point.
