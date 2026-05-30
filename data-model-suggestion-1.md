# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Reverse Logistics Manager · Created: 2026-05-20

## Philosophy

This model follows a traditional third-normal-form relational design where every concept in the reverse logistics domain has its own dedicated table with explicit foreign key relationships. The schema is optimised for data integrity, complex cross-entity queries, and regulatory compliance reporting. Every relationship is explicit, every constraint is enforced at the database level, and every entity has a clear, single source of truth.

This approach mirrors how mature enterprise ERP systems (SAP, Oracle, IFS Cloud) structure their returns modules. It is the most straightforward to reason about, the easiest to onboard new developers onto, and the most compatible with standard SQL reporting tools. The trade-off is a larger number of tables and more complex JOIN queries for operations that span the full return lifecycle.

**Best for:** Teams building for regulatory-heavy environments (EU WEEE, Right to Repair) where data integrity, auditability, and complex cross-entity reporting are non-negotiable.

**Trade-offs:**
- (+) Maximum data integrity via foreign keys and constraints
- (+) Standard SQL — no proprietary query patterns needed
- (+) Easy to build reports across any dimension (by product, by merchant, by disposition channel, by carrier)
- (+) Clear schema documentation serves as domain documentation
- (-) High table count (~45 tables) increases JOIN complexity
- (-) Adding jurisdiction-specific fields requires schema migrations
- (-) Multi-tenant row-level security must be enforced consistently across all tables
- (-) Schema evolution is slower — every new concept requires a new migration

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| GS1 GTIN | `product.gtin` field stores the 14-digit Global Trade Item Number for item-level identification |
| GS1 GRAI | `returnable_asset.grai` field stores the Global Returnable Asset Identifier for reusable containers/pallets |
| GS1 Digital Link | `product.digital_link_url` stores the GS1 Digital Link URI for QR-code-based returns |
| ISO 3166-1/2 | `address.country_code` and `address.subdivision_code` use ISO 3166 codes for jurisdiction modeling |
| ISO 4217 | `currency` fields use ISO 4217 three-letter currency codes |
| WEEE Directive | `disposition_record.weee_category` and `recycling_certificate` table track WEEE compliance |
| R2v3 / e-Stewards | `disposition_partner.certification_type` references R2 or e-Stewards certification |
| EU Right to Repair | `repair_job.right_to_repair_eligible` flag and `repair_compliance_doc` table |
| OAuth 2.0 / OIDC | `api_client` and `user_session` tables support OAuth 2.0 client credentials and OIDC SSO |
| GDPR / CCPA | `consumer.data_retention_expiry` and `data_deletion_request` table support privacy compliance |

---

## Core Identity & Multi-Tenancy

```sql
-- Every tenant (merchant, retailer, manufacturer, 3PL) in the system
CREATE TABLE tenant (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    type            VARCHAR(50) NOT NULL CHECK (type IN ('retailer', 'manufacturer', '3pl', 'brand')),
    subscription_tier VARCHAR(50) NOT NULL DEFAULT 'standard',
    default_currency VARCHAR(3) NOT NULL DEFAULT 'USD', -- ISO 4217
    timezone        VARCHAR(50) NOT NULL DEFAULT 'UTC',
    settings        JSONB NOT NULL DEFAULT '{}',
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_tenant_slug ON tenant (slug);

-- Users within a tenant
CREATE TABLE app_user (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    email           VARCHAR(255) NOT NULL,
    full_name       VARCHAR(255) NOT NULL,
    role            VARCHAR(50) NOT NULL CHECK (role IN ('admin', 'manager', 'operator', 'viewer', 'api_client')),
    password_hash   VARCHAR(255),
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, email)
);

CREATE INDEX idx_app_user_tenant ON app_user (tenant_id);
```

## Product & Inventory

