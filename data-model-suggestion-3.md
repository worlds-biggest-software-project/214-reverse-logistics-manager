# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: Reverse Logistics Manager · Created: 2026-05-20

## Philosophy

This model keeps the core return lifecycle in well-typed relational columns for integrity and queryability, but uses JSONB columns extensively for domain areas that vary by tenant, jurisdiction, product category, or integration platform. The key insight is that a reverse logistics platform must serve retailers selling t-shirts in Kansas and manufacturers processing WEEE-regulated electronics in Germany — and their data shapes are fundamentally different.

Rather than creating dozens of nullable columns or separate tables for every jurisdiction-specific field, this model stores variable data in indexed JSONB columns alongside the fixed relational core. PostgreSQL's JSONB support (GIN indexes, containment queries, JSONPath) makes this practical without sacrificing query performance. This is the approach used by modern SaaS platforms like Shopify (metafields), Stripe (metadata), and Zendesk (custom fields) that must serve diverse tenants with a single schema.

The trade-off is that JSONB fields are less self-documenting than explicit columns and require application-level validation (via JSON Schema). But for a platform that must accommodate varied return policies, jurisdiction-specific compliance fields, and diverse integration payloads, this is the most pragmatic path to shipping an MVP that can evolve rapidly.

**Best for:** Teams building an MVP that must serve diverse tenant types (B2B and B2C), multiple jurisdictions, and varied product categories without constant schema migrations.

**Trade-offs:**
- (+) Fast schema evolution — new fields added via JSONB without migrations
- (+) Tenant-specific and jurisdiction-specific data without schema explosion
- (+) Fewer tables than fully normalised (~20 vs ~28)
- (+) GIN indexes on JSONB enable fast containment and existence queries
- (+) Familiar to developers who have worked with Shopify metafields, Stripe metadata, etc.
- (-) JSONB fields lack database-level type enforcement — requires JSON Schema validation in application
- (-) Complex JSONB queries can be harder to optimise than relational JOINs
- (-) Reporting tools may struggle with deeply nested JSONB structures
- (-) Risk of "schema drift" if JSONB structures are not well-documented

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| GS1 GTIN | Relational `product.gtin` column — too important for JSONB |
| GS1 GRAI | Relational column on returnable assets |
| GS1 Digital Link | Stored in `product.extended_attrs` JSONB under `digital_link_url` key |
| ISO 3166-1/2 | Relational `country_code` on addresses; jurisdiction-specific rules in policy JSONB |
| ISO 4217 | Relational `currency` columns on all monetary fields |
| WEEE Directive | JSONB `compliance` object on disposition records: `{"weee_category": "...", "certificate": "..."}` |
| R2v3 / e-Stewards | JSONB `certifications` array on disposition partners |
| EU Right to Repair | JSONB `compliance` on repair records: `{"right_to_repair": true, "documentation": [...]}` |
| JSON Schema Draft 2020-12 | All JSONB column structures are validated via JSON Schema definitions stored in tenant config |
| EU Digital Product Passport | JSONB `dpp_data` on products — structure will evolve as ESPR delegated acts are published |

---

## Core Tables

```sql
-- Tenant with JSONB settings for tenant-specific configuration
CREATE TABLE tenant (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    type            VARCHAR(50) NOT NULL CHECK (type IN ('retailer', 'manufacturer', '3pl', 'brand')),
    default_currency VARCHAR(3) NOT NULL DEFAULT 'USD',
    timezone        VARCHAR(50) NOT NULL DEFAULT 'UTC',
    -- JSONB for variable tenant config
    settings        JSONB NOT NULL DEFAULT '{}',
    /*
    settings example:
    {
        "branding": {"logo_url": "...", "primary_color": "#1a73e8"},
        "notifications": {"email_sender": "returns@brand.com", "sms_enabled": true},
        "fraud": {"auto_reject_threshold": 85, "require_review_threshold": 60},
        "sustainability": {"track_carbon": true, "carbon_method": "carrier_reported"},
        "integrations": {"default_carrier": "ups", "ecommerce_platform": "shopify"},
        "custom_fields_schema": {
            "return_request": { ... JSON Schema for tenant-specific return fields ... },
            "inspection": { ... JSON Schema for tenant-specific inspection fields ... }
        }
    }
    */
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Users
CREATE TABLE app_user (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    email           VARCHAR(255) NOT NULL,
    full_name       VARCHAR(255) NOT NULL,
    role            VARCHAR(50) NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    profile         JSONB NOT NULL DEFAULT '{}',   -- Preferences, permissions, etc.
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, email)
);

CREATE INDEX idx_app_user_tenant ON app_user (tenant_id);
```

