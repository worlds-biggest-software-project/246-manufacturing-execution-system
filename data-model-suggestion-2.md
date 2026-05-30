# Data Model Suggestion 2: Event-Sourced / Audit-First (CQRS)

> Project: Manufacturing Execution System (MES) · Created: 2026-05-22

## Philosophy

This model treats every state change on the shop floor as an immutable domain event appended to a central event store. The event store is the single source of truth. Current state is derived by replaying events into materialized read models (views/projections) using the CQRS (Command Query Responsibility Segregation) pattern. There is no UPDATE or DELETE on the event store -- only INSERT.

This approach is inspired by financial trading systems and regulated healthcare platforms where complete, tamper-proof audit trails are not optional but foundational. In manufacturing, Rockwell/Plex already records every shop-floor transaction as an immutable transaction record. This model takes that philosophy to its logical conclusion: the audit trail IS the data model, not a secondary table populated by triggers. Replaying events reconstructs the state of any work order, equipment, or material lot at any point in time -- enabling "what was true on Tuesday at 14:30?" queries that are impossible with mutable-state relational models.

This architecture is best for deployments where full temporal audit trails are mandatory (pharma 21 CFR Part 11, automotive IATF 16949), where AI/ML pipelines need rich event streams for root-cause analysis and predictive modeling, and where the ability to add new read models without modifying the write path is valuable.

**Best for:** Heavily regulated environments (pharma, medical devices, aerospace) where complete temporal audit trails are non-negotiable, and AI-native deployments that need rich event streams for ML training and real-time analytics.

**Trade-offs:**
- (+) Perfect audit trail by construction -- every state change is an immutable event with timestamp, actor, and payload
- (+) Full temporal querying: reconstruct any entity's state at any past moment by replaying events up to that timestamp
- (+) Event streams are ideal input for AI/ML pipelines (OEE root-cause analysis, predictive maintenance, yield prediction)
- (+) New read models can be added without changing the write path -- just create a new projection
- (+) Natural fit for 21 CFR Part 11 compliance: events are inherently append-only with user attribution
- (-) Higher storage requirements: events accumulate indefinitely (snapshots mitigate replay cost)
- (-) Eventual consistency between event store and read models requires careful handling
- (-) More complex to implement than direct CRUD -- developers must think in events, not tables
- (-) Debugging requires understanding event replay; current state is not directly queryable without projections
- (-) Schema evolution of event payloads requires versioning strategy (upcasting)

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ISA-95 / IEC 62264 | Event types map to ISA-95 object model changes: ProductionRequestCreated, MaterialConsumed, EquipmentStateChanged, etc. |
| ISA-88 / IEC 61512 | Batch execution events follow the ISA-88 procedural hierarchy: PhaseStarted, OperationCompleted, UnitProcedureAborted |
| B2MML (MESA) | Read model projections output B2MML-compatible XML/JSON for ERP integration |
| ISO 22400 | OEE projections materialize from equipment state-change events using ISO 22400 time-state definitions |
| 21 CFR Part 11 | Audit trail is inherent: every event is immutable with user_id, timestamp, and event payload; e-signatures are events themselves |
| IATF 16949 | Genealogy chains are reconstructed by replaying MaterialConsumed and ProductLotCreated events in sequence |
| IEC 62443 | Command authorization enforced at the command handler layer before events are emitted |
| OPC-UA / MQTT | Machine telemetry ingested as SensorReadingReceived events in the same event store |

---

## Core Event Store

