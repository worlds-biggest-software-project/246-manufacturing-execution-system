# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: Manufacturing Execution System (MES) · Created: 2026-05-22

## Philosophy

This model uses a hybrid approach: core operational fields (IDs, statuses, timestamps, foreign keys, quantities) are stored as typed relational columns with full constraint enforcement, while variable, industry-specific, and extensible fields are stored in JSONB columns. Every major entity has a `properties` JSONB column that absorbs domain-specific variation without requiring DDL changes.

This design is informed by the reality that MES deployments span radically different manufacturing verticals -- automotive, pharmaceutical, food & beverage, electronics, aerospace -- each with unique data requirements. An automotive MES needs PPAP fields and AIAG-specific inspection attributes. A pharma MES needs 21 CFR Part 11 batch record fields and environmental monitoring data. A food manufacturer needs allergen tracking and HACCP fields. Rather than creating vertical-specific tables or adding hundreds of nullable columns, the JSONB approach absorbs this variation in a single, indexed, queryable column.

PostgreSQL's JSONB type provides binary storage with GIN indexing, containment operators (`@>`), and path-based queries (`->>`, `#>>`). This enables the hybrid model to deliver near-relational query performance on JSONB fields while maintaining the schema flexibility of a document store. The pattern is proven at scale in SaaS platforms like Shopify (product metafields), Stripe (metadata), and Salesforce (custom fields).

**Best for:** Multi-vertical MES SaaS deployments where the same platform serves automotive, pharma, food, and electronics manufacturers; rapid MVP development where the schema needs to evolve quickly; and deployments where jurisdiction-specific or customer-specific fields vary widely.

**Trade-offs:**
- (+) Fastest time-to-market: new fields added without migrations or downtime
- (+) Single codebase serves multiple manufacturing verticals without vertical-specific schema branches
- (+) JSONB GIN indexes provide performant queries on semi-structured data
- (+) Customers can define custom fields without application code changes
- (+) Relational columns preserve data integrity for core operational data
- (-) JSONB fields are not type-checked by the database -- validation must happen in the application layer
- (-) Complex JSONB queries can be slower than equivalent relational joins
- (-) Schema documentation requires discipline -- JSONB structures are not self-documenting in the DDL
- (-) Reporting and BI tools may struggle with nested JSONB structures
- (-) Regulatory auditors may question data integrity guarantees of JSONB columns vs. typed columns

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ISA-95 / IEC 62264 | Core ISA-95 entities (work orders, equipment, materials, personnel) use relational columns; ISA-95 extension attributes stored in JSONB `properties` |
| ISA-88 / IEC 61512 | Recipe parameters stored in JSONB arrays on recipe steps, allowing variable parameter sets per recipe type |
| B2MML (MESA) | B2MML import/export maps relational columns directly; JSONB `properties` mapped to B2MML `<Any>` extension elements |
| ISO 22400 | OEE calculation uses relational time-state and count columns; supplementary KPIs (custom plant metrics) stored in JSONB |
| 21 CFR Part 11 | Audit trail is fully relational; e-signature fields are relational; pharma-specific batch record attributes stored in JSONB on work orders |
| IATF 16949 | Automotive-specific PPAP fields, control plan references, and AIAG inspection attributes stored in JSONB on quality records |
| OPC-UA / MQTT | Equipment tag mappings stored in JSONB arrays on equipment records, enabling variable tag sets per machine type |

---

## Tenant & Site Hierarchy

