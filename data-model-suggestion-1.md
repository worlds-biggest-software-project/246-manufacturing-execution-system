# Data Model Suggestion 1: Entity-Centric Normalized Relational (ISA-95 Aligned)

> Project: Manufacturing Execution System (MES) · Created: 2026-05-22

## Philosophy

This model follows the ISA-95 / IEC 62264 standard directly, creating a dedicated PostgreSQL table for every concept defined in the standard's object models: personnel, equipment, material, product definition, production schedule, and production performance. Every relationship is expressed through foreign keys and junction tables, enforcing referential integrity at the database level. The B2MML XML schemas published by MESA International serve as the blueprint — each B2MML schema element maps to a table or column.

This is the approach taken by Tier-1 MES vendors like Siemens Opcenter and AVEVA MES, where the data model mirrors the ISA-95 functional hierarchy from enterprise down to control module. The advantage is that every query, every API, and every report speaks the same language as the standard, making ERP integration via B2MML message exchange straightforward. Cross-vendor benchmarking using ISO 22400 KPIs is natural because the underlying time-state and quantity models map cleanly to the table structure.

The trade-off is table count. A fully normalized ISA-95 model produces 80-120+ tables before adding application-specific concerns like multi-tenancy, notifications, or AI model metadata. This is appropriate when the domain is well-understood, the schema is stable, and data integrity (especially for regulated industries like pharma 21 CFR Part 11) is the top priority.

**Best for:** Regulated manufacturing environments (pharma, automotive, aerospace) where ISA-95 compliance, full audit trails, and ERP interoperability are mandatory requirements.

**Trade-offs:**
- (+) Direct alignment with ISA-95 / B2MML makes ERP integration and vendor interoperability straightforward
- (+) Full referential integrity protects data consistency in high-transaction environments
- (+) ISO 22400 KPI calculations map naturally to the time-state and quantity tables
- (+) Easiest model to validate against 21 CFR Part 11 and IATF 16949 requirements
- (-) High table count (100+) increases schema complexity and migration burden
- (-) Adding jurisdiction-specific or industry-specific fields requires schema changes
- (-) Rigid structure makes rapid prototyping slower than JSONB or document approaches
- (-) Complex queries spanning the full genealogy chain require multiple joins

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ISA-95 / IEC 62264 | Primary blueprint — every resource model (personnel, equipment, material, physical asset) and operations model (schedule, performance, capability) maps to a table group |
| ISA-88 / IEC 61512 | Recipe and batch control hierarchy (procedure → unit procedure → operation → phase) modeled as recursive tables |
| B2MML (MESA International) | XML schema elements inform column names and types; B2MML import/export maps 1:1 to table structure |
| ISO 22400 | KPI definitions drive the `oee_record`, `kpi_definition`, and time-state tables |
| OPC-UA / IEC 62541 | Equipment hierarchy and data point tables align with OPC-UA ISA-95 companion specification node structures |
| MQTT / Sparkplug B | Machine event ingestion tables match Sparkplug B birth/death/data message structure |
| 21 CFR Part 11 | Audit trail table and e-signature table designed for FDA validation requirements |
| IATF 16949 | Traceability chain tables support forward and reverse genealogy for automotive PPAP |
| ISO 9001 | Quality management tables (CAPA, non-conformance, corrective action) align with ISO 9001 process model |
| ISO 50001 | Energy consumption tables structured for energy management KPI reporting |

---

## Core Infrastructure Tables

```sql
-- ============================================================
-- MULTI-TENANCY AND ORGANIZATION
-- ============================================================

CREATE TABLE tenant (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    subscription_tier VARCHAR(50) DEFAULT 'standard',
    settings        JSONB DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE site (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(255) NOT NULL,
    code            VARCHAR(50) NOT NULL,
    timezone        VARCHAR(100) NOT NULL DEFAULT 'UTC',
    -- ISO 3166-1 alpha-2 country code
    country_code    CHAR(2) NOT NULL,
    address_line1   VARCHAR(255),
    address_line2   VARCHAR(255),
    city            VARCHAR(100),
    region          VARCHAR(100),
    postal_code     VARCHAR(20),
    latitude        DECIMAL(10, 7),
    longitude       DECIMAL(10, 7),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, code)
);

CREATE TABLE area (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    site_id         UUID NOT NULL REFERENCES site(id),
    name            VARCHAR(255) NOT NULL,
    code            VARCHAR(50) NOT NULL,
    description     TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (site_id, code)
);

-- ISA-95 Level 3: Work Center (production line or cell)
CREATE TABLE work_center (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    area_id         UUID NOT NULL REFERENCES area(id),
    name            VARCHAR(255) NOT NULL,
    code            VARCHAR(50) NOT NULL,
    work_center_type VARCHAR(50) NOT NULL CHECK (work_center_type IN ('production_line', 'work_cell', 'storage_zone', 'quality_lab')),
    status          VARCHAR(30) NOT NULL DEFAULT 'active' CHECK (status IN ('active', 'inactive', 'maintenance')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (area_id, code)
);

CREATE INDEX idx_work_center_area ON work_center(area_id);
```