```sql
-- The single source of truth: an append-only event log
CREATE TABLE event_store (
    event_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_type     VARCHAR(50) NOT NULL,      -- aggregate type: 'WorkOrder', 'Equipment', 'MaterialLot', etc.
    stream_id       UUID NOT NULL,             -- aggregate instance ID
    event_type      VARCHAR(100) NOT NULL,     -- e.g., 'WorkOrderCreated', 'EquipmentStateChanged'
    event_version   INTEGER NOT NULL,          -- schema version for this event type (for upcasting)
    sequence_number BIGINT NOT NULL,           -- monotonically increasing within a stream
    payload         JSONB NOT NULL,            -- event-specific data
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- metadata example:
    -- {
    --   "user_id": "uuid",
    --   "user_name": "Jane Operator",
    --   "ip_address": "10.0.1.50",
    --   "correlation_id": "uuid",
    --   "causation_id": "uuid",
    --   "site_id": "uuid",
    --   "auth_method": "mfa"
    -- }
    occurred_at     TIMESTAMPTZ NOT NULL,      -- when the event happened in the real world
    recorded_at     TIMESTAMPTZ NOT NULL DEFAULT now(),  -- when it was persisted
    UNIQUE (stream_id, sequence_number)
);

-- Primary query path: load all events for an aggregate
CREATE INDEX idx_event_stream ON event_store(stream_id, sequence_number);

-- Query by event type (for projections that consume specific event types)
CREATE INDEX idx_event_type ON event_store(event_type, recorded_at);

-- Query by time range (for AI/ML pipelines and reporting)
CREATE INDEX idx_event_recorded ON event_store(recorded_at);

-- Query by stream type + time (for type-specific projections)
CREATE INDEX idx_event_stream_type ON event_store(stream_type, recorded_at);

-- GIN index on metadata for site-scoped queries
CREATE INDEX idx_event_metadata ON event_store USING GIN (metadata jsonb_path_ops);

-- Snapshots: periodic state snapshots to avoid replaying entire history
CREATE TABLE event_snapshot (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_type     VARCHAR(50) NOT NULL,
    stream_id       UUID NOT NULL,
    last_sequence   BIGINT NOT NULL,           -- snapshot is valid up to this sequence
    state           JSONB NOT NULL,            -- serialized aggregate state
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_snapshot_stream ON event_snapshot(stream_id, last_sequence DESC);
```

### Event Type Catalogue

```sql
-- Reference table documenting all known event types and their payload schemas
CREATE TABLE event_type_registry (
    event_type      VARCHAR(100) PRIMARY KEY,
    stream_type     VARCHAR(50) NOT NULL,
    description     TEXT NOT NULL,
    payload_schema  JSONB NOT NULL,            -- JSON Schema defining the payload structure
    current_version INTEGER NOT NULL DEFAULT 1,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Example event type registrations:
-- INSERT INTO event_type_registry VALUES
-- ('WorkOrderCreated', 'WorkOrder', 'A new work order was created', '{...schema...}', 1, now()),
-- ('WorkOrderDispatched', 'WorkOrder', 'Work order dispatched to shop floor', '{...}', 1, now()),
-- ('WorkOrderStepStarted', 'WorkOrder', 'An individual step on a work order began execution', '{...}', 1, now()),
-- ('WorkOrderStepCompleted', 'WorkOrder', 'An individual step on a work order was completed', '{...}', 1, now()),
-- ('WorkOrderCompleted', 'WorkOrder', 'Work order finished; all steps done', '{...}', 1, now()),
-- ('MaterialConsumed', 'WorkOrder', 'Material lot consumed against a work order step', '{...}', 1, now()),
-- ('ProductLotProduced', 'WorkOrder', 'Finished goods lot created from a work order', '{...}', 1, now()),
-- ('EquipmentStateChanged', 'Equipment', 'Equipment transitioned to a new state', '{...}', 1, now()),
-- ('DowntimeRecorded', 'Equipment', 'Downtime event logged with reason code', '{...}', 1, now()),
-- ('SensorReadingReceived', 'Equipment', 'Real-time sensor/tag value from OPC-UA or MQTT', '{...}', 1, now()),
-- ('InspectionPerformed', 'QualityInspection', 'Quality inspection completed with measurements', '{...}', 1, now()),
-- ('NonConformanceRaised', 'QualityInspection', 'Non-conformance report created', '{...}', 1, now()),
-- ('CAPAOpened', 'CAPA', 'Corrective/preventive action initiated', '{...}', 1, now()),
-- ('ESignatureApplied', 'ESignature', 'Electronic signature applied to a record', '{...}', 1, now()),
-- ('RecipeApproved', 'Recipe', 'Master recipe approved for production use', '{...}', 1, now()),
-- ('MaintenanceOrderCreated', 'MaintenanceOrder', 'Maintenance work order created', '{...}', 1, now());
```

---

## Read Model Projections (Materialized Views)

These tables are derived from the event store. They can be rebuilt at any time by replaying events.