## Product Catalog

```sql
CREATE TABLE product (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    sku             VARCHAR(100) NOT NULL,
    gtin            VARCHAR(14),                  -- GS1 GTIN — relational, indexed
    name            VARCHAR(500) NOT NULL,
    category        VARCHAR(255),
    brand           VARCHAR(255),
    unit_cost       NUMERIC(12,2),
    unit_price      NUMERIC(12,2),
    currency        VARCHAR(3) NOT NULL DEFAULT 'USD',
    -- JSONB for variable product attributes
    extended_attrs  JSONB NOT NULL DEFAULT '{}',
    /*
    extended_attrs examples:

    -- Consumer electronics:
    {
        "weight_grams": 450,
        "digital_link_url": "https://id.gs1.org/01/00012345678905",
        "weee_category": "small_equipment",
        "right_to_repair_eligible": true,
        "dpp_data": {"material_composition": [...], "repairability_score": 7.2},
        "warranty_months": 24,
        "hazardous_materials": false
    }

    -- Apparel:
    {
        "weight_grams": 200,
        "size": "M",
        "color": "navy",
        "material": "100% cotton",
        "care_instructions": "Machine wash cold",
        "sustainability_certifications": ["OEKO-TEX", "GOTS"]
    }

    -- General merchandise:
    {
        "weight_grams": 1200,
        "dimensions_cm": {"l": 30, "w": 20, "h": 15}
    }
    */
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, sku)
);

CREATE INDEX idx_product_tenant ON product (tenant_id);
CREATE INDEX idx_product_gtin ON product (gtin) WHERE gtin IS NOT NULL;
CREATE INDEX idx_product_attrs ON product USING GIN (extended_attrs);
```

## Consumer & Orders

```sql
CREATE TABLE consumer (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    external_customer_id VARCHAR(255),
    email           VARCHAR(255) NOT NULL,
    full_name       VARCHAR(255),
    phone           VARCHAR(50),
    -- JSONB for risk profile and variable consumer data
    profile         JSONB NOT NULL DEFAULT '{}',
    /*
    profile example:
    {
        "customer_tier": "vip",
        "total_orders": 42,
        "total_returns": 3,
        "total_returns_value": 450.00,
        "fraud_risk_score": 12.5,
        "addresses": [
            {"type": "shipping", "line1": "...", "city": "...", "country_code": "US", "subdivision_code": "US-CA"}
        ],
        "data_retention_expiry": "2028-05-20T00:00:00Z",
        "consent": {"marketing": false, "analytics": true}
    }
    */
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, email)
);

CREATE INDEX idx_consumer_tenant ON consumer (tenant_id);
CREATE INDEX idx_consumer_external ON consumer (tenant_id, external_customer_id);
CREATE INDEX idx_consumer_profile ON consumer USING GIN (profile);

-- Original order
CREATE TABLE original_order (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    consumer_id     UUID NOT NULL REFERENCES consumer(id),
    external_order_id VARCHAR(255) NOT NULL,
    order_date      TIMESTAMPTZ NOT NULL,
    order_total     NUMERIC(12,2) NOT NULL,
    currency        VARCHAR(3) NOT NULL DEFAULT 'USD',
    channel         VARCHAR(50),
    platform        VARCHAR(50),
    -- JSONB for platform-specific order data
    order_data      JSONB NOT NULL DEFAULT '{}',
    /*
    order_data example (Shopify):
    {
        "shopify_order_id": 12345678,
        "financial_status": "paid",
        "fulfillment_status": "fulfilled",
        "shipping_address": {"line1": "...", "city": "...", "country_code": "US"},
        "line_items": [
            {"shopify_line_item_id": 987, "sku": "WIDGET-BLU-M", "quantity": 2, "price": 29.99}
        ],
        "tags": ["wholesale", "priority"]
    }
    */
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, external_order_id)
);

CREATE INDEX idx_original_order_consumer ON original_order (consumer_id);
CREATE INDEX idx_original_order_data ON original_order USING GIN (order_data);
```

## Return Lifecycle