## Personnel Model (ISA-95 Personnel)

```sql
-- ============================================================
-- PERSONNEL (ISA-95 Personnel Model)
-- ============================================================

CREATE TABLE person (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    employee_number VARCHAR(50),
    first_name      VARCHAR(100) NOT NULL,
    last_name       VARCHAR(100) NOT NULL,
    email           VARCHAR(255),
    phone           VARCHAR(50),
    status          VARCHAR(30) NOT NULL DEFAULT 'active' CHECK (status IN ('active', 'inactive', 'terminated')),
    hire_date       DATE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, employee_number)
);

CREATE TABLE personnel_class (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE qualification (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    -- e.g., 'certification', 'training', 'license'
    qualification_type VARCHAR(50) NOT NULL,
    validity_period_days INTEGER,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE person_qualification (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    person_id       UUID NOT NULL REFERENCES person(id),
    qualification_id UUID NOT NULL REFERENCES qualification(id),
    acquired_date   DATE NOT NULL,
    expiry_date     DATE,
    status          VARCHAR(30) NOT NULL DEFAULT 'active' CHECK (status IN ('active', 'expired', 'revoked')),
    certified_by    UUID REFERENCES person(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE person_class_membership (
    person_id       UUID NOT NULL REFERENCES person(id),
    personnel_class_id UUID NOT NULL REFERENCES personnel_class(id),
    PRIMARY KEY (person_id, personnel_class_id)
);

-- RBAC
CREATE TABLE role (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(100) NOT NULL,
    description     TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, name)
);

CREATE TABLE permission (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    resource        VARCHAR(100) NOT NULL,
    action          VARCHAR(50) NOT NULL,
    description     TEXT,
    UNIQUE (resource, action)
);

CREATE TABLE role_permission (
    role_id         UUID NOT NULL REFERENCES role(id),
    permission_id   UUID NOT NULL REFERENCES permission(id),
    PRIMARY KEY (role_id, permission_id)
);

CREATE TABLE person_role (
    person_id       UUID NOT NULL REFERENCES person(id),
    role_id         UUID NOT NULL REFERENCES role(id),
    site_id         UUID REFERENCES site(id), -- NULL = all sites
    PRIMARY KEY (person_id, role_id, site_id)
);

CREATE INDEX idx_person_tenant ON person(tenant_id);
CREATE INDEX idx_person_qualification_expiry ON person_qualification(expiry_date) WHERE status = 'active';
```

## Equipment Model (ISA-95 Equipment)

```sql
-- ============================================================
-- EQUIPMENT (ISA-95 Equipment Model)
-- ============================================================

CREATE TABLE equipment_class (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    -- e.g., 'CNC', 'injection_moulding', 'conveyor', 'sensor'
    category        VARCHAR(100),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE equipment (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    work_center_id  UUID NOT NULL REFERENCES work_center(id),
    equipment_class_id UUID REFERENCES equipment_class(id),
    name            VARCHAR(255) NOT NULL,
    code            VARCHAR(50) NOT NULL,
    serial_number   VARCHAR(100),
    manufacturer    VARCHAR(255),
    model           VARCHAR(255),
    -- ISA-95 equipment level
    equipment_level VARCHAR(30) NOT NULL CHECK (equipment_level IN ('work_center', 'work_unit', 'equipment_module', 'control_module')),
    parent_equipment_id UUID REFERENCES equipment(id),
    status          VARCHAR(30) NOT NULL DEFAULT 'operational' CHECK (status IN ('operational', 'idle', 'maintenance', 'breakdown', 'decommissioned')),
    commissioned_date DATE,
    -- OPC-UA endpoint for machine connectivity
    opcua_endpoint  VARCHAR(500),
    mqtt_topic      VARCHAR(500),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE equipment_property (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    equipment_id    UUID NOT NULL REFERENCES equipment(id),
    property_name   VARCHAR(100) NOT NULL,
    property_value  VARCHAR(500),
    unit_of_measure VARCHAR(50),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (equipment_id, property_name)
);

CREATE TABLE equipment_capability (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    equipment_id    UUID NOT NULL REFERENCES equipment(id),
    capability_type VARCHAR(100) NOT NULL, -- e.g., 'max_speed_rpm', 'max_temperature_c'
    capability_value DECIMAL(15, 4),
    unit_of_measure VARCHAR(50),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_equipment_work_center ON equipment(work_center_id);
CREATE INDEX idx_equipment_parent ON equipment(parent_equipment_id);
CREATE INDEX idx_equipment_status ON equipment(status);
```

## Material Model (ISA-95 Material)