### Work Order Projection

```sql
-- Current state of work orders (rebuilt from WorkOrder stream events)
CREATE TABLE rm_work_order (
    id              UUID PRIMARY KEY,          -- same as stream_id
    order_number    VARCHAR(50) UNIQUE NOT NULL,
    product_definition_id UUID NOT NULL,
    product_name    VARCHAR(200),
    site_id         UUID NOT NULL,
    planned_quantity NUMERIC(14,4) NOT NULL,
    actual_quantity  NUMERIC(14,4) DEFAULT 0,
    uom             VARCHAR(20) NOT NULL,
    priority        INTEGER NOT NULL DEFAULT 5,
    status          VARCHAR(30) NOT NULL,
    planned_start   TIMESTAMPTZ,
    planned_end     TIMESTAMPTZ,
    actual_start    TIMESTAMPTZ,
    actual_end      TIMESTAMPTZ,
    erp_order_id    VARCHAR(50),
    last_event_seq  BIGINT NOT NULL,           -- tracks which events have been projected
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_rm_wo_status ON rm_work_order(status);
CREATE INDEX idx_rm_wo_site ON rm_work_order(site_id);
CREATE INDEX idx_rm_wo_planned ON rm_work_order(planned_start);

-- Work order steps projection
CREATE TABLE rm_work_order_step (
    id              UUID PRIMARY KEY,
    work_order_id   UUID NOT NULL REFERENCES rm_work_order(id),
    step_number     INTEGER NOT NULL,
    name            VARCHAR(200) NOT NULL,
    equipment_id    UUID,
    assigned_to     UUID,
    status          VARCHAR(30) NOT NULL DEFAULT 'pending',
    actual_start    TIMESTAMPTZ,
    actual_end      TIMESTAMPTZ,
    last_event_seq  BIGINT NOT NULL,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_rm_wo_step_wo ON rm_work_order_step(work_order_id);
```

### Equipment State Projection

```sql
-- Current equipment state (rebuilt from Equipment stream events)
CREATE TABLE rm_equipment (
    id              UUID PRIMARY KEY,
    name            VARCHAR(200) NOT NULL,
    work_unit_id    UUID,
    equipment_class VARCHAR(200),
    serial_number   VARCHAR(100),
    current_state   VARCHAR(30) NOT NULL DEFAULT 'idle',
    current_work_order_id UUID,
    state_since     TIMESTAMPTZ,
    last_event_seq  BIGINT NOT NULL,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_rm_eq_state ON rm_equipment(current_state);

-- Equipment time log (materialized from EquipmentStateChanged events)
CREATE TABLE rm_equipment_time_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    equipment_id    UUID NOT NULL,
    time_state      VARCHAR(30) NOT NULL,
    started_at      TIMESTAMPTZ NOT NULL,
    ended_at        TIMESTAMPTZ,
    duration_seconds NUMERIC(12,2),
    work_order_id   UUID,
    downtime_reason_code VARCHAR(20),
    downtime_notes  TEXT,
    reported_by     UUID,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_rm_eq_time_equip ON rm_equipment_time_log(equipment_id, started_at);
```

### OEE Projection (ISO 22400)

```sql
-- OEE hourly rollup (rebuilt from equipment state and production count events)
CREATE TABLE rm_oee_hourly (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    equipment_id    UUID NOT NULL,
    site_id         UUID NOT NULL,
    hour_start      TIMESTAMPTZ NOT NULL,
    planned_time_sec    NUMERIC(12,2) NOT NULL DEFAULT 0,
    run_time_sec        NUMERIC(12,2) NOT NULL DEFAULT 0,
    availability_pct    NUMERIC(6,4),
    total_count         INTEGER NOT NULL DEFAULT 0,
    good_count          INTEGER NOT NULL DEFAULT 0,
    ideal_cycle_time_sec NUMERIC(10,4),
    performance_pct     NUMERIC(6,4),
    quality_pct         NUMERIC(6,4),
    oee_pct             NUMERIC(6,4),
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_rm_oee_equip_hour ON rm_oee_hourly(equipment_id, hour_start);
CREATE INDEX idx_rm_oee_site_hour ON rm_oee_hourly(site_id, hour_start);

-- OEE daily rollup
CREATE TABLE rm_oee_daily (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    equipment_id    UUID NOT NULL,
    site_id         UUID NOT NULL,
    summary_date    DATE NOT NULL,
    shift_name      VARCHAR(100),
    planned_time_sec    NUMERIC(12,2) NOT NULL DEFAULT 0,
    run_time_sec        NUMERIC(12,2) NOT NULL DEFAULT 0,
    availability_pct    NUMERIC(6,4),
    total_count         INTEGER NOT NULL DEFAULT 0,
    good_count          INTEGER NOT NULL DEFAULT 0,
    performance_pct     NUMERIC(6,4),
    quality_pct         NUMERIC(6,4),
    oee_pct             NUMERIC(6,4),
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_rm_oee_daily_equip ON rm_oee_daily(equipment_id, summary_date);
CREATE INDEX idx_rm_oee_daily_site ON rm_oee_daily(site_id, summary_date);
```