```sql
-- Product catalog (synced from merchant's e-commerce/ERP)
CREATE TABLE product (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    sku             VARCHAR(100) NOT NULL,
    gtin            VARCHAR(14),                  -- GS1 GTIN-14
    name            VARCHAR(500) NOT NULL,
    description     TEXT,
    category        VARCHAR(255),
    brand           VARCHAR(255),
    unit_cost       NUMERIC(12,2),
    unit_price      NUMERIC(12,2),
    currency        VARCHAR(3) NOT NULL DEFAULT 'USD', -- ISO 4217
    weight_grams    INTEGER,
    digital_link_url VARCHAR(500),                -- GS1 Digital Link URI
    weee_category   VARCHAR(50),                  -- WEEE product category if applicable
    right_to_repair_eligible BOOLEAN NOT NULL DEFAULT FALSE,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, sku)
);

CREATE INDEX idx_product_tenant ON product (tenant_id);
CREATE INDEX idx_product_gtin ON product (gtin) WHERE gtin IS NOT NULL;
CREATE INDEX idx_product_sku ON product (tenant_id, sku);

-- Returnable assets (pallets, crates, reusable packaging)
CREATE TABLE returnable_asset (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    grai            VARCHAR(30) NOT NULL,         -- GS1 GRAI
    asset_type      VARCHAR(100) NOT NULL,
    description     VARCHAR(500),
    current_location VARCHAR(255),
    status          VARCHAR(50) NOT NULL DEFAULT 'available',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, grai)
);
```

## Consumer & Order Context

```sql
-- Consumer who initiated the return (PII subject to GDPR/CCPA)
CREATE TABLE consumer (
    id                      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id               UUID NOT NULL REFERENCES tenant(id),
    external_customer_id    VARCHAR(255),         -- ID in merchant's e-commerce system
    email                   VARCHAR(255) NOT NULL,
    full_name               VARCHAR(255),
    phone                   VARCHAR(50),
    customer_tier           VARCHAR(50) DEFAULT 'standard', -- standard, vip, high_risk
    total_returns_count     INTEGER NOT NULL DEFAULT 0,
    total_returns_value     NUMERIC(12,2) NOT NULL DEFAULT 0,
    fraud_risk_score        NUMERIC(5,2),         -- 0.00 to 100.00
    data_retention_expiry   TIMESTAMPTZ,          -- GDPR: when to purge PII
    created_at              TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at              TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, email)
);

CREATE INDEX idx_consumer_tenant ON consumer (tenant_id);
CREATE INDEX idx_consumer_external ON consumer (tenant_id, external_customer_id);

-- Address (shared by consumer, warehouse, drop-off location)
CREATE TABLE address (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    address_line_1  VARCHAR(255) NOT NULL,
    address_line_2  VARCHAR(255),
    city            VARCHAR(100) NOT NULL,
    state_province  VARCHAR(100),
    postal_code     VARCHAR(20),
    country_code    CHAR(2) NOT NULL,             -- ISO 3166-1 alpha-2
    subdivision_code VARCHAR(6),                  -- ISO 3166-2
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Original order from which the return originates
CREATE TABLE original_order (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    consumer_id     UUID NOT NULL REFERENCES consumer(id),
    external_order_id VARCHAR(255) NOT NULL,      -- Order ID in merchant's OMS
    order_date      TIMESTAMPTZ NOT NULL,
    order_total     NUMERIC(12,2) NOT NULL,
    currency        VARCHAR(3) NOT NULL DEFAULT 'USD',
    channel         VARCHAR(50),                  -- online, in_store, marketplace
    platform        VARCHAR(50),                  -- shopify, magento, woocommerce, sap
    shipping_address_id UUID REFERENCES address(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, external_order_id)
);

CREATE INDEX idx_original_order_consumer ON original_order (consumer_id);
```

## Return (RMA) Lifecycle