```sql
-- Multi-tenant support with RLS
CREATE TABLE tenant (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(200) NOT NULL,
    slug            VARCHAR(50) UNIQUE NOT NULL,
    industry        VARCHAR(50),               -- 'automotive','pharma','food','electronics','general'
    settings        JSONB NOT NULL DEFAULT '{}',
    -- settings example:
    -- {
    --   "default_uom": "EA",
    --   "oee_target_pct": 0.85,
    --   "require_esignature": true,
    --   "compliance_packs": ["21_cfr_part_11", "eu_annex_11"],
    --   "custom_field_schemas": { ... }
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE site (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(200) NOT NULL,
    country_code    CHAR(2) NOT NULL,          -- ISO 3166-1
    timezone        VARCHAR(50) NOT NULL,
    address         TEXT,
    properties      JSONB NOT NULL DEFAULT '{}',
    -- properties example:
    -- {
    --   "gmp_classification": "Class C",
    --   "clean_room_zones": ["Zone A", "Zone B"],
    --   "energy_provider": "National Grid"
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_site_tenant ON site(tenant_id);

-- Row-Level Security policy (applied to all tenant-scoped tables)
ALTER TABLE site ENABLE ROW LEVEL SECURITY;
CREATE POLICY site_tenant_isolation ON site
    USING (tenant_id = current_setting('app.current_tenant_id')::UUID);

CREATE TABLE area (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    site_id         UUID NOT NULL REFERENCES site(id),
    name            VARCHAR(200) NOT NULL,
    properties      JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_area_site ON area(site_id);

CREATE TABLE work_center (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    area_id         UUID NOT NULL REFERENCES area(id),
    name            VARCHAR(200) NOT NULL,
    properties      JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_wc_area ON work_center(area_id);
```

---

## Equipment Management

```sql
CREATE TABLE equipment (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    work_center_id  UUID REFERENCES work_center(id),
    name            VARCHAR(200) NOT NULL,
    equipment_type  VARCHAR(100) NOT NULL,     -- 'CNC','injection_molder','assembly_station','oven'
    serial_number   VARCHAR(100),
    manufacturer    VARCHAR(200),
    model           VARCHAR(200),
    status          VARCHAR(30) NOT NULL DEFAULT 'idle'
                    CHECK (status IN ('idle','running','down','maintenance','setup')),
    connectivity    JSONB NOT NULL DEFAULT '[]',
    -- connectivity example:
    -- [
    --   {
    --     "protocol": "OPC-UA",
    --     "endpoint": "opc.tcp://10.0.1.50:4840",
    --     "tags": [
    --       {"name": "spindle_speed", "node_id": "ns=2;s=SpindleSpeed", "uom": "RPM"},
    --       {"name": "temperature", "node_id": "ns=2;s=Temp1", "uom": "C"}
    --     ]
    --   },
    --   {
    --     "protocol": "MQTT",
    --     "broker": "mqtt://broker.plant.local:1883",
    --     "topic": "plant/line1/cnc03/#"
    --   }
    -- ]
    capabilities    JSONB NOT NULL DEFAULT '[]',
    -- capabilities example:
    -- [
    --   {"type": "milling", "nominal_rate": 120, "rate_uom": "parts/hr"},
    --   {"type": "drilling", "nominal_rate": 200, "rate_uom": "holes/hr"}
    -- ]
    properties      JSONB NOT NULL DEFAULT '{}',
    -- properties varies by equipment type and industry:
    -- CNC: {"axis_count": 5, "spindle_max_rpm": 24000, "tool_magazine_size": 60}
    -- Injection molder: {"tonnage": 500, "shot_size_oz": 32, "mold_capacity": 4}
    -- Clean room oven: {"max_temp_c": 300, "chamber_volume_l": 500, "clean_room_class": "ISO 7"}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_equip_tenant ON equipment(tenant_id);
CREATE INDEX idx_equip_wc ON equipment(work_center_id);
CREATE INDEX idx_equip_status ON equipment(status);
CREATE INDEX idx_equip_type ON equipment(equipment_type);
CREATE INDEX idx_equip_properties ON equipment USING GIN (properties jsonb_path_ops);
```

---

## Material Management