### Material & Genealogy Projection

```sql
-- Current material lot state
CREATE TABLE rm_material_lot (
    id              UUID PRIMARY KEY,
    part_number     VARCHAR(100) NOT NULL,
    material_name   VARCHAR(200) NOT NULL,
    lot_number      VARCHAR(100) NOT NULL,
    quantity_on_hand NUMERIC(14,4) NOT NULL,
    uom             VARCHAR(20) NOT NULL,
    status          VARCHAR(30) NOT NULL,
    site_id         UUID NOT NULL,
    storage_location VARCHAR(100),
    last_event_seq  BIGINT NOT NULL,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_rm_mat_lot_number ON rm_material_lot(lot_number);
CREATE INDEX idx_rm_mat_lot_status ON rm_material_lot(status);

-- Product lot (finished goods) state
CREATE TABLE rm_product_lot (
    id              UUID PRIMARY KEY,
    product_part_number VARCHAR(100) NOT NULL,
    lot_number      VARCHAR(100) NOT NULL,
    serial_number   VARCHAR(100),
    work_order_id   UUID,
    quantity         NUMERIC(14,4) NOT NULL,
    uom             VARCHAR(20) NOT NULL,
    status          VARCHAR(30) NOT NULL,
    completed_at    TIMESTAMPTZ,
    last_event_seq  BIGINT NOT NULL,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_rm_prod_lot_number ON rm_product_lot(lot_number);

-- Genealogy projection: maps material inputs to product outputs
CREATE TABLE rm_genealogy (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    product_lot_id  UUID NOT NULL,
    product_lot_number VARCHAR(100) NOT NULL,
    material_lot_id UUID NOT NULL,
    material_lot_number VARCHAR(100) NOT NULL,
    material_part_number VARCHAR(100) NOT NULL,
    quantity_consumed NUMERIC(14,4) NOT NULL,
    consumed_at     TIMESTAMPTZ NOT NULL,
    work_order_id   UUID,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_rm_gen_product ON rm_genealogy(product_lot_id);
CREATE INDEX idx_rm_gen_material ON rm_genealogy(material_lot_id);
```

### Quality Projection

```sql
-- Current quality inspection results
CREATE TABLE rm_inspection_result (
    id              UUID PRIMARY KEY,
    work_order_id   UUID NOT NULL,
    inspection_plan_name VARCHAR(200),
    material_lot_id UUID,
    equipment_id    UUID,
    inspector_name  VARCHAR(200),
    overall_result  VARCHAR(20) NOT NULL,
    inspected_at    TIMESTAMPTZ NOT NULL,
    measurement_count INTEGER NOT NULL DEFAULT 0,
    out_of_spec_count INTEGER NOT NULL DEFAULT 0,
    last_event_seq  BIGINT NOT NULL,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_rm_insp_wo ON rm_inspection_result(work_order_id);
CREATE INDEX idx_rm_insp_date ON rm_inspection_result(inspected_at);

-- Non-conformance tracker
CREATE TABLE rm_non_conformance (
    id              UUID PRIMARY KEY,
    ncr_number      VARCHAR(50) UNIQUE NOT NULL,
    work_order_id   UUID,
    material_lot_id UUID,
    severity        VARCHAR(20) NOT NULL,
    status          VARCHAR(30) NOT NULL,
    description     TEXT NOT NULL,
    disposition     VARCHAR(30),
    reported_by_name VARCHAR(200),
    reported_at     TIMESTAMPTZ NOT NULL,
    last_event_seq  BIGINT NOT NULL,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_rm_ncr_status ON rm_non_conformance(status);
```