```sql
CREATE TABLE return_request (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    rma_number      VARCHAR(50) NOT NULL,
    consumer_id     UUID NOT NULL REFERENCES consumer(id),
    original_order_id UUID NOT NULL REFERENCES original_order(id),
    -- Core lifecycle fields: relational for integrity
    status          VARCHAR(50) NOT NULL DEFAULT 'requested'
                    CHECK (status IN ('requested', 'approved', 'rejected',
                           'label_generated', 'in_transit', 'received',
                           'inspecting', 'disposition_decided', 'resolved', 'cancelled')),
    return_type     VARCHAR(50) NOT NULL DEFAULT 'return',
    resolution_type VARCHAR(50),
    initiated_via   VARCHAR(50) DEFAULT 'portal',
    fraud_risk_score NUMERIC(5,2),
    approved_at     TIMESTAMPTZ,
    received_at     TIMESTAMPTZ,
    resolved_at     TIMESTAMPTZ,
    -- Financial summary: relational for aggregation queries
    refund_amount   NUMERIC(12,2),
    store_credit_amount NUMERIC(12,2),
    currency        VARCHAR(3) DEFAULT 'USD',
    -- JSONB for variable/extensible return data
    return_data     JSONB NOT NULL DEFAULT '{}',
    /*
    return_data example:
    {
        "fraud_flags": ["high_frequency", "value_anomaly"],
        "policy_applied": {"id": "uuid", "name": "Standard 30-day", "window_days": 30},
        "shipping": {
            "carrier": "ups",
            "tracking_number": "1Z999AA10123456784",
            "label_url": "https://...",
            "qr_code_url": "https://...",
            "shipping_cost": 8.50,
            "drop_off_location": {"name": "UPS Store #1234", "type": "return_bar"},
            "customs": {"declaration_number": "CUS-2026-001", "duty_drawback_eligible": true}
        },
        "exchange": {
            "exchange_product_sku": "WIDGET-RED-M",
            "exchange_tracking": "1Z999AA10123456799",
            "price_difference": -10.00
        },
        "carbon_footprint": {
            "transport_kg_co2": 2.34,
            "processing_kg_co2": 0.15,
            "total_kg_co2": 2.49,
            "method": "carrier_reported"
        },
        "notes": "Customer called to expedite",
        "custom_fields": {
            "department_code": "ELEC-042",
            "priority_level": "urgent"
        }
    }
    */
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, rma_number)
);

CREATE INDEX idx_return_request_tenant_status ON return_request (tenant_id, status);
CREATE INDEX idx_return_request_consumer ON return_request (consumer_id);
CREATE INDEX idx_return_request_created ON return_request (tenant_id, created_at DESC);
CREATE INDEX idx_return_request_data ON return_request USING GIN (return_data);

-- Return items: relational core + JSONB for inspection/disposition
CREATE TABLE return_item (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    return_request_id UUID NOT NULL REFERENCES return_request(id),
    tenant_id       UUID NOT NULL,
    product_id      UUID REFERENCES product(id),
    sku             VARCHAR(100) NOT NULL,
    quantity        INTEGER NOT NULL DEFAULT 1,
    unit_price      NUMERIC(12,2) NOT NULL,
    reason_code     VARCHAR(100) NOT NULL,
    reason_detail   TEXT,
    serial_number   VARCHAR(255),
    -- JSONB for inspection, grading, disposition, and repair data
    item_data       JSONB NOT NULL DEFAULT '{}',
    /*
    item_data example:
    {
        "inspection": {
            "condition_grade": "fair",
            "is_authentic": true,
            "inspector_id": "uuid",
            "inspected_at": "2026-05-22T10:30:00Z",
            "notes": "Minor cosmetic damage, fully functional",
            "photos": [
                {"url": "https://...", "type": "overview"},
                {"url": "https://...", "type": "damage_detail", "ai_analysis": {"damage_type": "scratch", "confidence": 0.94}}
            ],
            "ai_grading": {
                "predicted_grade": "fair",
                "confidence": 0.92,
                "model_version": "cv-grading-v1.4"
            }
        },
        "disposition": {
            "type": "refurbish",
            "decided_by": "ml_model",
            "ml_confidence": 0.88,
            "partner_id": "uuid",
            "partner_name": "CertifiedRefurb Inc",
            "estimated_recovery": 180.00,
            "actual_recovery": 195.00,
            "decided_at": "2026-05-22T11:00:00Z",
            "completed_at": "2026-05-28T16:00:00Z",
            "compliance": {
                "weee_category": "consumer_electronics",
                "recycling_certificate": "CERT-2026-0042",
                "r2_certified": true
            }
        },
        "repair": {
            "status": "completed",
            "diagnosis": "Display assembly replacement",
            "estimated_cost": 45.00,
            "actual_cost": 42.50,
            "right_to_repair_eligible": true,
            "parts": [
                {"sku": "DISP-ASMB-42", "name": "Display Assembly", "qty": 1, "cost": 28.00, "source": "new"}
            ],
            "completed_at": "2026-05-27T14:00:00Z"
        }
    }
    */
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_return_item_request ON return_item (return_request_id);
CREATE INDEX idx_return_item_tenant ON return_item (tenant_id);
CREATE INDEX idx_return_item_data ON return_item USING GIN (item_data);
```

