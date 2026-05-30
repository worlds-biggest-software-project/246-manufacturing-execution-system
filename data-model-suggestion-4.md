# Data Model Suggestion 4: Graph-Relational Hybrid

> Project: Manufacturing Execution System (MES) · Created: 2026-05-22

## Philosophy

This model uses a relational core for operational CRUD (work orders, equipment, materials, quality) combined with a property graph layer for relationship-heavy queries: product genealogy chains, equipment dependency graphs, multi-tier supply chain traceability, and impact analysis. The graph layer is implemented either as PostgreSQL tables with recursive CTEs (`graph_node` / `graph_edge` pattern) or as a dedicated graph database (Neo4j, Amazon Neptune) synced from the relational core.

Manufacturing traceability is fundamentally a graph problem. "Which finished goods contain material from lot X?" requires traversing an arbitrarily deep chain of material-to-product relationships. "If machine Y fails, which work orders and downstream products are affected?" requires traversing equipment dependency paths. "Show me all personnel certified to operate equipment class Z across all sites" requires a personnel-equipment-qualification graph. Relational models handle these with recursive CTEs, but graph-native queries (Cypher, Gremlin) express them more naturally and execute more efficiently on deep or wide traversals.

This approach is inspired by supply chain graph implementations at automotive OEMs and electronics manufacturers who use Neo4j or similar graph databases for multi-tier supplier traceability, conflict mineral tracking, and recall impact analysis. Research from Microsoft (Azure SQL Graph), Neo4j (supply chain use cases), and Springer (graph database for supply chain resilience) confirms the pattern's effectiveness for relationship-heavy manufacturing domains.

**Best for:** Manufacturers with deep multi-tier supply chains, automotive/aerospace environments requiring rapid recall impact analysis, and deployments where genealogy traversal performance is critical (e.g., "trace all products containing recalled material lot within 60 seconds").

**Trade-offs:**
- (+) Genealogy and traceability queries are natural graph traversals -- no recursive CTEs needed
- (+) Impact analysis ("which products are affected by this material recall?") executes in milliseconds on graph
- (+) Equipment dependency and personnel qualification graphs enable sophisticated resource planning
- (+) Graph visualization provides intuitive operational dashboards for traceability
- (+) Relational core preserves transactional integrity for operational data
- (-) Two data stores to manage (relational + graph) increases operational complexity
- (-) Graph sync lag introduces eventual consistency risk for real-time traceability queries
- (-) Fewer developers are familiar with Cypher/Gremlin than SQL
- (-) Graph databases have less mature tooling for backup, monitoring, and compliance auditing
- (-) Licensing costs for enterprise graph databases (Neo4j Enterprise) add to TCO

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ISA-95 / IEC 62264 | Relational tables implement ISA-95 operational models; graph edges represent ISA-95 relationships (equipment-to-area, material-to-work-order) |
| ISA-88 / IEC 61512 | Recipe hierarchy modeled as a directed acyclic graph: Recipe -> Procedure -> Unit Procedure -> Operation -> Phase |
| B2MML (MESA) | Relational tables export B2MML-compatible messages; graph layer provides pre-computed traversal results for B2MML genealogy reports |
| ISO 22400 | OEE calculation from relational time-state tables; graph layer links equipment to downstream product quality outcomes |
| 21 CFR Part 11 | Audit trail in relational tables; graph edges carry timestamps enabling temporal genealogy traversal |
| IATF 16949 | Forward and reverse traceability chains are native graph traversals; PPAP documentation pulls data from both relational and graph layers |
| OPC-UA / MQTT | Equipment connectivity configuration in relational tables; equipment-to-equipment data flow modeled as graph edges |

---

## Relational Core: Operational Tables

### Site & Equipment

```sql
CREATE TABLE site (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(200) NOT NULL,
    country_code    CHAR(2) NOT NULL,
    timezone        VARCHAR(50) NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE area (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    site_id         UUID NOT NULL REFERENCES site(id),
    name            VARCHAR(200) NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_area_site ON area(site_id);

CREATE TABLE work_center (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    area_id         UUID NOT NULL REFERENCES area(id),
    name            VARCHAR(200) NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_wc_area ON work_center(area_id);

CREATE TABLE equipment (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    work_center_id  UUID REFERENCES work_center(id),
    name            VARCHAR(200) NOT NULL,
    equipment_type  VARCHAR(100) NOT NULL,
    serial_number   VARCHAR(100),
    status          VARCHAR(30) NOT NULL DEFAULT 'idle'
                    CHECK (status IN ('idle','running','down','maintenance','setup')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_equip_wc ON equipment(work_center_id);
CREATE INDEX idx_equip_status ON equipment(status);
```