### Sensor Telemetry Projection

```sql
-- Time-series projection of sensor data (partitioned by time for performance)
CREATE TABLE rm_sensor_reading (
    equipment_id    UUID NOT NULL,
    parameter_name  VARCHAR(100) NOT NULL,
    reading_value   NUMERIC(14,4) NOT NULL,
    uom             VARCHAR(20),
    is_alarm        BOOLEAN NOT NULL DEFAULT false,
    occurred_at     TIMESTAMPTZ NOT NULL,
    recorded_at     TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (occurred_at);

CREATE INDEX idx_rm_sensor_equip_time ON rm_sensor_reading(equipment_id, occurred_at);
CREATE INDEX idx_rm_sensor_param ON rm_sensor_reading(parameter_name, occurred_at);

-- Create monthly partitions (example)
-- CREATE TABLE rm_sensor_reading_2026_05 PARTITION OF rm_sensor_reading
--     FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');
```

---

## Reference Data Tables

Reference data is mutable (not event-sourced) since it changes infrequently and does not require temporal querying.

```sql
-- Site hierarchy (configuration data, not event-sourced)
CREATE TABLE ref_site (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    enterprise_name VARCHAR(200) NOT NULL,
    name            VARCHAR(200) NOT NULL,
    country_code    CHAR(2) NOT NULL,
    timezone        VARCHAR(50) NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Equipment class reference
CREATE TABLE ref_equipment_class (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(200) NOT NULL,
    description     TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Material definitions reference
CREATE TABLE ref_material_definition (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    part_number     VARCHAR(100) UNIQUE NOT NULL,
    name            VARCHAR(200) NOT NULL,
    uom             VARCHAR(20) NOT NULL,
    is_lot_tracked  BOOLEAN NOT NULL DEFAULT false,
    is_serial_tracked BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Product definitions reference
CREATE TABLE ref_product_definition (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    part_number     VARCHAR(100) UNIQUE NOT NULL,
    name            VARCHAR(200) NOT NULL,
    revision        VARCHAR(20) NOT NULL DEFAULT 'A',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Downtime reason codes reference
CREATE TABLE ref_downtime_reason (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    category        VARCHAR(50) NOT NULL,
    code            VARCHAR(20) UNIQUE NOT NULL,
    description     VARCHAR(200) NOT NULL,
    is_planned      BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Shift definitions reference
CREATE TABLE ref_shift (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    site_id         UUID NOT NULL REFERENCES ref_site(id),
    name            VARCHAR(100) NOT NULL,
    start_time      TIME NOT NULL,
    end_time        TIME NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Personnel reference
CREATE TABLE ref_person (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    employee_number VARCHAR(50) UNIQUE NOT NULL,
    first_name      VARCHAR(100) NOT NULL,
    last_name       VARCHAR(100) NOT NULL,
    email           VARCHAR(255),
    site_id         UUID REFERENCES ref_site(id),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Roles and permissions (RBAC)
CREATE TABLE ref_role (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(100) UNIQUE NOT NULL,
    permissions     JSONB NOT NULL DEFAULT '[]',
    -- permissions example: ["work_order:create", "work_order:dispatch", "inspection:perform"]
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE ref_person_role (
    person_id       UUID NOT NULL REFERENCES ref_person(id),
    role_id         UUID NOT NULL REFERENCES ref_role(id),
    site_id         UUID REFERENCES ref_site(id),
    PRIMARY KEY (person_id, role_id)
);
```

---

## Example Event Payloads

### WorkOrderCreated

```json
{
  "order_number": "WO-2026-00142",
  "product_definition_id": "550e8400-e29b-41d4-a716-446655440010",
  "product_part_number": "ASSY-4500",
  "bom_id": "550e8400-e29b-41d4-a716-446655440020",
  "routing_id": "550e8400-e29b-41d4-a716-446655440030",
  "planned_quantity": 500,
  "uom": "EA",
  "priority": 3,
  "planned_start": "2026-05-23T06:00:00Z",
  "planned_end": "2026-05-23T18:00:00Z",
  "erp_order_id": "SAP-100042",
  "site_id": "550e8400-e29b-41d4-a716-446655440001"
}
```