```sql
CREATE TABLE material_definition (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    part_number     VARCHAR(100) NOT NULL,
    name            VARCHAR(200) NOT NULL,
    material_class  VARCHAR(50) NOT NULL,      -- 'raw_material','component','consumable','packaging'
    uom             VARCHAR(20) NOT NULL,
    is_lot_tracked  BOOLEAN NOT NULL DEFAULT false,
    is_serial_tracked BOOLEAN NOT NULL DEFAULT false,
    properties      JSONB NOT NULL DEFAULT '{}',
    -- properties varies by industry:
    -- Food: {"allergens": ["gluten","dairy"], "kosher": true, "halal": false, "storage_temp_c": "2-8"}
    -- Pharma: {"cas_number": "50-78-2", "pharmacopeia": "USP", "controlled_substance": false}
    -- Electronics: {"rohs_compliant": true, "moisture_sensitivity_level": 3, "esd_class": "Class 1"}
    -- Automotive: {"ppap_level": 3, "hsn_code": "7326.90", "country_of_origin": "DE"}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, part_number)
);
CREATE INDEX idx_mat_def_tenant ON material_definition(tenant_id);
CREATE INDEX idx_mat_def_class ON material_definition(material_class);
CREATE INDEX idx_mat_def_props ON material_definition USING GIN (properties jsonb_path_ops);

CREATE TABLE material_lot (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    material_definition_id UUID NOT NULL REFERENCES material_definition(id),
    lot_number      VARCHAR(100) NOT NULL,
    quantity         NUMERIC(14,4) NOT NULL,
    uom             VARCHAR(20) NOT NULL,
    status          VARCHAR(30) NOT NULL DEFAULT 'available'
                    CHECK (status IN ('available','reserved','in_use','consumed','quarantined','scrapped')),
    site_id         UUID NOT NULL REFERENCES site(id),
    storage_location VARCHAR(100),
    received_at     TIMESTAMPTZ,
    expires_at      TIMESTAMPTZ,
    supplier_name   VARCHAR(200),
    supplier_lot    VARCHAR(100),
    properties      JSONB NOT NULL DEFAULT '{}',
    -- properties example (pharma):
    -- {
    --   "certificate_of_analysis": "COA-2026-0058.pdf",
    --   "assay_pct": 99.7,
    --   "moisture_pct": 0.12,
    --   "microbial_test": "pass",
    --   "quarantine_reason": null
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_mat_lot_tenant ON material_lot(tenant_id);
CREATE INDEX idx_mat_lot_def ON material_lot(material_definition_id);
CREATE INDEX idx_mat_lot_number ON material_lot(lot_number);
CREATE INDEX idx_mat_lot_status ON material_lot(status);
```

---

## Product Definition & BOM

```sql
CREATE TABLE product_definition (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    part_number     VARCHAR(100) NOT NULL,
    name            VARCHAR(200) NOT NULL,
    revision        VARCHAR(20) NOT NULL DEFAULT 'A',
    status          VARCHAR(30) NOT NULL DEFAULT 'active',
    properties      JSONB NOT NULL DEFAULT '{}',
    -- properties example (automotive):
    -- {
    --   "customer_part_number": "CUST-99042",
    --   "ppap_status": "approved",
    --   "ppap_level": 3,
    --   "drawing_number": "DWG-4500-R3",
    --   "weight_kg": 1.25,
    --   "material_grade": "AISI 304"
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, part_number, revision)
);
CREATE INDEX idx_prod_def_tenant ON product_definition(tenant_id);

CREATE TABLE bom (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    product_definition_id UUID NOT NULL REFERENCES product_definition(id),
    revision        VARCHAR(20) NOT NULL DEFAULT 'A',
    status          VARCHAR(30) NOT NULL DEFAULT 'active',
    effective_from  DATE,
    effective_to    DATE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_bom_product ON bom(product_definition_id);

CREATE TABLE bom_item (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    bom_id          UUID NOT NULL REFERENCES bom(id),
    material_definition_id UUID NOT NULL REFERENCES material_definition(id),
    sequence_number INTEGER NOT NULL,
    quantity_per    NUMERIC(14,6) NOT NULL,
    uom             VARCHAR(20) NOT NULL,
    properties      JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_bom_item_bom ON bom_item(bom_id);

CREATE TABLE routing (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    product_definition_id UUID NOT NULL REFERENCES product_definition(id),
    revision        VARCHAR(20) NOT NULL DEFAULT 'A',
    status          VARCHAR(30) NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_routing_product ON routing(product_definition_id);

CREATE TABLE routing_step (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    routing_id      UUID NOT NULL REFERENCES routing(id),
    step_number     INTEGER NOT NULL,
    name            VARCHAR(200) NOT NULL,
    work_center_id  UUID REFERENCES work_center(id),
    equipment_type  VARCHAR(100),
    setup_time_min  NUMERIC(10,2),
    run_time_per_unit_min NUMERIC(10,4),
    parameters      JSONB NOT NULL DEFAULT '[]',
    -- parameters example:
    -- [
    --   {"name": "spindle_speed", "target": 12000, "uom": "RPM", "low": 11500, "high": 12500},
    --   {"name": "feed_rate", "target": 200, "uom": "mm/min", "low": 180, "high": 220},
    --   {"name": "coolant_type", "value": "semi-synthetic", "type": "enum"}
    -- ]
    properties      JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_rs_routing ON routing_step(routing_id);
```