### Material & Product

```sql
CREATE TABLE material_definition (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    part_number     VARCHAR(100) UNIQUE NOT NULL,
    name            VARCHAR(200) NOT NULL,
    material_class  VARCHAR(50) NOT NULL,
    uom             VARCHAR(20) NOT NULL,
    is_lot_tracked  BOOLEAN NOT NULL DEFAULT false,
    is_serial_tracked BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_mat_def_part ON material_definition(part_number);

CREATE TABLE material_lot (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    material_definition_id UUID NOT NULL REFERENCES material_definition(id),
    lot_number      VARCHAR(100) NOT NULL,
    quantity         NUMERIC(14,4) NOT NULL,
    uom             VARCHAR(20) NOT NULL,
    status          VARCHAR(30) NOT NULL DEFAULT 'available'
                    CHECK (status IN ('available','reserved','in_use','consumed','quarantined','scrapped')),
    site_id         UUID NOT NULL REFERENCES site(id),
    supplier_name   VARCHAR(200),
    supplier_lot    VARCHAR(100),
    received_at     TIMESTAMPTZ,
    expires_at      TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_mat_lot_def ON material_lot(material_definition_id);
CREATE INDEX idx_mat_lot_number ON material_lot(lot_number);
CREATE INDEX idx_mat_lot_status ON material_lot(status);

CREATE TABLE product_definition (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    part_number     VARCHAR(100) UNIQUE NOT NULL,
    name            VARCHAR(200) NOT NULL,
    revision        VARCHAR(20) NOT NULL DEFAULT 'A',
    status          VARCHAR(30) NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE product_lot (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    product_definition_id UUID NOT NULL REFERENCES product_definition(id),
    work_order_id   UUID NOT NULL,
    lot_number      VARCHAR(100) NOT NULL,
    serial_number   VARCHAR(100),
    quantity         NUMERIC(14,4) NOT NULL,
    uom             VARCHAR(20) NOT NULL,
    status          VARCHAR(30) NOT NULL DEFAULT 'in_progress',
    completed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_prod_lot_def ON product_lot(product_definition_id);
CREATE INDEX idx_prod_lot_number ON product_lot(lot_number);
```

### Work Orders

```sql
CREATE TABLE work_order (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_number    VARCHAR(50) UNIQUE NOT NULL,
    product_definition_id UUID NOT NULL REFERENCES product_definition(id),
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
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_wo_product ON work_order(product_definition_id);
CREATE INDEX idx_wo_status ON work_order(status);
CREATE INDEX idx_wo_site ON work_order(site_id);

CREATE TABLE work_order_step (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    work_order_id   UUID NOT NULL REFERENCES work_order(id),
    step_number     INTEGER NOT NULL,
    name            VARCHAR(200) NOT NULL,
    equipment_id    UUID REFERENCES equipment(id),
    assigned_to     UUID,
    status          VARCHAR(30) NOT NULL DEFAULT 'pending',
    actual_start    TIMESTAMPTZ,
    actual_end      TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_wos_wo ON work_order_step(work_order_id);
CREATE INDEX idx_wos_equip ON work_order_step(equipment_id);
```

### OEE & Performance