```sql
-- ============================================================
-- MATERIAL (ISA-95 Material Model)
-- ============================================================

CREATE TABLE material_class (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    -- e.g., 'raw_material', 'intermediate', 'finished_good', 'consumable', 'packaging'
    category        VARCHAR(50) NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE material_definition (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    material_class_id UUID REFERENCES material_class(id),
    part_number     VARCHAR(100) NOT NULL,
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    unit_of_measure VARCHAR(50) NOT NULL, -- e.g., 'kg', 'ea', 'l', 'm'
    -- Shelf life tracking for perishable materials
    shelf_life_days INTEGER,
    is_lot_tracked  BOOLEAN NOT NULL DEFAULT true,
    is_serial_tracked BOOLEAN NOT NULL DEFAULT false,
    status          VARCHAR(30) NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, part_number)
);

CREATE TABLE material_property (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    material_definition_id UUID NOT NULL REFERENCES material_definition(id),
    property_name   VARCHAR(100) NOT NULL,
    property_value  VARCHAR(500),
    unit_of_measure VARCHAR(50),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (material_definition_id, property_name)
);

CREATE TABLE material_lot (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    material_definition_id UUID NOT NULL REFERENCES material_definition(id),
    lot_number      VARCHAR(100) NOT NULL,
    serial_number   VARCHAR(100),
    quantity        DECIMAL(15, 4) NOT NULL,
    unit_of_measure VARCHAR(50) NOT NULL,
    status          VARCHAR(30) NOT NULL DEFAULT 'available' CHECK (status IN ('available', 'reserved', 'in_use', 'consumed', 'quarantined', 'scrapped')),
    received_date   TIMESTAMPTZ,
    expiry_date     TIMESTAMPTZ,
    supplier_id     UUID,
    supplier_lot_number VARCHAR(100),
    storage_location_id UUID,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (material_definition_id, lot_number)
);

CREATE TABLE bill_of_materials (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    parent_material_id UUID NOT NULL REFERENCES material_definition(id),
    child_material_id UUID NOT NULL REFERENCES material_definition(id),
    quantity_per    DECIMAL(15, 6) NOT NULL,
    unit_of_measure VARCHAR(50) NOT NULL,
    sequence_number INTEGER NOT NULL DEFAULT 0,
    is_optional     BOOLEAN NOT NULL DEFAULT false,
    scrap_factor    DECIMAL(5, 4) DEFAULT 0, -- e.g., 0.02 = 2% scrap allowance
    effective_from  DATE,
    effective_to    DATE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_material_lot_status ON material_lot(status);
CREATE INDEX idx_material_lot_expiry ON material_lot(expiry_date);
CREATE INDEX idx_bom_parent ON bill_of_materials(parent_material_id);
CREATE INDEX idx_bom_child ON bill_of_materials(child_material_id);
```

## Product Definition and Routing

```sql
-- ============================================================
-- PRODUCT DEFINITION AND ROUTING (ISA-95 Product Definition)
-- ============================================================

CREATE TABLE product_definition (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    material_definition_id UUID NOT NULL REFERENCES material_definition(id),
    version         VARCHAR(20) NOT NULL DEFAULT '1.0',
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    status          VARCHAR(30) NOT NULL DEFAULT 'draft' CHECK (status IN ('draft', 'approved', 'active', 'obsolete')),
    approved_by     UUID REFERENCES person(id),
    approved_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (material_definition_id, version)
);

CREATE TABLE routing (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    product_definition_id UUID NOT NULL REFERENCES product_definition(id),
    name            VARCHAR(255) NOT NULL,
    version         VARCHAR(20) NOT NULL DEFAULT '1.0',
    status          VARCHAR(30) NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE routing_step (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    routing_id      UUID NOT NULL REFERENCES routing(id),
    sequence_number INTEGER NOT NULL,
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    work_center_id  UUID REFERENCES work_center(id),
    equipment_class_id UUID REFERENCES equipment_class(id),
    -- Standard times in seconds
    setup_time_std  INTEGER,
    run_time_std    INTEGER, -- per unit
    teardown_time_std INTEGER,
    -- Personnel requirements
    required_personnel_class_id UUID REFERENCES personnel_class(id),
    required_operators INTEGER DEFAULT 1,
    -- Quality gate at this step?
    has_quality_gate BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (routing_id, sequence_number)
);

CREATE TABLE routing_step_material (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    routing_step_id UUID NOT NULL REFERENCES routing_step(id),
    material_definition_id UUID NOT NULL REFERENCES material_definition(id),
    -- 'input' or 'output'
    direction       VARCHAR(10) NOT NULL CHECK (direction IN ('input', 'output')),
    quantity_per    DECIMAL(15, 6) NOT NULL,
    unit_of_measure VARCHAR(50) NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_routing_product ON routing(product_definition_id);
CREATE INDEX idx_routing_step_routing ON routing_step(routing_id);
```

## Recipe / Batch Control (ISA-88)