---

## Work Orders & Production

```sql
CREATE TABLE work_order (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    order_number    VARCHAR(50) NOT NULL,
    product_definition_id UUID NOT NULL REFERENCES product_definition(id),
    bom_id          UUID REFERENCES bom(id),
    routing_id      UUID REFERENCES routing(id),
    site_id         UUID NOT NULL REFERENCES site(id),
    planned_quantity NUMERIC(14,4) NOT NULL,
    actual_quantity  NUMERIC(14,4) DEFAULT 0,
    uom             VARCHAR(20) NOT NULL,
    priority        INTEGER NOT NULL DEFAULT 5,
    status          VARCHAR(30) NOT NULL DEFAULT 'planned'
                    CHECK (status IN ('planned','released','dispatched','in_progress',
                                      'paused','completed','closed','cancelled')),
    planned_start   TIMESTAMPTZ,
    planned_end     TIMESTAMPTZ,
    actual_start    TIMESTAMPTZ,
    actual_end      TIMESTAMPTZ,
    erp_order_id    VARCHAR(50),
    properties      JSONB NOT NULL DEFAULT '{}',
    -- properties varies by industry:
    -- Pharma: {
    --   "batch_number": "BATCH-2026-142",
    --   "clean_room_required": "ISO 7",
    --   "environmental_monitoring": true,
    --   "batch_record_status": "open",
    --   "yield_target_pct": 98.5
    -- }
    -- Food: {
    --   "production_line": "Line 3",
    --   "allergen_changeover": true,
    --   "haccp_plan_ref": "HACCP-042",
    --   "best_before_days": 365
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, order_number)
);
CREATE INDEX idx_wo_tenant ON work_order(tenant_id);
CREATE INDEX idx_wo_product ON work_order(product_definition_id);
CREATE INDEX idx_wo_status ON work_order(status);
CREATE INDEX idx_wo_site ON work_order(site_id);
CREATE INDEX idx_wo_planned ON work_order(planned_start);
CREATE INDEX idx_wo_props ON work_order USING GIN (properties jsonb_path_ops);

CREATE TABLE work_order_step (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    work_order_id   UUID NOT NULL REFERENCES work_order(id),
    routing_step_id UUID REFERENCES routing_step(id),
    step_number     INTEGER NOT NULL,
    name            VARCHAR(200) NOT NULL,
    equipment_id    UUID REFERENCES equipment(id),
    assigned_to     UUID,
    status          VARCHAR(30) NOT NULL DEFAULT 'pending'
                    CHECK (status IN ('pending','ready','in_progress','paused','completed','skipped')),
    actual_start    TIMESTAMPTZ,
    actual_end      TIMESTAMPTZ,
    actual_parameters JSONB NOT NULL DEFAULT '[]',
    -- actual_parameters: recorded actuals vs. routing_step targets
    -- [
    --   {"name": "spindle_speed", "actual": 11980, "target": 12000, "uom": "RPM"},
    --   {"name": "feed_rate", "actual": 195, "target": 200, "uom": "mm/min"}
    -- ]
    properties      JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_wos_wo ON work_order_step(work_order_id);
CREATE INDEX idx_wos_equip ON work_order_step(equipment_id);
CREATE INDEX idx_wos_status ON work_order_step(status);

-- Material consumption
CREATE TABLE work_order_material (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    work_order_id   UUID NOT NULL REFERENCES work_order(id),
    work_order_step_id UUID REFERENCES work_order_step(id),
    material_definition_id UUID NOT NULL REFERENCES material_definition(id),
    material_lot_id UUID REFERENCES material_lot(id),
    planned_quantity NUMERIC(14,4) NOT NULL,
    actual_quantity  NUMERIC(14,4) DEFAULT 0,
    uom             VARCHAR(20) NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_wom_wo ON work_order_material(work_order_id);
CREATE INDEX idx_wom_lot ON work_order_material(material_lot_id);
```