```sql
-- Return Merchandise Authorization — the core entity
CREATE TABLE return_request (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    rma_number      VARCHAR(50) NOT NULL,         -- Human-readable RMA number
    consumer_id     UUID NOT NULL REFERENCES consumer(id),
    original_order_id UUID NOT NULL REFERENCES original_order(id),
    status          VARCHAR(50) NOT NULL DEFAULT 'requested'
                    CHECK (status IN ('requested', 'approved', 'rejected',
                           'label_generated', 'in_transit', 'received',
                           'inspecting', 'disposition_decided', 'resolved', 'cancelled')),
    return_type     VARCHAR(50) NOT NULL DEFAULT 'return'
                    CHECK (return_type IN ('return', 'exchange', 'warranty_claim', 'repair')),
    resolution_type VARCHAR(50)
                    CHECK (resolution_type IN ('refund', 'exchange', 'store_credit',
                           'repair', 'keep_item', NULL)),
    initiated_via   VARCHAR(50) DEFAULT 'portal'
                    CHECK (initiated_via IN ('portal', 'api', 'chatbot', 'phone', 'in_store')),
    fraud_risk_score NUMERIC(5,2),                -- 0.00 to 100.00 at time of request
    fraud_flags     TEXT[],                       -- Array of flag codes
    approved_by     UUID REFERENCES app_user(id),
    approved_at     TIMESTAMPTZ,
    received_at     TIMESTAMPTZ,
    resolved_at     TIMESTAMPTZ,
    refund_amount   NUMERIC(12,2),
    refund_currency VARCHAR(3),
    store_credit_amount NUMERIC(12,2),
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, rma_number)
);

CREATE INDEX idx_return_request_tenant ON return_request (tenant_id);
CREATE INDEX idx_return_request_status ON return_request (tenant_id, status);
CREATE INDEX idx_return_request_consumer ON return_request (consumer_id);
CREATE INDEX idx_return_request_order ON return_request (original_order_id);
CREATE INDEX idx_return_request_created ON return_request (tenant_id, created_at DESC);

-- Individual items within a return request
CREATE TABLE return_item (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    return_request_id UUID NOT NULL REFERENCES return_request(id),
    product_id      UUID NOT NULL REFERENCES product(id),
    quantity        INTEGER NOT NULL DEFAULT 1,
    reason_code     VARCHAR(100) NOT NULL,        -- damaged, wrong_item, not_as_described, changed_mind, defective, etc.
    reason_detail   TEXT,
    condition_at_request VARCHAR(50),             -- Consumer-reported condition
    unit_price      NUMERIC(12,2) NOT NULL,
    serial_number   VARCHAR(255),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_return_item_request ON return_item (return_request_id);
CREATE INDEX idx_return_item_product ON return_item (product_id);

-- Reason codes reference table
CREATE TABLE reason_code (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    code            VARCHAR(100) NOT NULL,
    label           VARCHAR(255) NOT NULL,
    category        VARCHAR(100),                 -- quality, preference, logistics, fraud
    requires_photo  BOOLEAN NOT NULL DEFAULT FALSE,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    display_order   INTEGER NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, code)
);
```

## Return Policy Engine

```sql
-- Configurable return policies per tenant
CREATE TABLE return_policy (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(255) NOT NULL,
    return_window_days INTEGER NOT NULL DEFAULT 30,
    exchange_window_days INTEGER,
    requires_receipt BOOLEAN NOT NULL DEFAULT TRUE,
    restocking_fee_pct NUMERIC(5,2) DEFAULT 0,
    free_return_shipping BOOLEAN NOT NULL DEFAULT FALSE,
    allow_keep_item BOOLEAN NOT NULL DEFAULT FALSE,
    keep_item_threshold NUMERIC(12,2),            -- Max item value for keep-the-item
    is_default      BOOLEAN NOT NULL DEFAULT FALSE,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Policy rules: conditions that override default policy
CREATE TABLE policy_rule (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    policy_id       UUID NOT NULL REFERENCES return_policy(id),
    rule_type       VARCHAR(50) NOT NULL,         -- product_category, order_value, customer_tier, reason_code, jurisdiction
    operator        VARCHAR(20) NOT NULL,         -- equals, not_equals, greater_than, less_than, in, not_in
    field_value     TEXT NOT NULL,
    action          VARCHAR(50) NOT NULL,         -- approve, reject, require_review, extend_window, apply_fee
    action_params   JSONB,                        -- e.g. {"fee_pct": 15, "extended_days": 60}
    priority        INTEGER NOT NULL DEFAULT 0,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_policy_rule_policy ON policy_rule (policy_id);
```

## Shipping & Logistics