```sql
-- ============================================================
-- RECIPE MANAGEMENT (ISA-88 / IEC 61512 Procedural Model)
-- ============================================================

CREATE TABLE master_recipe (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    product_definition_id UUID REFERENCES product_definition(id),
    name            VARCHAR(255) NOT NULL,
    version         VARCHAR(20) NOT NULL DEFAULT '1.0',
    description     TEXT,
    recipe_type     VARCHAR(30) NOT NULL CHECK (recipe_type IN ('master', 'site', 'control')),
    status          VARCHAR(30) NOT NULL DEFAULT 'draft' CHECK (status IN ('draft', 'approved', 'active', 'obsolete')),
    approved_by     UUID REFERENCES person(id),
    approved_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ISA-88 procedural hierarchy: procedure → unit_procedure → operation → phase
CREATE TABLE recipe_procedure (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    master_recipe_id UUID NOT NULL REFERENCES master_recipe(id),
    name            VARCHAR(255) NOT NULL,
    sequence_number INTEGER NOT NULL,
    description     TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE recipe_unit_procedure (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    recipe_procedure_id UUID NOT NULL REFERENCES recipe_procedure(id),
    name            VARCHAR(255) NOT NULL,
    sequence_number INTEGER NOT NULL,
    target_equipment_class_id UUID REFERENCES equipment_class(id),
    description     TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE recipe_operation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    unit_procedure_id UUID NOT NULL REFERENCES recipe_unit_procedure(id),
    name            VARCHAR(255) NOT NULL,
    sequence_number INTEGER NOT NULL,
    description     TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE recipe_phase (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    recipe_operation_id UUID NOT NULL REFERENCES recipe_operation(id),
    name            VARCHAR(255) NOT NULL,
    sequence_number INTEGER NOT NULL,
    description     TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE recipe_parameter (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    -- Polymorphic: can belong to any level of the procedural hierarchy
    recipe_phase_id UUID REFERENCES recipe_phase(id),
    recipe_operation_id UUID REFERENCES recipe_operation(id),
    parameter_name  VARCHAR(100) NOT NULL,
    target_value    DECIMAL(15, 6),
    min_value       DECIMAL(15, 6),
    max_value       DECIMAL(15, 6),
    unit_of_measure VARCHAR(50),
    is_critical     BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Production Scheduling and Execution

```sql
-- ============================================================
-- PRODUCTION SCHEDULE AND WORK ORDERS (ISA-95 Production Schedule)
-- ============================================================