---

## OEE & Performance

```sql
CREATE TABLE equipment_state_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    equipment_id    UUID NOT NULL REFERENCES equipment(id),
    work_order_id   UUID REFERENCES work_order(id),
    state           VARCHAR(30) NOT NULL
                    CHECK (state IN ('running','planned_downtime','unplanned_downtime',
                                     'setup','standby','idle','off')),
    started_at      TIMESTAMPTZ NOT NULL,
    ended_at        TIMESTAMPTZ,
    duration_seconds NUMERIC(12,2),
    downtime_reason_code VARCHAR(20),
    downtime_category VARCHAR(50),
    reported_by     UUID,
    notes           TEXT,
    properties      JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_esl_equip_time ON equipment_state_log(equipment_id, started_at);
CREATE INDEX idx_esl_state ON equipment_state_log(state);

CREATE TABLE production_count (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    equipment_id    UUID NOT NULL REFERENCES equipment(id),
    work_order_id   UUID REFERENCES work_order(id),
    period_start    TIMESTAMPTZ NOT NULL,
    period_end      TIMESTAMPTZ NOT NULL,
    total_count     INTEGER NOT NULL DEFAULT 0,
    good_count      INTEGER NOT NULL DEFAULT 0,
    reject_count    INTEGER NOT NULL DEFAULT 0,
    rework_count    INTEGER NOT NULL DEFAULT 0,
    ideal_cycle_time_sec NUMERIC(10,4),
    properties      JSONB NOT NULL DEFAULT '{}',
    -- properties example: {"scrap_reasons": {"burr": 3, "crack": 1, "dimension": 2}}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_pc_equip_time ON production_count(equipment_id, period_start);

-- Downtime reason codes (reference data)
CREATE TABLE downtime_reason (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    category        VARCHAR(50) NOT NULL,
    code            VARCHAR(20) NOT NULL,
    description     VARCHAR(200) NOT NULL,
    is_planned      BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, code)
);
```

---

## Quality Management