```sql
CREATE TABLE equipment_state_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    equipment_id    UUID NOT NULL REFERENCES equipment(id),
    work_order_id   UUID REFERENCES work_order(id),
    state           VARCHAR(30) NOT NULL,
    started_at      TIMESTAMPTZ NOT NULL,
    ended_at        TIMESTAMPTZ,
    duration_seconds NUMERIC(12,2),
    downtime_reason_code VARCHAR(20),
    notes           TEXT,
    reported_by     UUID,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_esl_equip_time ON equipment_state_log(equipment_id, started_at);

CREATE TABLE production_count (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    equipment_id    UUID NOT NULL REFERENCES equipment(id),
    work_order_id   UUID REFERENCES work_order(id),
    period_start    TIMESTAMPTZ NOT NULL,
    period_end      TIMESTAMPTZ NOT NULL,
    total_count     INTEGER NOT NULL DEFAULT 0,
    good_count      INTEGER NOT NULL DEFAULT 0,
    reject_count    INTEGER NOT NULL DEFAULT 0,
    ideal_cycle_time_sec NUMERIC(10,4),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_pc_equip_time ON production_count(equipment_id, period_start);

CREATE TABLE downtime_reason (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    category        VARCHAR(50) NOT NULL,
    code            VARCHAR(20) UNIQUE NOT NULL,
    description     VARCHAR(200) NOT NULL,
    is_planned      BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### Quality

```sql
CREATE TABLE inspection_plan (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    product_definition_id UUID NOT NULL REFERENCES product_definition(id),
    name            VARCHAR(200) NOT NULL,
    inspection_type VARCHAR(30) NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_ip_product ON inspection_plan(product_definition_id);

CREATE TABLE inspection_characteristic (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    inspection_plan_id UUID NOT NULL REFERENCES inspection_plan(id),
    name            VARCHAR(200) NOT NULL,
    measurement_type VARCHAR(20) NOT NULL,
    uom             VARCHAR(20),
    nominal_value   NUMERIC(14,6),
    upper_spec_limit NUMERIC(14,6),
    lower_spec_limit NUMERIC(14,6),
    is_critical     BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_ic_plan ON inspection_characteristic(inspection_plan_id);

CREATE TABLE inspection_result (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    work_order_id   UUID NOT NULL REFERENCES work_order(id),
    inspection_plan_id UUID NOT NULL REFERENCES inspection_plan(id),
    material_lot_id UUID REFERENCES material_lot(id),
    equipment_id    UUID REFERENCES equipment(id),
    inspector_id    UUID NOT NULL,
    overall_result  VARCHAR(20) NOT NULL CHECK (overall_result IN ('pass','fail','conditional')),
    inspected_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_ir_wo ON inspection_result(work_order_id);
CREATE INDEX idx_ir_date ON inspection_result(inspected_at);

CREATE TABLE inspection_measurement (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    inspection_result_id UUID NOT NULL REFERENCES inspection_result(id),
    inspection_characteristic_id UUID NOT NULL REFERENCES inspection_characteristic(id),
    measured_value  NUMERIC(14,6),
    attribute_result VARCHAR(20),
    is_in_spec      BOOLEAN NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_im_result ON inspection_measurement(inspection_result_id);

CREATE TABLE non_conformance (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    ncr_number      VARCHAR(50) UNIQUE NOT NULL,
    work_order_id   UUID REFERENCES work_order(id),
    material_lot_id UUID REFERENCES material_lot(id),
    severity        VARCHAR(20) NOT NULL,
    status          VARCHAR(30) NOT NULL DEFAULT 'open',
    description     TEXT NOT NULL,
    root_cause      TEXT,
    disposition     VARCHAR(30),
    reported_by     UUID NOT NULL,
    reported_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_ncr_status ON non_conformance(status);
```

### Personnel & RBAC

```sql
CREATE TABLE person (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    employee_number VARCHAR(50) UNIQUE NOT NULL,
    first_name      VARCHAR(100) NOT NULL,
    last_name       VARCHAR(100) NOT NULL,
    email           VARCHAR(255),
    site_id         UUID REFERENCES site(id),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE role (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(100) UNIQUE NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE person_role (
    person_id       UUID NOT NULL REFERENCES person(id),
    role_id         UUID NOT NULL REFERENCES role(id),
    PRIMARY KEY (person_id, role_id)
);

CREATE TABLE person_qualification (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    person_id       UUID NOT NULL REFERENCES person(id),
    qualification_name VARCHAR(200) NOT NULL,
    issued_at       DATE NOT NULL,
    expires_at      DATE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_pq_person ON person_qualification(person_id);
```

### Audit Trail

```sql
CREATE TABLE audit_trail (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    table_name      VARCHAR(100) NOT NULL,
    record_id       UUID NOT NULL,
    action          VARCHAR(20) NOT NULL,
    field_name      VARCHAR(100),
    old_value       TEXT,
    new_value       TEXT,
    reason          TEXT,
    performed_by    UUID NOT NULL,
    performed_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_audit_record ON audit_trail(table_name, record_id);
CREATE INDEX idx_audit_date ON audit_trail(performed_at);

CREATE TABLE electronic_signature (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
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

## Graph Layer: Property Graph Tables

The graph layer can be implemented in PostgreSQL (using the tables below with recursive CTEs) or in a dedicated graph database (Neo4j, Amazon Neptune). The PostgreSQL approach is shown here for self-contained deployment.

```sql
-- Graph node: represents any entity that participates in relationships
CREATE TABLE graph_node (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    node_type       VARCHAR(50) NOT NULL,
    -- Node types:
    -- 'MaterialLot', 'ProductLot', 'WorkOrder', 'Equipment',
    -- 'Person', 'Site', 'WorkCenter', 'Supplier',
    -- 'MaterialDefinition', 'ProductDefinition', 'InspectionResult'
    entity_id       UUID NOT NULL,             -- FK to the relational table (not enforced for flexibility)
    label           VARCHAR(200) NOT NULL,     -- human-readable label for graph visualization
    properties      JSONB NOT NULL DEFAULT '{}',
    -- properties: denormalized key attributes for graph-only queries
    -- MaterialLot: {"lot_number": "LOT-2026-0058", "part_number": "RAW-STEEL-304", "status": "consumed"}
    -- Equipment: {"serial_number": "SN-12345", "type": "CNC", "status": "running"}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(node_type, entity_id)
);
CREATE INDEX idx_gn_type ON graph_node(node_type);
CREATE INDEX idx_gn_entity ON graph_node(entity_id);
CREATE INDEX idx_gn_props ON graph_node USING GIN (properties jsonb_path_ops);

-- Graph edge: directed relationship between two nodes
CREATE TABLE graph_edge (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    edge_type       VARCHAR(50) NOT NULL,
    -- Edge types:
    -- 'CONSUMED_BY'       -- MaterialLot -> ProductLot (genealogy)
    -- 'PRODUCED_BY'       -- ProductLot -> WorkOrder
    -- 'EXECUTED_ON'       -- WorkOrder -> Equipment
    -- 'ASSIGNED_TO'       -- WorkOrder -> Person
    -- 'LOCATED_IN'        -- Equipment -> WorkCenter
    -- 'PART_OF'           -- WorkCenter -> Area -> Site
    -- 'INSPECTED_BY'      -- ProductLot -> InspectionResult
    -- 'SUPPLIED_BY'       -- MaterialLot -> Supplier
    -- 'QUALIFIED_FOR'     -- Person -> Equipment (via qualification)
    -- 'DEPENDS_ON'        -- Equipment -> Equipment (physical dependency)
    -- 'FEEDS_INTO'        -- Equipment -> Equipment (production line flow)
    -- 'COMPONENT_OF'      -- MaterialDefinition -> ProductDefinition (BOM)
    source_node_id  UUID NOT NULL REFERENCES graph_node(id),
    target_node_id  UUID NOT NULL REFERENCES graph_node(id),
    properties      JSONB NOT NULL DEFAULT '{}',
    -- properties: edge-specific attributes
    -- CONSUMED_BY: {"quantity": 25.5, "uom": "KG", "consumed_at": "2026-05-23T10:30:00Z"}
    -- QUALIFIED_FOR: {"qualification": "CNC Operator Level 2", "expires": "2027-06-01"}
    -- FEEDS_INTO: {"sequence": 3, "transfer_method": "conveyor"}
    valid_from      TIMESTAMPTZ NOT NULL DEFAULT now(),
    valid_to        TIMESTAMPTZ,               -- NULL = currently active
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_ge_source ON graph_edge(source_node_id);
CREATE INDEX idx_ge_target ON graph_edge(target_node_id);
CREATE INDEX idx_ge_type ON graph_edge(edge_type);
CREATE INDEX idx_ge_valid ON graph_edge(valid_from, valid_to);
CREATE INDEX idx_ge_props ON graph_edge USING GIN (properties jsonb_path_ops);

-- For PostgreSQL ltree-based hierarchy (alternative to recursive CTEs for site/area/work center)
-- Requires: CREATE EXTENSION ltree;
CREATE TABLE location_hierarchy (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    entity_id       UUID NOT NULL,
    entity_type     VARCHAR(30) NOT NULL,      -- 'site', 'area', 'work_center'
    path            TEXT NOT NULL,             -- e.g., 'site1.area2.wc3' (ltree path)
    name            VARCHAR(200) NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_lh_path ON location_hierarchy(path);
-- With ltree extension:
-- CREATE INDEX idx_lh_ltree ON location_hierarchy USING GIST (path::ltree);
```

---

## Graph Sync Process

```sql
-- Sync checkpoint: tracks when graph was last synchronized from relational tables
CREATE TABLE graph_sync_checkpoint (
    table_name      VARCHAR(100) PRIMARY KEY,
    last_synced_id  UUID,
    last_synced_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    records_synced  BIGINT NOT NULL DEFAULT 0
);

-- Example sync: when a new material lot is created in the relational table,
-- create a corresponding graph node and edges:
--
-- 1. INSERT INTO graph_node (node_type, entity_id, label, properties)
--    VALUES ('MaterialLot', <lot_id>, 'LOT-2026-0058 (RAW-STEEL-304)',
--            '{"lot_number":"LOT-2026-0058","part_number":"RAW-STEEL-304","status":"available"}');
--
-- 2. INSERT INTO graph_edge (edge_type, source_node_id, target_node_id, properties)
--    VALUES ('SUPPLIED_BY', <lot_node_id>, <supplier_node_id>,
--            '{"supplier_lot":"SUP-LOT-001","received_at":"2026-05-20T08:00:00Z"}');
--
-- When a genealogy_link is created (material consumed into product):
-- 3. INSERT INTO graph_edge (edge_type, source_node_id, target_node_id, properties)
--    VALUES ('CONSUMED_BY', <material_lot_node_id>, <product_lot_node_id>,
--            '{"quantity":25.5,"uom":"KG","consumed_at":"2026-05-23T10:30:00Z"}');
```

---

## Graph Query Examples (PostgreSQL Recursive CTEs)

### Forward genealogy: "What materials went into product lot PROD-LOT-2026-0099?"

```sql
WITH RECURSIVE trace AS (
    -- Start from the product lot
    SELECT
        gn.id AS node_id,
        gn.node_type,
        gn.label,
        gn.properties,
        ge.edge_type,
        ge.properties AS edge_properties,
        1 AS depth,
        ARRAY[gn.id] AS path
    FROM graph_node gn
    JOIN graph_edge ge ON ge.target_node_id = gn.id
    WHERE gn.node_type = 'ProductLot'
      AND gn.properties->>'lot_number' = 'PROD-LOT-2026-0099'
      AND ge.edge_type = 'CONSUMED_BY'

    UNION ALL

    -- Traverse upstream: find what was consumed to make each input
    SELECT
        gn2.id,
        gn2.node_type,
        gn2.label,
        gn2.properties,
        ge2.edge_type,
        ge2.properties,
        t.depth + 1,
        t.path || gn2.id
    FROM trace t
    JOIN graph_edge ge2 ON ge2.target_node_id = (
        -- Find the product lot that this material lot became part of
        SELECT ge3.target_node_id
        FROM graph_edge ge3
        WHERE ge3.source_node_id = t.node_id
          AND ge3.edge_type = 'PRODUCED_BY'
        LIMIT 1
    )
    JOIN graph_node gn2 ON ge2.source_node_id = gn2.id
    WHERE ge2.edge_type = 'CONSUMED_BY'
      AND t.depth < 10
      AND NOT (gn2.id = ANY(t.path))  -- prevent cycles
)
SELECT node_type, label, properties, edge_properties, depth
FROM trace
ORDER BY depth;
```

### Reverse traceability: "Which finished products contain material from lot LOT-2026-0058?"

```sql
WITH RECURSIVE impact AS (
    -- Start from the material lot
    SELECT
        gn.id AS node_id,
        gn.node_type,
        gn.label,
        gn.properties,
        0 AS depth,
        ARRAY[gn.id] AS path
    FROM graph_node gn
    WHERE gn.node_type = 'MaterialLot'
      AND gn.properties->>'lot_number' = 'LOT-2026-0058'

    UNION ALL

    -- Traverse downstream: find all product lots that consumed this material
    SELECT
        gn2.id,
        gn2.node_type,
        gn2.label,
        gn2.properties,
        i.depth + 1,
        i.path || gn2.id
    FROM impact i
    JOIN graph_edge ge ON ge.source_node_id = i.node_id
    JOIN graph_node gn2 ON ge.target_node_id = gn2.id
    WHERE ge.edge_type = 'CONSUMED_BY'
      AND i.depth < 10
      AND NOT (gn2.id = ANY(i.path))
)
SELECT node_type, label, properties, depth
FROM impact
WHERE node_type = 'ProductLot'
ORDER BY depth;
```

### Equipment dependency chain: "If CNC-03 goes down, what work orders are affected?"

```sql
WITH RECURSIVE downstream AS (
    SELECT
        gn.id AS node_id,
        gn.label,
        gn.properties,
        0 AS depth
    FROM graph_node gn
    WHERE gn.node_type = 'Equipment'
      AND gn.properties->>'serial_number' = 'CNC-03'

    UNION ALL

    SELECT
        gn2.id,
        gn2.label,
        gn2.properties,
        d.depth + 1
    FROM downstream d
    JOIN graph_edge ge ON ge.source_node_id = d.node_id
    JOIN graph_node gn2 ON ge.target_node_id = gn2.id
    WHERE ge.edge_type = 'FEEDS_INTO'
      AND ge.valid_to IS NULL  -- currently active edges only
      AND d.depth < 20
)
SELECT
    ds.label AS equipment,
    wo.order_number,
    wo.status,
    wo.planned_start
FROM downstream ds
JOIN graph_edge ge ON ge.target_node_id = ds.node_id AND ge.edge_type = 'EXECUTED_ON'
JOIN graph_node wo_node ON ge.source_node_id = wo_node.id AND wo_node.node_type = 'WorkOrder'
JOIN work_order wo ON wo_node.entity_id = wo.id
WHERE wo.status IN ('released', 'dispatched', 'in_progress');
```

### Personnel qualification graph: "Who can operate 5-axis CNC machines?"

```sql
SELECT
    p.first_name || ' ' || p.last_name AS operator_name,
    p.employee_number,
    ge.properties->>'qualification' AS qualification,
    ge.properties->>'expires' AS expires,
    eq_node.label AS equipment
FROM graph_node p_node
JOIN graph_edge ge ON ge.source_node_id = p_node.id AND ge.edge_type = 'QUALIFIED_FOR'
JOIN graph_node eq_node ON ge.target_node_id = eq_node.id AND eq_node.node_type = 'Equipment'
JOIN person p ON p_node.entity_id = p.id
WHERE eq_node.properties->>'type' = 'CNC'
  AND eq_node.properties @> '{"axis_count": 5}'
  AND ge.valid_to IS NULL
  AND p.is_active = true
ORDER BY p.last_name;
```

---

## Neo4j Alternative (Cypher Queries)

If using Neo4j instead of PostgreSQL graph tables, the same queries become more natural:

### Forward genealogy in Cypher

```cypher
MATCH (product:ProductLot {lot_number: 'PROD-LOT-2026-0099'})
      <-[r:CONSUMED_BY]-(material:MaterialLot)
RETURN material.lot_number, material.part_number, r.quantity, r.uom, r.consumed_at
ORDER BY r.consumed_at
```

### Multi-tier reverse traceability in Cypher

```cypher
MATCH path = (material:MaterialLot {lot_number: 'LOT-2026-0058'})
              -[:CONSUMED_BY*1..10]->(product:ProductLot)
RETURN product.lot_number, product.status, length(path) AS depth
ORDER BY depth
```

### Equipment impact analysis in Cypher

```cypher
MATCH (failed:Equipment {serial_number: 'CNC-03'})
      -[:FEEDS_INTO*0..20]->(downstream:Equipment)
      <-[:EXECUTED_ON]-(wo:WorkOrder)
WHERE wo.status IN ['released', 'dispatched', 'in_progress']
RETURN downstream.name, wo.order_number, wo.status, wo.planned_start
```

### BOM explosion as graph traversal in Cypher

```cypher
MATCH path = (product:ProductDefinition {part_number: 'ASSY-4500'})
              <-[:COMPONENT_OF*1..10]-(component:MaterialDefinition)
RETURN component.part_number, component.name, length(path) AS bom_level
ORDER BY bom_level, component.part_number
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Site & Location | 3 | site, area, work_center |
| Equipment | 1 | equipment |
| Material & Product | 4 | material_definition, material_lot, product_definition, product_lot |
| Work Orders | 2 | work_order, work_order_step |
| OEE & Performance | 3 | equipment_state_log, production_count, downtime_reason |
| Quality | 5 | inspection_plan, inspection_characteristic, inspection_result, inspection_measurement, non_conformance |
| Personnel & RBAC | 4 | person, role, person_role, person_qualification |
| Audit & Compliance | 2 | audit_trail, electronic_signature |
| Graph Layer | 3 | graph_node, graph_edge, location_hierarchy |
| Graph Infrastructure | 1 | graph_sync_checkpoint |
| **Total** | **~28** | Relational core + graph layer for relationship queries |

---

## Key Design Decisions

1. **Dual-layer architecture: relational core + graph overlay.** Operational CRUD (create work orders, log inspections, record OEE) goes through the relational tables for transactional integrity. Relationship-heavy queries (genealogy, impact analysis, qualification graphs) go through the graph layer for natural traversal. The graph is synced from the relational core, not the other way around.

2. **Generic graph_node / graph_edge tables.** Rather than creating separate graph tables for each entity type, a single `graph_node` table with a `node_type` discriminator and a single `graph_edge` table with an `edge_type` discriminator provide a flexible, extensible graph. New node and edge types can be added without DDL changes.

3. **Temporal edges with `valid_from` / `valid_to`.** Graph edges carry temporal validity. When an equipment dependency changes (a machine is moved to a different line), the old edge gets a `valid_to` timestamp and a new edge is created. This enables point-in-time graph queries: "what was the production line layout on date X?"

4. **Denormalized properties on graph nodes.** Graph nodes carry key attributes (lot numbers, part numbers, statuses) in JSONB `properties` so that many graph queries can be answered without joining back to the relational tables. This trades storage for query performance on graph traversals.

5. **PostgreSQL recursive CTEs as the default graph engine.** For deployments that cannot run a separate graph database, PostgreSQL recursive CTEs provide graph traversal capability with no additional infrastructure. Performance is adequate for genealogy chains of 5-10 levels and equipment networks of 100-500 nodes. For larger graphs (10,000+ nodes, 20+ level traversals), Neo4j or Amazon Neptune is recommended.

6. **Neo4j as the scale-out option.** The same graph model (node types, edge types, properties) maps directly to Neo4j's labeled property graph model. A CDC (Change Data Capture) process syncs relational changes to Neo4j in near-real-time. Cypher queries are dramatically more readable and performant than recursive CTEs for complex multi-hop traversals.

7. **BOM as a graph.** Bill of Materials is modeled as `COMPONENT_OF` edges between `MaterialDefinition` and `ProductDefinition` nodes. Multi-level BOM explosion is a natural graph traversal rather than a recursive CTE on relational BOM tables. This is particularly valuable for complex assemblies with 10+ levels of sub-assemblies.

8. **Equipment production line flow modeled as graph.** `FEEDS_INTO` edges between equipment nodes represent the physical flow of materials through a production line. This enables impact analysis ("if this machine goes down, what downstream equipment and work orders are affected?") that would require complex multi-join queries in a purely relational model.

9. **Qualification graph connects personnel to equipment.** `QUALIFIED_FOR` edges between `Person` nodes and `Equipment` nodes (with qualification details and expiry dates in edge properties) enable queries like "who can operate this machine?" and "which machines can this person operate?" -- critical for shift planning and safety compliance.

10. **Graph sync is eventually consistent.** The graph layer may lag the relational core by seconds. For operational decisions that require absolute consistency (e.g., "is this lot quarantined?"), the application queries the relational tables directly. The graph layer is used for analytical and navigational queries where slight lag is acceptable.