## Policy Engine

```sql
-- Policies stored as structured JSONB rules
CREATE TABLE return_policy (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(255) NOT NULL,
    is_default      BOOLEAN NOT NULL DEFAULT FALSE,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    -- JSONB for the full policy definition
    policy_def      JSONB NOT NULL,
    /*
    policy_def example:
    {
        "return_window_days": 30,
        "exchange_window_days": 45,
        "requires_receipt": true,
        "free_return_shipping": false,
        "restocking_fee_pct": 0,
        "keep_item": {"enabled": true, "max_value": 15.00},
        "rules": [
            {
                "name": "VIP extended window",
                "conditions": [{"field": "customer_tier", "op": "equals", "value": "vip"}],
                "actions": [{"type": "extend_window", "days": 60}],
                "priority": 10
            },
            {
                "name": "Electronics require photo",
                "conditions": [{"field": "product_category", "op": "in", "value": ["electronics", "appliances"]}],
                "actions": [{"type": "require_photo"}, {"type": "require_serial"}],
                "priority": 5
            },
            {
                "name": "High-value manual review",
                "conditions": [{"field": "item_value", "op": "greater_than", "value": 500}],
                "actions": [{"type": "require_review"}],
                "priority": 20
            },
            {
                "name": "Germany WEEE compliance",
                "conditions": [{"field": "jurisdiction", "op": "equals", "value": "DE"}, {"field": "weee_category", "op": "not_null"}],
                "actions": [{"type": "require_weee_documentation"}, {"type": "route_to_certified_recycler"}],
                "priority": 30
            }
        ]
    }
    */
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_return_policy_tenant ON return_policy (tenant_id);
```

## Disposition Partners

```sql
CREATE TABLE disposition_partner (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(255) NOT NULL,
    partner_type    VARCHAR(50) NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    -- JSONB for certifications, contact info, capabilities
    partner_data    JSONB NOT NULL DEFAULT '{}',
    /*
    partner_data example:
    {
        "certifications": [
            {"type": "r2v3", "number": "R2-2025-1234", "expiry": "2027-03-15"},
            {"type": "e_stewards", "number": "ES-2025-5678", "expiry": "2026-12-31"}
        ],
        "contact": {"email": "ops@refurber.com", "phone": "+1-555-0100"},
        "address": {"line1": "...", "city": "...", "country_code": "US"},
        "capabilities": ["electronics", "appliances", "batteries"],
        "pricing": {"per_unit_fee": 5.00, "revenue_share_pct": 15},
        "regions_served": ["US", "CA", "MX"]
    }
    */
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_disposition_partner_tenant ON disposition_partner (tenant_id);
CREATE INDEX idx_disposition_partner_data ON disposition_partner USING GIN (partner_data);
```

## Integration & Webhooks

```sql
CREATE TABLE integration (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    platform        VARCHAR(50) NOT NULL,
    integration_type VARCHAR(50) NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    -- JSONB for platform-specific config and credentials
    config          JSONB NOT NULL,
    /*
    config example (Shopify):
    {
        "shop_domain": "mystore.myshopify.com",
        "api_key": "encrypted:...",
        "api_secret": "encrypted:...",
        "scopes": ["read_orders", "write_orders"],
        "webhook_topics": ["orders/create", "orders/fulfilled"],
        "sync_frequency_minutes": 15
    }
    */
    last_sync_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Audit log
CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    entity_type     VARCHAR(100) NOT NULL,
    entity_id       UUID NOT NULL,
    action          VARCHAR(50) NOT NULL,
    changes         JSONB NOT NULL,               -- {"field": {"old": ..., "new": ...}}
    performed_by    UUID,
    ip_address      INET,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_log_entity ON audit_log (entity_type, entity_id);
CREATE INDEX idx_audit_log_tenant ON audit_log (tenant_id, created_at DESC);
```

## Example Queries

### Find all returns with WEEE compliance data