```sql
CREATE TABLE inspection_plan (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    product_definition_id UUID NOT NULL REFERENCES product_definition(id),
    name            VARCHAR(200) NOT NULL,
    inspection_type VARCHAR(30) NOT NULL
                    CHECK (inspection_type IN ('incoming','in_process','final','patrol')),
    sampling_method VARCHAR(30) NOT NULL DEFAULT 'every_unit',
    characteristics JSONB NOT NULL DEFAULT '[]',
    -- characteristics: array of what to measure
    -- [
    --   {
    --     "name": "Wall Thickness",
    --     "type": "variable",
    --     "uom": "mm",
    --     "nominal": 2.50,
    --     "usl": 2.55,
    --     "lsl": 2.45,
    --     "is_critical": true
    --   },
    --   {
    --     "name": "Surface Finish",
    --     "type": "attribute",
    --     "acceptance_criteria": "No visible scratches or blemishes"
    --   }
    -- ]
    properties      JSONB NOT NULL DEFAULT '{}',
    -- properties (automotive): {"control_plan_ref": "CP-4500-R2", "ppap_element": "dimensional"}
    -- properties (pharma): {"gmp_requirement": true, "validation_protocol": "VP-042"}
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_ip_tenant ON inspection_plan(tenant_id);
CREATE INDEX idx_ip_product ON inspection_plan(product_definition_id);

CREATE TABLE inspection_result (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    work_order_id   UUID NOT NULL REFERENCES work_order(id),
    inspection_plan_id UUID NOT NULL REFERENCES inspection_plan(id),
    material_lot_id UUID REFERENCES material_lot(id),
    equipment_id    UUID REFERENCES equipment(id),
    inspector_id    UUID NOT NULL,
    overall_result  VARCHAR(20) NOT NULL CHECK (overall_result IN ('pass','fail','conditional')),
    measurements    JSONB NOT NULL DEFAULT '[]',
    -- measurements: array of actual readings
    -- [
    --   {
    --     "characteristic": "Wall Thickness",
    --     "measured_value": 2.51,
    --     "nominal": 2.50,
    --     "usl": 2.55,
    --     "lsl": 2.45,
    --     "in_spec": true
    --   },
    --   {
    --     "characteristic": "Surface Finish",
    --     "result": "pass",
    --     "notes": "Minor tool mark on edge, within acceptance"
    --   }
    -- ]
    inspected_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    properties      JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_ir_wo ON inspection_result(work_order_id);
CREATE INDEX idx_ir_lot ON inspection_result(material_lot_id);
CREATE INDEX idx_ir_date ON inspection_result(inspected_at);
CREATE INDEX idx_ir_result ON inspection_result(overall_result);

CREATE TABLE non_conformance (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    ncr_number      VARCHAR(50) NOT NULL,
    inspection_result_id UUID REFERENCES inspection_result(id),
    work_order_id   UUID REFERENCES work_order(id),
    material_lot_id UUID REFERENCES material_lot(id),
    severity        VARCHAR(20) NOT NULL CHECK (severity IN ('critical','major','minor')),
    status          VARCHAR(30) NOT NULL DEFAULT 'open'
                    CHECK (status IN ('open','investigating','disposition','closed')),
    description     TEXT NOT NULL,
    root_cause      TEXT,
    disposition     VARCHAR(30),
    reported_by     UUID NOT NULL,
    reported_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    properties      JSONB NOT NULL DEFAULT '{}',
    -- properties (automotive): {"8d_report_ref": "8D-2026-042", "customer_complaint_id": "CC-100"}
    -- properties (pharma): {"deviation_number": "DEV-2026-015", "impact_assessment": "low"}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, ncr_number)
);
CREATE INDEX idx_ncr_status ON non_conformance(status);
CREATE INDEX idx_ncr_wo ON non_conformance(work_order_id);

CREATE TABLE capa (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    capa_number     VARCHAR(50) NOT NULL,
    non_conformance_id UUID REFERENCES non_conformance(id),
    capa_type       VARCHAR(20) NOT NULL CHECK (capa_type IN ('corrective','preventive')),
    status          VARCHAR(30) NOT NULL DEFAULT 'open',
    description     TEXT NOT NULL,
    action_plan     TEXT,
    assigned_to     UUID,
    due_date        DATE,
    properties      JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, capa_number)
);
CREATE INDEX idx_capa_ncr ON capa(non_conformance_id);
CREATE INDEX idx_capa_status ON capa(status);
```

---

## Genealogy & Traceability

```sql
CREATE TABLE product_lot (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    product_definition_id UUID NOT NULL REFERENCES product_definition(id),
    work_order_id   UUID NOT NULL REFERENCES work_order(id),
    lot_number      VARCHAR(100) NOT NULL,
    serial_number   VARCHAR(100),
    quantity         NUMERIC(14,4) NOT NULL,
    uom             VARCHAR(20) NOT NULL,
    status          VARCHAR(30) NOT NULL DEFAULT 'in_progress',
    completed_at    TIMESTAMPTZ,
    properties      JSONB NOT NULL DEFAULT '{}',
    -- properties (food): {"best_before": "2027-05-22", "storage_conditions": "2-8C"}
    -- properties (pharma): {"release_status": "pending_qa", "stability_study": "SS-042"}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_pl_product ON product_lot(product_definition_id);
CREATE INDEX idx_pl_wo ON product_lot(work_order_id);
CREATE INDEX idx_pl_lot ON product_lot(lot_number);

CREATE TABLE genealogy_link (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    product_lot_id  UUID NOT NULL REFERENCES product_lot(id),
    material_lot_id UUID NOT NULL REFERENCES material_lot(id),
    quantity_consumed NUMERIC(14,4) NOT NULL,
    uom             VARCHAR(20) NOT NULL,
    work_order_step_id UUID REFERENCES work_order_step(id),
    consumed_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_gen_product ON genealogy_link(product_lot_id);
CREATE INDEX idx_gen_material ON genealogy_link(material_lot_id);
```