### EquipmentStateChanged

```json
{
  "equipment_id": "550e8400-e29b-41d4-a716-446655440050",
  "previous_state": "running",
  "new_state": "down",
  "work_order_id": "550e8400-e29b-41d4-a716-446655440100",
  "downtime_reason_code": "MECH-003",
  "downtime_category": "mechanical",
  "notes": "Spindle bearing overheated; maintenance called"
}
```

### MaterialConsumed

```json
{
  "work_order_id": "550e8400-e29b-41d4-a716-446655440100",
  "work_order_step_id": "550e8400-e29b-41d4-a716-446655440101",
  "material_lot_id": "550e8400-e29b-41d4-a716-446655440200",
  "material_part_number": "RAW-STEEL-304",
  "lot_number": "LOT-2026-0058",
  "quantity_consumed": 25.5,
  "uom": "KG"
}
```

### InspectionPerformed

```json
{
  "work_order_id": "550e8400-e29b-41d4-a716-446655440100",
  "inspection_plan_id": "550e8400-e29b-41d4-a716-446655440300",
  "material_lot_id": "550e8400-e29b-41d4-a716-446655440200",
  "equipment_id": "550e8400-e29b-41d4-a716-446655440050",
  "overall_result": "pass",
  "measurements": [
    {
      "characteristic": "Wall Thickness",
      "measured_value": 2.51,
      "nominal": 2.50,
      "usl": 2.55,
      "lsl": 2.45,
      "uom": "mm",
      "in_spec": true
    },
    {
      "characteristic": "Surface Roughness",
      "measured_value": 0.8,
      "nominal": 0.6,
      "usl": 1.0,
      "lsl": 0.2,
      "uom": "Ra",
      "in_spec": true
    }
  ]
}
```

---

## Example Queries

### Reconstruct work order state at a specific point in time

```sql
-- Get all events for work order WO-2026-00142 up to a specific timestamp
SELECT event_type, payload, occurred_at, metadata->>'user_name' AS actor
FROM event_store
WHERE stream_type = 'WorkOrder'
  AND stream_id = '550e8400-e29b-41d4-a716-446655440100'
  AND occurred_at <= '2026-05-23T14:30:00Z'
ORDER BY sequence_number ASC;
```

### Calculate OEE from raw events (without using projections)

```sql
-- Build OEE for a specific equipment on a specific day from events
WITH state_changes AS (
    SELECT
        payload->>'new_state' AS state,
        occurred_at,
        LEAD(occurred_at) OVER (ORDER BY sequence_number) AS next_change
    FROM event_store
    WHERE stream_type = 'Equipment'
      AND stream_id = :equipment_id
      AND event_type = 'EquipmentStateChanged'
      AND occurred_at >= '2026-05-23T00:00:00Z'
      AND occurred_at < '2026-05-24T00:00:00Z'
),
time_summary AS (
    SELECT
        state,
        SUM(EXTRACT(EPOCH FROM (COALESCE(next_change, '2026-05-24T00:00:00Z') - occurred_at))) AS seconds
    FROM state_changes
    GROUP BY state
)
SELECT
    COALESCE(SUM(seconds) FILTER (WHERE state IN ('running','setup')), 0) AS planned_time_sec,
    COALESCE(SUM(seconds) FILTER (WHERE state = 'running'), 0) AS run_time_sec,
    ROUND(
        COALESCE(SUM(seconds) FILTER (WHERE state = 'running'), 0) /
        NULLIF(COALESCE(SUM(seconds) FILTER (WHERE state IN ('running','setup','down')), 0), 0)
    , 4) AS availability
FROM time_summary;
```

### Full genealogy trace from events

```sql
-- Trace all material inputs for a product lot (from events)
SELECT
    payload->>'product_lot_number' AS product,
    payload->>'material_lot_number' AS material_input,
    payload->>'material_part_number' AS part,
    (payload->>'quantity_consumed')::NUMERIC AS qty,
    occurred_at
FROM event_store
WHERE stream_type = 'WorkOrder'
  AND event_type = 'MaterialConsumed'
  AND stream_id = (
      SELECT stream_id FROM event_store
      WHERE event_type = 'ProductLotProduced'
        AND payload->>'lot_number' = 'PROD-LOT-2026-0099'
      LIMIT 1
  )
ORDER BY occurred_at;
```