```sql
-- Return shipment (label, tracking)
CREATE TABLE return_shipment (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    return_request_id UUID NOT NULL REFERENCES return_request(id),
    carrier         VARCHAR(100) NOT NULL,        -- ups, fedex, usps, dhl, evri, dpd
    service_level   VARCHAR(100),
    tracking_number VARCHAR(255),
    label_url       VARCHAR(1000),
    label_format    VARCHAR(10) DEFAULT 'PDF',
    qr_code_url     VARCHAR(1000),                -- For box-free drop-off
    drop_off_location_id UUID REFERENCES drop_off_location(id),
    ship_from_address_id UUID REFERENCES address(id),
    ship_to_address_id UUID REFERENCES address(id),
    estimated_delivery TIMESTAMPTZ,
    actual_delivery TIMESTAMPTZ,
    shipping_cost   NUMERIC(12,2),
    currency        VARCHAR(3) DEFAULT 'USD',
    status          VARCHAR(50) NOT NULL DEFAULT 'pending'
                    CHECK (status IN ('pending', 'label_created', 'picked_up',
                           'in_transit', 'delivered', 'exception', 'cancelled')),
    customs_declaration JSONB,                    -- Cross-border customs data
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_return_shipment_request ON return_shipment (return_request_id);
CREATE INDEX idx_return_shipment_tracking ON return_shipment (tracking_number);

-- Drop-off locations (Return Bars, lockers, partner stores)
CREATE TABLE drop_off_location (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(255) NOT NULL,
    location_type   VARCHAR(50) NOT NULL,         -- return_bar, locker, partner_store, warehouse
    address_id      UUID NOT NULL REFERENCES address(id),
    latitude        NUMERIC(10,7),
    longitude       NUMERIC(10,7),
    operating_hours JSONB,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_drop_off_location_tenant ON drop_off_location (tenant_id);
```

## Inspection, Grading & Disposition