---

## Personnel & RBAC

```sql
CREATE TABLE person (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    employee_number VARCHAR(50) NOT NULL,
    first_name      VARCHAR(100) NOT NULL,
    last_name       VARCHAR(100) NOT NULL,
    email           VARCHAR(255),
    site_id         UUID REFERENCES site(id),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    qualifications  JSONB NOT NULL DEFAULT '[]',
    -- qualifications example:
    -- [
    --   {"name": "CNC Operator Level 2", "issued": "2025-06-01", "expires": "2026-06-01"},
    --   {"name": "Forklift License", "issued": "2024-01-15", "expires": null}
    -- ]
    properties      JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, employee_number)
);
CREATE INDEX idx_person_tenant ON person(tenant_id);
CREATE INDEX idx_person_site ON person(site_id);

CREATE TABLE role (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(100) NOT NULL,
    permissions     JSONB NOT NULL DEFAULT '[]',
    -- permissions: ["work_order:create", "work_order:dispatch", "inspection:perform", "report:view"]
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, name)
);

CREATE TABLE person_role (
    person_id       UUID NOT NULL REFERENCES person(id),
    role_id         UUID NOT NULL REFERENCES role(id),
    PRIMARY KEY (person_id, role_id)
);
```

---

## Audit Trail & Compliance

```sql
CREATE TABLE audit_trail (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    table_name      VARCHAR(100) NOT NULL,
    record_id       UUID NOT NULL,
    action          VARCHAR(20) NOT NULL CHECK (action IN ('INSERT','UPDATE','DELETE')),
    changes         JSONB NOT NULL,
    -- changes example:
    -- {
    --   "status": {"old": "in_progress", "new": "completed"},
    --   "actual_quantity": {"old": 480, "new": 500},
    --   "properties.batch_record_status": {"old": "open", "new": "closed"}
    -- }
    reason          TEXT,
    performed_by    UUID NOT NULL,
    performed_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_audit_table_record ON audit_trail(table_name, record_id);
CREATE INDEX idx_audit_date ON audit_trail(performed_at);
CREATE INDEX idx_audit_tenant ON audit_trail(tenant_id);

CREATE TABLE electronic_signature (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    table_name      VARCHAR(100) NOT NULL,
    record_id       UUID NOT NULL,
    signer_id       UUID NOT NULL REFERENCES person(id),
    signature_meaning VARCHAR(50) NOT NULL,
    signed_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    auth_method     VARCHAR(30) NOT NULL,
    reason          TEXT
);
CREATE INDEX idx_esig_record ON electronic_signature(table_name, record_id);
```

---

## Custom Field Definitions

```sql
-- Tenant-defined custom field schemas (validated in application layer)
CREATE TABLE custom_field_definition (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    entity_type     VARCHAR(50) NOT NULL,      -- 'work_order','equipment','material_lot', etc.
    field_name      VARCHAR(100) NOT NULL,
    field_label     VARCHAR(200) NOT NULL,
    field_type      VARCHAR(20) NOT NULL CHECK (field_type IN ('text','number','date','boolean','enum','url')),
    is_required     BOOLEAN NOT NULL DEFAULT false,
    enum_values     JSONB,                     -- for enum type: ["Option A", "Option B", "Option C"]
    display_order   INTEGER NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, entity_type, field_name)
);
CREATE INDEX idx_cfd_tenant_entity ON custom_field_definition(tenant_id, entity_type);
```

---

## Example Queries

### Find all pharma work orders requiring environmental monitoring

```sql
SELECT wo.order_number, wo.status, wo.planned_start,
       wo.properties->>'batch_number' AS batch_number,
       wo.properties->>'clean_room_required' AS clean_room
FROM work_order wo
WHERE wo.tenant_id = :tenant_id
  AND wo.properties @> '{"environmental_monitoring": true}'
  AND wo.status IN ('released', 'in_progress')
ORDER BY wo.planned_start;
```

### Find all RoHS-compliant materials