---

## Projection Rebuild Process

```sql
-- Projection checkpoint: tracks which events each projection has processed
CREATE TABLE projection_checkpoint (
    projection_name VARCHAR(100) PRIMARY KEY,
    last_event_id   UUID NOT NULL,
    last_recorded_at TIMESTAMPTZ NOT NULL,
    events_processed BIGINT NOT NULL DEFAULT 0,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- To rebuild a projection:
-- 1. TRUNCATE the read model table
-- 2. DELETE from projection_checkpoint WHERE projection_name = '<name>'
-- 3. Replay all events of the relevant type, applying the projection logic
-- 4. Update the checkpoint
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store (write side) | 3 | event_store, event_snapshot, event_type_registry |
| Projection Infrastructure | 1 | projection_checkpoint |
| Work Order Projections | 2 | rm_work_order, rm_work_order_step |
| Equipment Projections | 2 | rm_equipment, rm_equipment_time_log |
| OEE Projections | 2 | rm_oee_hourly, rm_oee_daily |
| Material & Genealogy Projections | 3 | rm_material_lot, rm_product_lot, rm_genealogy |
| Quality Projections | 2 | rm_inspection_result, rm_non_conformance |
| Sensor Telemetry Projection | 1 | rm_sensor_reading (partitioned) |
| Reference Data | 10 | Sites, equipment classes, materials, products, reasons, shifts, personnel, roles |
| **Total** | **~26** | Plus event_store contains ALL state (the true data model is the event catalogue) |

---

## Key Design Decisions

1. **Single event store table as source of truth.** All domain state changes -- work orders, equipment states, material movements, quality inspections, e-signatures -- are events in one table. The `stream_type` and `stream_id` columns partition events by aggregate. This dramatically simplifies the write path and guarantees a complete, ordered history.

2. **JSONB payloads with schema registry.** Event payloads are JSONB for flexibility; the `event_type_registry` table documents the expected schema for each event type. The `event_version` column enables payload schema evolution via upcasting (transforming old event versions to current format during replay).

3. **Read model tables prefixed with `rm_`.** Clear naming convention distinguishes projections (derived, rebuildable) from the event store (source of truth) and reference data (configuration). Any `rm_` table can be dropped and rebuilt without data loss.

4. **Snapshots for replay performance.** For aggregates with thousands of events (e.g., a high-volume equipment stream), periodic snapshots avoid replaying the entire history. The snapshot stores the serialized aggregate state at a known sequence number; replay starts from the snapshot.

5. **21 CFR Part 11 compliance by construction.** Every event includes the acting user, timestamp, and authentication method in metadata. E-signatures are events themselves (`ESignatureApplied`), inherently immutable. There is no mechanism to alter past events -- the audit trail IS the data.

6. **Time-partitioned sensor telemetry.** The `rm_sensor_reading` projection is partitioned by month to handle high-volume machine data (potentially millions of rows per day across a plant). Old partitions can be archived to cold storage without affecting current queries.

7. **Eventual consistency accepted.** Read models may lag the event store by milliseconds to seconds. For operator dashboards this is acceptable. For critical command validation (e.g., "has this lot been quarantined?"), commands read directly from the event store to ensure consistency.

8. **Event-driven AI/ML pipeline integration.** The event store doubles as the training data source for AI models. OEE root-cause analysis, predictive maintenance, and yield prediction models consume events directly via CDC (Change Data Capture) or polling on `recorded_at`, without needing a separate ETL pipeline.

9. **Reference data is NOT event-sourced.** Site hierarchy, material definitions, personnel, and roles are simple mutable tables. They change infrequently and do not benefit from temporal history. This avoids over-engineering the parts of the system that are essentially configuration.

10. **Correlation and causation IDs in metadata.** Every event carries a `correlation_id` (the original request that triggered the chain) and `causation_id` (the specific event that caused this one). This enables full causal chain tracing for debugging and compliance investigations.