```sql
-- GIN index makes this fast even at scale
SELECT r.rma_number, ri.sku, ri.item_data->'disposition'->'compliance'->>'weee_category' as weee_cat
FROM return_item ri
JOIN return_request r ON r.id = ri.return_request_id
WHERE ri.tenant_id = '...'
  AND ri.item_data @> '{"disposition": {"compliance": {"weee_category": "consumer_electronics"}}}';
```

### Find returns with high fraud risk and specific flags

```sql
SELECT rma_number, fraud_risk_score, return_data->'fraud_flags' as flags
FROM return_request
WHERE tenant_id = '...'
  AND fraud_risk_score > 75
  AND return_data @> '{"fraud_flags": ["wardrobing"]}';
```

### Aggregate disposition recovery by type

```sql
SELECT
    ri.item_data->'disposition'->>'type' as disposition_type,
    COUNT(*) as count,
    SUM((ri.item_data->'disposition'->>'actual_recovery')::numeric) as total_recovery
FROM return_item ri
WHERE ri.tenant_id = '...'
  AND ri.item_data ? 'disposition'
  AND ri.item_data->'disposition' ? 'actual_recovery'
GROUP BY ri.item_data->'disposition'->>'type';
```

### Search products by extended attributes

```sql
-- Find all WEEE-eligible products
SELECT sku, name, extended_attrs->>'weee_category' as weee
FROM product
WHERE tenant_id = '...'
  AND extended_attrs @> '{"right_to_repair_eligible": true}';
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Identity & Multi-Tenancy | 2 | tenant, app_user |
| Product | 1 | product (with extended_attrs JSONB) |
| Consumer & Order | 2 | consumer, original_order (both with JSONB profiles) |
| Return Lifecycle | 2 | return_request, return_item (inspection/disposition/repair in JSONB) |
| Policy Engine | 1 | return_policy (rules as JSONB) |
| Disposition Partners | 1 | disposition_partner (certifications in JSONB) |
| Integration & Audit | 2 | integration, audit_log |
| **Total** | **11** | JSONB absorbs what would be ~17 additional tables in a normalised model |

---

## Key Design Decisions

1. **Relational for the lifecycle spine, JSONB for everything variable** — status, rma_number, consumer_id, financial amounts, and timestamps are relational columns because they drive queries, constraints, and aggregations. Shipping details, inspection results, disposition compliance, and repair logs go in JSONB because their structure varies by tenant, product category, and jurisdiction.

2. **GIN indexes on all JSONB columns** — enables containment queries (`@>`) and existence checks (`?`) at scale. The GIN index on `return_item.item_data` supports queries like "find all items with WEEE compliance data" without scanning the full table.

3. **Tenant-defined custom fields via JSON Schema** — the `tenant.settings.custom_fields_schema` stores JSON Schema definitions that the application layer uses to validate custom fields in JSONB columns. This gives tenants extensibility without schema migrations.

4. **Inspection, disposition, and repair as nested JSONB objects on return_item** — in a normalised model these would be 5+ separate tables (inspection, inspection_photo, disposition, repair_job, repair_part). Collapsing them into a single JSONB column on return_item reduces JOINs for the most common read pattern: "show me this item with all its processing history."

5. **Platform-specific order data in JSONB** — a Shopify order has `shopify_order_id` and `financial_status`; a SAP order has a completely different structure. JSONB on `original_order.order_data` accommodates all platforms without platform-specific columns.

6. **Policy rules as JSONB arrays** — the policy engine is a JSON-defined rule set with conditions and actions. This enables no-code policy configuration through a UI without database migrations. Each rule has a priority for deterministic evaluation order.

7. **Carbon footprint as nested JSONB** — rather than a separate table, carbon data lives in `return_request.return_data.carbon_footprint`. This is simpler to query (no JOIN) and reflects that carbon calculation is always in the context of a specific return.

8. **Fewer tables, faster MVP** — 11 tables vs. 28 in the normalised model. Fewer migrations, fewer foreign key constraints to manage, and less boilerplate code. The trade-off is that JSONB structures must be documented and validated at the application layer.

9. **Compliance data is jurisdiction-contextual** — WEEE category, R2 certification, Right to Repair documentation, and recycling certificates are stored in the disposition JSONB under a `compliance` key. Different jurisdictions populate different compliance fields without requiring separate columns for each regulation.

10. **Audit log with JSONB changes** — the audit log captures field-level changes as `{"field_name": {"old": value, "new": value}}` in JSONB. This provides a basic audit trail without the full complexity of event sourcing.