```sql
SELECT md.part_number, md.name, md.properties->>'moisture_sensitivity_level' AS msl
FROM material_definition md
WHERE md.tenant_id = :tenant_id
  AND md.properties @> '{"rohs_compliant": true}'
ORDER BY md.part_number;
```

### Genealogy trace with lot properties

```sql
SELECT pl.lot_number AS product_lot,
       ml.lot_number AS material_lot,
       md.part_number,
       gl.quantity_consumed,
       ml.properties->>'certificate_of_analysis' AS coa
FROM genealogy_link gl
JOIN product_lot pl ON gl.product_lot_id = pl.id
JOIN material_lot ml ON gl.material_lot_id = ml.id
JOIN material_definition md ON ml.material_definition_id = md.id
WHERE pl.id = :product_lot_id;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Tenant & Site Hierarchy | 4 | tenant, site, area, work_center |
| Equipment Management | 1 | Single table with JSONB connectivity/capabilities/properties |
| Material Management | 2 | material_definition, material_lot |
| Product Definition & BOM | 5 | Products, BOMs, BOM items, routings, routing steps |
| Work Orders & Production | 3 | work_order, work_order_step, work_order_material |
| OEE & Performance | 3 | equipment_state_log, production_count, downtime_reason |
| Quality Management | 4 | inspection_plan, inspection_result, non_conformance, capa |
| Genealogy & Traceability | 2 | product_lot, genealogy_link |
| Personnel & RBAC | 3 | person, role, person_role |
| Audit & Compliance | 2 | audit_trail, electronic_signature |
| Custom Fields | 1 | custom_field_definition |
| **Total** | **~30** | Significantly fewer than normalized model; JSONB absorbs variation |

---

## Key Design Decisions

1. **Every major entity has a `properties` JSONB column.** This is the extensibility mechanism. Industry-specific fields (pharma batch records, automotive PPAP attributes, food allergen data) live in `properties` rather than in vertical-specific columns or tables. The application layer validates JSONB content against `custom_field_definition` schemas.

2. **GIN indexes on JSONB for query performance.** The `jsonb_path_ops` GIN index class supports containment queries (`@>`) efficiently. This enables queries like "find all materials where RoHS is true" without scanning the entire table.

3. **Multi-tenant via `tenant_id` + PostgreSQL RLS.** Every tenant-scoped table has a `tenant_id` column. Row-Level Security policies ensure data isolation at the database level, even if application code has bugs. The `current_setting('app.current_tenant_id')` approach sets the tenant context per database session.

4. **Inspection characteristics in JSONB arrays.** Rather than a separate `inspection_characteristic` table, characteristics are stored as a JSONB array on `inspection_plan`. Similarly, measurements are a JSONB array on `inspection_result`. This trades normalization for developer ergonomics -- a single API call returns the full inspection plan with all characteristics.

5. **Equipment connectivity as JSONB.** OPC-UA endpoints, MQTT brokers, and tag mappings are stored in a JSONB array on the equipment record. This accommodates variable tag sets per machine type without a separate tag configuration table for every protocol.

6. **Recipe parameters as JSONB on routing steps.** Process parameters (spindle speed, temperature, pressure) are stored as JSONB arrays on routing steps. This avoids a separate parameter table and allows parameters to vary freely by step type.

7. **Audit trail captures JSONB changes.** The `changes` column in the audit trail stores a diff of what changed, including changes to JSONB `properties` fields. This provides field-level change tracking for both relational and JSONB columns.

8. **~30 tables vs. ~59 in the normalized model.** The JSONB approach consolidates what would be 10-15 separate lookup/detail tables into JSONB columns on parent entities. This reduces migration complexity and API surface area at the cost of some type safety.

9. **Custom field definitions table.** Tenants can define their own custom fields per entity type. The application UI renders these dynamically, and the API validates JSONB content against the definitions. This replaces Salesforce-style custom objects with a simpler, JSONB-native approach.

10. **Relational columns for everything that gets filtered, sorted, or joined.** Status, dates, foreign keys, quantities, and identifiers remain relational columns. JSONB is reserved for descriptive, variable, and industry-specific attributes. This hybrid preserves query performance for operational queries while enabling flexibility for domain variation.