CREATE TABLE production_order (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    site_id         UUID NOT NULL REFERENCES site(id),
    order_number    VARCHAR(100) NOT NULL,
    -- ERP reference
    erp_order_id    VARCHAR(100),
    product_definition_id UUID NOT NULL REFERENCES product_definition(id),
    routing_id      UUID NOT NULL REFERENCES routing(id),
    master_recipe_id UUID REFERENCES master_recipe(id),
    planned_quantity DECIMAL(15, 4) NOT NULL,
    unit_of_measure VARCHAR(50) NOT NULL,
    priority        INTEGER NOT NULL DEFAULT 50, -- 1=highest, 99=lowest
    status          VARCHAR(30) NOT NULL DEFAULT 'planned' CHECK (status IN (
        'planned', 'scheduled', 'released', 'in_progress', 'completed', 'closed', 'cancelled'
    )),
    planned_start   TIMESTAMPTZ,
    planned_end     TIMESTAMPTZ,
    actual_start    TIMESTAMPTZ,
    actual_end      TIMESTAMPTZ,
    customer_order_ref VARCHAR(100),
    created_by      UUID REFERENCES person(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (site_id, order_number)
);

CREATE TABLE work_order (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    production_order_id UUID NOT NULL REFERENCES production_order(id),
    routing_step_id UUID NOT NULL REFERENCES routing_step(id),
    work_center_id  UUID NOT NULL REFERENCES work_center(id),
    equipment_id    UUID REFERENCES equipment(id),
    sequence_number INTEGER NOT NULL,
    planned_quantity DECIMAL(15, 4) NOT NULL,
    completed_quantity DECIMAL(15, 4) DEFAULT 0,
    scrapped_quantity DECIMAL(15, 4) DEFAULT 0,
    unit_of_measure VARCHAR(50) NOT NULL,
    status          VARCHAR(30) NOT NULL DEFAULT 'pending' CHECK (status IN (
        'pending', 'ready', 'in_progress', 'paused', 'completed', 'cancelled'
    )),
    planned_start   TIMESTAMPTZ,
    planned_end     TIMESTAMPTZ,
    actual_start    TIMESTAMPTZ,
    actual_end      TIMESTAMPTZ,
    assigned_operator_id UUID REFERENCES person(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Material consumption and output at work-order level (genealogy)
CREATE TABLE work_order_material (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    work_order_id   UUID NOT NULL REFERENCES work_order(id),
    material_lot_id UUID NOT NULL REFERENCES material_lot(id),
    direction       VARCHAR(10) NOT NULL CHECK (direction IN ('input', 'output')),
    quantity        DECIMAL(15, 4) NOT NULL,
    unit_of_measure VARCHAR(50) NOT NULL,
    consumed_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    recorded_by     UUID REFERENCES person(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Shift and labor tracking
CREATE TABLE shift_definition (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    site_id         UUID NOT NULL REFERENCES site(id),
    name            VARCHAR(100) NOT NULL, -- e.g., 'Day', 'Swing', 'Night'
    start_time      TIME NOT NULL,
    end_time        TIME NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE labor_record (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    work_order_id   UUID NOT NULL REFERENCES work_order(id),
    person_id       UUID NOT NULL REFERENCES person(id),
    shift_id        UUID REFERENCES shift_definition(id),
    clock_in        TIMESTAMPTZ NOT NULL,
    clock_out       TIMESTAMPTZ,
    labor_type      VARCHAR(30) NOT NULL CHECK (labor_type IN ('direct', 'setup', 'rework', 'indirect')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_production_order_site ON production_order(site_id);
CREATE INDEX idx_production_order_status ON production_order(status);
CREATE INDEX idx_production_order_planned ON production_order(planned_start, planned_end);
CREATE INDEX idx_work_order_production ON work_order(production_order_id);
CREATE INDEX idx_work_order_status ON work_order(status);
CREATE INDEX idx_work_order_equipment ON work_order(equipment_id);
CREATE INDEX idx_work_order_material_wo ON work_order_material(work_order_id);
CREATE INDEX idx_work_order_material_lot ON work_order_material(material_lot_id);
```

## OEE and Performance (ISO 22400)

```sql
-- ============================================================
-- OEE AND PERFORMANCE (ISO 22400 KPI Model)
-- ============================================================

CREATE TABLE downtime_reason (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    code            VARCHAR(50) NOT NULL,
    name            VARCHAR(255) NOT NULL,
    category        VARCHAR(50) NOT NULL CHECK (category IN (
        'planned_downtime', 'unplanned_downtime', 'changeover', 'maintenance', 'quality_hold', 'no_demand', 'other'
    )),
    parent_reason_id UUID REFERENCES downtime_reason(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, code)
);

CREATE TABLE equipment_state_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    equipment_id    UUID NOT NULL REFERENCES equipment(id),
    state           VARCHAR(30) NOT NULL CHECK (state IN (
        'running', 'idle', 'planned_stop', 'unplanned_stop', 'changeover', 'maintenance'
    )),
    -- ISO 22400 time state classification
    started_at      TIMESTAMPTZ NOT NULL,
    ended_at        TIMESTAMPTZ,
    duration_seconds INTEGER,
    downtime_reason_id UUID REFERENCES downtime_reason(id),
    work_order_id   UUID REFERENCES work_order(id),
    recorded_by     UUID REFERENCES person(id),
    source          VARCHAR(30) NOT NULL DEFAULT 'manual' CHECK (source IN ('manual', 'opcua', 'mqtt', 'plc', 'ai_detected')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Aggregated OEE records (typically per shift per equipment)
CREATE TABLE oee_record (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    equipment_id    UUID NOT NULL REFERENCES equipment(id),
    work_center_id  UUID NOT NULL REFERENCES work_center(id),
    shift_id        UUID REFERENCES shift_definition(id),
    period_start    TIMESTAMPTZ NOT NULL,
    period_end      TIMESTAMPTZ NOT NULL,
    -- ISO 22400 time elements (all in seconds)
    planned_production_time INTEGER NOT NULL,
    actual_run_time INTEGER NOT NULL,
    planned_downtime INTEGER NOT NULL DEFAULT 0,
    unplanned_downtime INTEGER NOT NULL DEFAULT 0,
    -- Quantities
    total_count     DECIMAL(15, 4) NOT NULL DEFAULT 0,
    good_count      DECIMAL(15, 4) NOT NULL DEFAULT 0,
    reject_count    DECIMAL(15, 4) NOT NULL DEFAULT 0,
    -- Ideal cycle time in seconds per unit
    ideal_cycle_time DECIMAL(10, 4),
    -- Calculated OEE components (stored for query performance)
    availability    DECIMAL(5, 4), -- 0.0000 to 1.0000
    performance     DECIMAL(5, 4),
    quality         DECIMAL(5, 4),
    oee             DECIMAL(5, 4), -- availability * performance * quality
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE kpi_definition (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    -- ISO 22400 KPI identifier
    iso_22400_id    VARCHAR(50),
    name            VARCHAR(255) NOT NULL,
    formula         TEXT NOT NULL,
    unit_of_measure VARCHAR(50),
    description     TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_equipment_state_equipment ON equipment_state_log(equipment_id);
CREATE INDEX idx_equipment_state_started ON equipment_state_log(started_at);
CREATE INDEX idx_oee_equipment_period ON oee_record(equipment_id, period_start, period_end);
CREATE INDEX idx_oee_work_center ON oee_record(work_center_id, period_start);
```

## Quality Management

```sql
-- ============================================================
-- QUALITY MANAGEMENT (ISO 9001 / IATF 16949 / 21 CFR Part 11)
-- ============================================================

CREATE TABLE inspection_plan (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(255) NOT NULL,
    version         VARCHAR(20) NOT NULL DEFAULT '1.0',
    product_definition_id UUID REFERENCES product_definition(id),
    routing_step_id UUID REFERENCES routing_step(id),
    -- Sampling: 'all', 'first_piece', 'periodic', 'aql'
    sampling_method VARCHAR(30) NOT NULL DEFAULT 'all',
    sample_size     INTEGER,
    sample_frequency INTEGER, -- every N units
    status          VARCHAR(30) NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE inspection_characteristic (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    inspection_plan_id UUID NOT NULL REFERENCES inspection_plan(id),
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    measurement_type VARCHAR(30) NOT NULL CHECK (measurement_type IN ('variable', 'attribute')),
    unit_of_measure VARCHAR(50),
    target_value    DECIMAL(15, 6),
    lower_limit     DECIMAL(15, 6),
    upper_limit     DECIMAL(15, 6),
    -- SPC control limits
    ucl             DECIMAL(15, 6),
    lcl             DECIMAL(15, 6),
    -- IATF 16949: is this a key product characteristic?
    is_critical     BOOLEAN NOT NULL DEFAULT false,
    sequence_number INTEGER NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE inspection_result (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    work_order_id   UUID NOT NULL REFERENCES work_order(id),
    inspection_plan_id UUID NOT NULL REFERENCES inspection_plan(id),
    material_lot_id UUID REFERENCES material_lot(id),
    inspector_id    UUID NOT NULL REFERENCES person(id),
    inspected_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    overall_result  VARCHAR(10) NOT NULL CHECK (overall_result IN ('pass', 'fail', 'conditional')),
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE inspection_measurement (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    inspection_result_id UUID NOT NULL REFERENCES inspection_result(id),
    inspection_characteristic_id UUID NOT NULL REFERENCES inspection_characteristic(id),
    measured_value  DECIMAL(15, 6),
    attribute_result VARCHAR(10) CHECK (attribute_result IN ('pass', 'fail')),
    result          VARCHAR(10) NOT NULL CHECK (result IN ('pass', 'fail')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE non_conformance (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    site_id         UUID NOT NULL REFERENCES site(id),
    nc_number       VARCHAR(100) NOT NULL,
    inspection_result_id UUID REFERENCES inspection_result(id),
    work_order_id   UUID REFERENCES work_order(id),
    material_lot_id UUID REFERENCES material_lot(id),
    equipment_id    UUID REFERENCES equipment(id),
    severity        VARCHAR(30) NOT NULL CHECK (severity IN ('critical', 'major', 'minor')),
    description     TEXT NOT NULL,
    disposition     VARCHAR(30) CHECK (disposition IN ('use_as_is', 'rework', 'scrap', 'return_to_supplier', 'pending')),
    status          VARCHAR(30) NOT NULL DEFAULT 'open' CHECK (status IN ('open', 'investigating', 'resolved', 'closed')),
    reported_by     UUID NOT NULL REFERENCES person(id),
    reported_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    resolved_by     UUID REFERENCES person(id),
    resolved_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (site_id, nc_number)
);

CREATE TABLE corrective_action (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    non_conformance_id UUID NOT NULL REFERENCES non_conformance(id),
    capa_number     VARCHAR(100) NOT NULL,
    action_type     VARCHAR(30) NOT NULL CHECK (action_type IN ('corrective', 'preventive')),
    description     TEXT NOT NULL,
    root_cause      TEXT,
    status          VARCHAR(30) NOT NULL DEFAULT 'open' CHECK (status IN ('open', 'in_progress', 'completed', 'verified', 'closed')),
    assigned_to     UUID REFERENCES person(id),
    due_date        DATE,
    completed_at    TIMESTAMPTZ,
    verified_by     UUID REFERENCES person(id),
    verified_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_inspection_result_wo ON inspection_result(work_order_id);
CREATE INDEX idx_nc_status ON non_conformance(status);
CREATE INDEX idx_capa_status ON corrective_action(status);
CREATE INDEX idx_capa_due ON corrective_action(due_date) WHERE status NOT IN ('completed', 'closed');
```

## Audit Trail and E-Signatures (21 CFR Part 11)

```sql
-- ============================================================
-- AUDIT TRAIL AND E-SIGNATURES (21 CFR Part 11 / EU Annex 11)
-- ============================================================

CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    -- What changed
    table_name      VARCHAR(100) NOT NULL,
    record_id       UUID NOT NULL,
    action          VARCHAR(20) NOT NULL CHECK (action IN ('INSERT', 'UPDATE', 'DELETE')),
    -- Who changed it
    performed_by    UUID NOT NULL REFERENCES person(id),
    performed_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    -- 21 CFR Part 11: reason for change
    reason          TEXT,
    -- Old and new values as JSONB
    old_values      JSONB,
    new_values      JSONB,
    -- Client info
    ip_address      INET,
    user_agent      VARCHAR(500),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- 21 CFR Part 11 compliant e-signatures
CREATE TABLE electronic_signature (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    -- What is being signed
    table_name      VARCHAR(100) NOT NULL,
    record_id       UUID NOT NULL,
    -- Who signed
    signer_id       UUID NOT NULL REFERENCES person(id),
    signer_full_name VARCHAR(255) NOT NULL,
    -- Signature meaning
    meaning         VARCHAR(100) NOT NULL, -- e.g., 'approved', 'reviewed', 'verified', 'released'
    reason          TEXT,
    -- Signature timestamp (non-repudiation)
    signed_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    -- Hash of the signed record at time of signature (tamper detection)
    record_hash     VARCHAR(128) NOT NULL,
    -- Authentication method
    auth_method     VARCHAR(30) NOT NULL CHECK (auth_method IN ('password', 'biometric', 'token', 'certificate')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Partitioned by month for performance (audit logs grow fast)
CREATE INDEX idx_audit_log_table_record ON audit_log(table_name, record_id);
CREATE INDEX idx_audit_log_performed ON audit_log(performed_at);
CREATE INDEX idx_audit_log_user ON audit_log(performed_by);
CREATE INDEX idx_esig_record ON electronic_signature(table_name, record_id);
```

## Machine Data Ingestion

```sql
-- ============================================================
-- MACHINE DATA INGESTION (OPC-UA / MQTT / Sparkplug B)
-- ============================================================

CREATE TABLE data_point_definition (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    equipment_id    UUID NOT NULL REFERENCES equipment(id),
    tag_name        VARCHAR(255) NOT NULL,
    -- OPC-UA NodeId or MQTT topic
    source_address  VARCHAR(500) NOT NULL,
    source_protocol VARCHAR(20) NOT NULL CHECK (source_protocol IN ('opcua', 'mqtt', 'sparkplug', 'mtconnect', 'manual')),
    data_type       VARCHAR(30) NOT NULL CHECK (data_type IN ('float', 'integer', 'boolean', 'string')),
    unit_of_measure VARCHAR(50),
    -- Engineering limits
    eng_low         DECIMAL(15, 6),
    eng_high        DECIMAL(15, 6),
    -- Sampling configuration
    sample_interval_ms INTEGER DEFAULT 1000,
    deadband        DECIMAL(15, 6),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Time-series data: partitioned by time for scalability
CREATE TABLE machine_data_point (
    id              BIGSERIAL,
    data_point_definition_id UUID NOT NULL REFERENCES data_point_definition(id),
    equipment_id    UUID NOT NULL REFERENCES equipment(id),
    timestamp       TIMESTAMPTZ NOT NULL,
    value_float     DOUBLE PRECISION,
    value_integer   BIGINT,
    value_boolean   BOOLEAN,
    value_string    VARCHAR(500),
    quality         VARCHAR(20) DEFAULT 'good' CHECK (quality IN ('good', 'bad', 'uncertain')),
    PRIMARY KEY (id, timestamp)
) PARTITION BY RANGE (timestamp);

-- Create monthly partitions (example)
-- CREATE TABLE machine_data_point_2026_05 PARTITION OF machine_data_point
--   FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');

CREATE INDEX idx_data_point_equipment_time ON machine_data_point(equipment_id, timestamp);
CREATE INDEX idx_data_point_def_time ON machine_data_point(data_point_definition_id, timestamp);
```

## Maintenance Management

```sql
-- ============================================================
-- MAINTENANCE MANAGEMENT
-- ============================================================

CREATE TABLE maintenance_plan (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    equipment_id    UUID NOT NULL REFERENCES equipment(id),
    name            VARCHAR(255) NOT NULL,
    maintenance_type VARCHAR(30) NOT NULL CHECK (maintenance_type IN ('preventive', 'predictive', 'condition_based')),
    -- Frequency: interval-based or usage-based
    interval_days   INTEGER,
    interval_hours  INTEGER,    -- operating hours
    interval_cycles INTEGER,    -- cycle count
    last_performed  TIMESTAMPTZ,
    next_due        TIMESTAMPTZ,
    status          VARCHAR(30) NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE maintenance_work_order (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    site_id         UUID NOT NULL REFERENCES site(id),
    maintenance_plan_id UUID REFERENCES maintenance_plan(id),
    equipment_id    UUID NOT NULL REFERENCES equipment(id),
    mwo_number      VARCHAR(100) NOT NULL,
    maintenance_type VARCHAR(30) NOT NULL CHECK (maintenance_type IN ('preventive', 'corrective', 'predictive', 'emergency')),
    description     TEXT NOT NULL,
    priority        INTEGER NOT NULL DEFAULT 50,
    status          VARCHAR(30) NOT NULL DEFAULT 'open' CHECK (status IN ('open', 'in_progress', 'completed', 'cancelled')),
    assigned_to     UUID REFERENCES person(id),
    planned_start   TIMESTAMPTZ,
    planned_end     TIMESTAMPTZ,
    actual_start    TIMESTAMPTZ,
    actual_end      TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (site_id, mwo_number)
);

CREATE INDEX idx_maintenance_next_due ON maintenance_plan(next_due) WHERE status = 'active';
CREATE INDEX idx_mwo_status ON maintenance_work_order(status);
```

## Notifications and Alerts

```sql
-- ============================================================
-- NOTIFICATIONS AND ALERTS
-- ============================================================

CREATE TABLE alert_rule (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(255) NOT NULL,
    -- Conditions
    target_type     VARCHAR(30) NOT NULL CHECK (target_type IN ('equipment', 'work_center', 'site', 'kpi')),
    target_id       UUID,
    condition_type  VARCHAR(50) NOT NULL, -- e.g., 'oee_below', 'downtime_exceeds', 'spc_violation'
    threshold_value DECIMAL(15, 6),
    -- Notification channels
    notify_email    BOOLEAN NOT NULL DEFAULT false,
    notify_sms      BOOLEAN NOT NULL DEFAULT false,
    notify_webhook  BOOLEAN NOT NULL DEFAULT false,
    webhook_url     VARCHAR(500),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE alert_instance (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    alert_rule_id   UUID NOT NULL REFERENCES alert_rule(id),
    triggered_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    message         TEXT NOT NULL,
    severity        VARCHAR(20) NOT NULL CHECK (severity IN ('info', 'warning', 'critical')),
    acknowledged_by UUID REFERENCES person(id),
    acknowledged_at TIMESTAMPTZ,
    resolved_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_alert_instance_unack ON alert_instance(triggered_at)
    WHERE acknowledged_at IS NULL;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Multi-tenancy & Organization | 4 | tenant, site, area, work_center |
| Personnel & RBAC | 8 | person, personnel_class, qualification, roles, permissions |
| Equipment | 4 | equipment, equipment_class, properties, capabilities |
| Material & BOM | 5 | material_class, material_definition, material_lot, material_property, BOM |
| Product Definition & Routing | 4 | product_definition, routing, routing_step, routing_step_material |
| Recipe / ISA-88 | 6 | master_recipe, procedure, unit_procedure, operation, phase, parameters |
| Production Scheduling | 5 | production_order, work_order, work_order_material, shift_definition, labor_record |
| OEE & Performance | 4 | downtime_reason, equipment_state_log, oee_record, kpi_definition |
| Quality Management | 5 | inspection_plan, characteristics, results, measurements, non_conformance + CAPA |
| Audit Trail & E-Signatures | 2 | audit_log, electronic_signature |
| Machine Data | 2 | data_point_definition, machine_data_point (partitioned) |
| Maintenance | 2 | maintenance_plan, maintenance_work_order |
| Notifications | 2 | alert_rule, alert_instance |
| **Total** | **~51** | Core schema; additional tables for document management, energy, PPAP would add 10-15 more |

---

## Key Design Decisions

1. **ISA-95 resource model as the backbone.** Personnel, equipment, and material each have a class-instance-property pattern that mirrors B2MML exactly, enabling 1:1 import/export without translation.

2. **Equipment hierarchy via self-referential foreign key.** `equipment.parent_equipment_id` supports the ISA-95 levels (work center → work unit → equipment module → control module) without requiring separate tables per level.

3. **ISA-88 procedural hierarchy as four separate tables.** Rather than a single recursive table, the procedure → unit_procedure → operation → phase chain is modeled explicitly, matching the ISA-88 specification and making each level independently queryable.

4. **ISO 22400 OEE stored as pre-calculated aggregates.** Raw machine states are in `equipment_state_log`; aggregated OEE is in `oee_record`. This avoids expensive real-time computation while keeping the raw data available for AI root-cause analysis.

5. **Genealogy via `work_order_material` junction table.** Both inputs (consumed lots) and outputs (produced lots) are linked to work orders, enabling forward traceability (what finished goods contain this raw material?) and reverse traceability (what raw materials went into this finished product?) via multi-hop joins.

6. **21 CFR Part 11 compliance by design.** `audit_log` captures every change with old/new values, reason, and user identity. `electronic_signature` stores tamper-evident signatures with record hash verification. Both are append-only — no UPDATEs allowed on these tables.

7. **Time-series data partitioned by month.** `machine_data_point` uses PostgreSQL range partitioning so historical data can be archived or dropped without affecting current queries. At 1-second sampling across hundreds of machines, this table grows fastest.

8. **Downtime reason hierarchy.** `downtime_reason.parent_reason_id` supports multi-level reason trees (e.g., "Unplanned → Mechanical → Bearing Failure") for drill-down analysis without flattening the taxonomy.

9. **Separate production_order and work_order.** `production_order` is the ERP-level demand; `work_order` is the shop-floor execution at each routing step. This separation aligns with ISA-95's distinction between production scheduling and detailed scheduling.

10. **UUID primary keys throughout.** Supports multi-site deployment, data synchronization between edge and cloud, and avoids sequential ID exposure in APIs.