```sql
-- Condition inspection performed on received items
CREATE TABLE inspection (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    return_item_id  UUID NOT NULL REFERENCES return_item(id),
    inspector_id    UUID REFERENCES app_user(id),
    condition_grade VARCHAR(20) NOT NULL
                    CHECK (condition_grade IN ('new', 'like_new', 'good', 'fair', 'poor', 'damaged', 'counterfeit')),
    is_authentic    BOOLEAN NOT NULL DEFAULT TRUE,
    inspection_notes TEXT,
    ai_condition_grade VARCHAR(20),               -- AI-predicted grade from photos
    ai_confidence   NUMERIC(5,4),                 -- 0.0000 to 1.0000
    inspected_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_inspection_return_item ON inspection (return_item_id);

-- Photos captured during inspection
CREATE TABLE inspection_photo (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    inspection_id   UUID NOT NULL REFERENCES inspection(id),
    photo_url       VARCHAR(1000) NOT NULL,
    photo_type      VARCHAR(50) NOT NULL,         -- overview, damage_detail, serial_number, packaging
    ai_analysis     JSONB,                        -- CV model output: damage detection, counterfeit score
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_inspection_photo_inspection ON inspection_photo (inspection_id);

-- Disposition decision for each returned item
CREATE TABLE disposition (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    return_item_id  UUID NOT NULL REFERENCES return_item(id),
    inspection_id   UUID REFERENCES inspection(id),
    disposition_type VARCHAR(50) NOT NULL
                    CHECK (disposition_type IN ('restock', 'refurbish', 'liquidate',
                           'recycle', 'donate', 'destroy', 'return_to_vendor')),
    decided_by      VARCHAR(50) NOT NULL DEFAULT 'rule_engine'
                    CHECK (decided_by IN ('rule_engine', 'ml_model', 'manual')),
    ml_confidence   NUMERIC(5,4),                 -- ML model confidence if AI-decided
    destination_partner_id UUID REFERENCES disposition_partner(id),
    estimated_recovery NUMERIC(12,2),
    actual_recovery NUMERIC(12,2),
    recovery_currency VARCHAR(3) DEFAULT 'USD',
    decided_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_disposition_return_item ON disposition (return_item_id);
CREATE INDEX idx_disposition_type ON disposition (disposition_type);

-- Partners for disposition channels (recyclers, liquidators, refurbishers)
CREATE TABLE disposition_partner (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(255) NOT NULL,
    partner_type    VARCHAR(50) NOT NULL,         -- recycler, liquidator, refurbisher, recommerce
    certification_type VARCHAR(50),               -- r2v3, e_stewards, weee_approved, none
    certification_expiry DATE,
    contact_email   VARCHAR(255),
    address_id      UUID REFERENCES address(id),
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Warranty & Repair

```sql
-- Warranty associated with a product/order
CREATE TABLE warranty (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    product_id      UUID NOT NULL REFERENCES product(id),
    consumer_id     UUID NOT NULL REFERENCES consumer(id),
    original_order_id UUID REFERENCES original_order(id),
    warranty_type   VARCHAR(50) NOT NULL,         -- manufacturer, extended, third_party
    start_date      DATE NOT NULL,
    end_date        DATE NOT NULL,
    serial_number   VARCHAR(255),
    status          VARCHAR(50) NOT NULL DEFAULT 'active'
                    CHECK (status IN ('active', 'expired', 'claimed', 'voided')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_warranty_product ON warranty (product_id);
CREATE INDEX idx_warranty_consumer ON warranty (consumer_id);

-- Repair jobs (depot repair workflow)
CREATE TABLE repair_job (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    return_item_id  UUID NOT NULL REFERENCES return_item(id),
    warranty_id     UUID REFERENCES warranty(id),
    assigned_to     UUID REFERENCES app_user(id),
    status          VARCHAR(50) NOT NULL DEFAULT 'pending'
                    CHECK (status IN ('pending', 'diagnosed', 'parts_ordered',
                           'in_repair', 'testing', 'completed', 'failed', 'cancelled')),
    diagnosis       TEXT,
    estimated_cost  NUMERIC(12,2),
    actual_cost     NUMERIC(12,2),
    right_to_repair_eligible BOOLEAN NOT NULL DEFAULT FALSE,
    estimated_completion TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_repair_job_return_item ON repair_job (return_item_id);

-- Parts used in a repair
CREATE TABLE repair_part (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    repair_job_id   UUID NOT NULL REFERENCES repair_job(id),
    part_sku        VARCHAR(100) NOT NULL,
    part_name       VARCHAR(255) NOT NULL,
    quantity        INTEGER NOT NULL DEFAULT 1,
    unit_cost       NUMERIC(12,2),
    source          VARCHAR(50),                  -- new, harvested, refurbished
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Sustainability & Compliance

```sql
-- Carbon footprint per return
CREATE TABLE carbon_footprint (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    return_request_id UUID NOT NULL REFERENCES return_request(id),
    transport_kg_co2 NUMERIC(10,4),
    processing_kg_co2 NUMERIC(10,4),
    packaging_kg_co2 NUMERIC(10,4),
    total_kg_co2    NUMERIC(10,4) NOT NULL,
    calculation_method VARCHAR(50) NOT NULL,      -- estimated, measured, carrier_reported
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_carbon_footprint_return ON carbon_footprint (return_request_id);

-- Recycling certificates for WEEE / R2 / e-Stewards compliance
CREATE TABLE recycling_certificate (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    disposition_id  UUID NOT NULL REFERENCES disposition(id),
    certificate_number VARCHAR(255) NOT NULL,
    issuing_body    VARCHAR(255) NOT NULL,
    standard        VARCHAR(50) NOT NULL,         -- weee, r2v3, e_stewards
    issued_date     DATE NOT NULL,
    document_url    VARCHAR(1000),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- GDPR / CCPA data deletion requests
CREATE TABLE data_deletion_request (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    consumer_id     UUID NOT NULL REFERENCES consumer(id),
    regulation      VARCHAR(20) NOT NULL,         -- gdpr, ccpa
    requested_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at    TIMESTAMPTZ,
    status          VARCHAR(50) NOT NULL DEFAULT 'pending'
                    CHECK (status IN ('pending', 'processing', 'completed', 'rejected'))
);
```

## Analytics & Fraud

```sql
-- Fraud detection signals
CREATE TABLE fraud_signal (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    return_request_id UUID NOT NULL REFERENCES return_request(id),
    signal_type     VARCHAR(100) NOT NULL,        -- high_frequency, value_anomaly, serial_mismatch, image_suspicious, wardrobing
    severity        VARCHAR(20) NOT NULL CHECK (severity IN ('low', 'medium', 'high', 'critical')),
    details         TEXT,
    model_version   VARCHAR(50),                  -- AI model version that generated the signal
    confidence      NUMERIC(5,4),
    reviewed_by     UUID REFERENCES app_user(id),
    reviewed_at     TIMESTAMPTZ,
    is_confirmed    BOOLEAN,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_fraud_signal_return ON fraud_signal (return_request_id);
CREATE INDEX idx_fraud_signal_type ON fraud_signal (signal_type);

-- Audit log for all state changes
CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    entity_type     VARCHAR(100) NOT NULL,        -- return_request, inspection, disposition, repair_job
    entity_id       UUID NOT NULL,
    action          VARCHAR(50) NOT NULL,          -- created, updated, status_changed, approved, rejected
    old_values      JSONB,
    new_values      JSONB,
    performed_by    UUID REFERENCES app_user(id),
    ip_address      INET,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_log_entity ON audit_log (entity_type, entity_id);
CREATE INDEX idx_audit_log_tenant ON audit_log (tenant_id, created_at DESC);
```

## Integration & Webhooks

```sql
-- External system integrations per tenant
CREATE TABLE integration (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    platform        VARCHAR(50) NOT NULL,         -- shopify, magento, woocommerce, netsuite, sap
    integration_type VARCHAR(50) NOT NULL,        -- ecommerce, wms, erp, carrier
    credentials     JSONB NOT NULL,               -- Encrypted OAuth tokens or API keys
    webhook_url     VARCHAR(1000),
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    last_sync_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Webhook event deliveries
CREATE TABLE webhook_delivery (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    event_type      VARCHAR(100) NOT NULL,        -- return.created, return.approved, disposition.decided
    payload         JSONB NOT NULL,
    target_url      VARCHAR(1000) NOT NULL,
    http_status     INTEGER,
    response_body   TEXT,
    attempt_count   INTEGER NOT NULL DEFAULT 0,
    delivered_at    TIMESTAMPTZ,
    next_retry_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_webhook_delivery_tenant ON webhook_delivery (tenant_id, created_at DESC);
CREATE INDEX idx_webhook_delivery_retry ON webhook_delivery (next_retry_at) WHERE delivered_at IS NULL;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Identity & Multi-Tenancy | 2 | tenant, app_user |
| Product & Inventory | 2 | product, returnable_asset |
| Consumer & Order | 3 | consumer, address, original_order |
| Return Lifecycle | 3 | return_request, return_item, reason_code |
| Policy Engine | 2 | return_policy, policy_rule |
| Shipping & Logistics | 2 | return_shipment, drop_off_location |
| Inspection & Disposition | 4 | inspection, inspection_photo, disposition, disposition_partner |
| Warranty & Repair | 3 | warranty, repair_job, repair_part |
| Sustainability & Compliance | 3 | carbon_footprint, recycling_certificate, data_deletion_request |
| Analytics & Fraud | 2 | fraud_signal, audit_log |
| Integration | 2 | integration, webhook_delivery |
| **Total** | **28** | |

---

## Key Design Decisions

1. **UUID primary keys everywhere** — enables distributed ID generation, safe cross-system references, and no sequential ID enumeration attacks in a multi-tenant SaaS.

2. **Tenant ID on every data table** — supports row-level security via PostgreSQL RLS policies. Every query can be scoped to a tenant without schema-per-tenant complexity.

3. **Explicit status columns with CHECK constraints** — the return lifecycle states (requested -> approved -> received -> inspecting -> disposition_decided -> resolved) are enforced at the database level, not just in application code.

4. **Separate inspection and disposition tables** — decouples the "what condition is this item in?" question from the "what should we do with it?" question. This allows multiple inspection rounds and disposition changes.

5. **GS1 identifiers as first-class fields** — GTIN on products and GRAI on returnable assets are dedicated columns with indexes, not buried in metadata, because they are the industry-standard interoperability keys.

6. **Fraud signals as a separate table** — rather than a single fraud_score column on the return, individual signals are stored with their type, severity, and model version. This supports explainable AI and human review workflows.

7. **Audit log with old/new JSONB values** — captures every state change for regulatory compliance without requiring event sourcing infrastructure. Trade-off: replay is harder than with a true event store.

8. **Carbon footprint per return** — a dedicated table rather than a column on return_request, because carbon calculation may happen asynchronously and may be revised.

9. **Policy rules as a separate table** — enables a no-code rule engine where merchants configure return policies without code changes. The priority field enables rule precedence.

10. **Integration credentials in JSONB** — different platforms (Shopify API key vs. SAP OAuth token) have different credential shapes. JSONB accommodates this without separate tables per platform. Credentials must be encrypted at the application layer.
