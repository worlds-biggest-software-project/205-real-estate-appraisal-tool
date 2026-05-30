# Data Model Suggestion 2: Event-Sourced / Audit-First

> Project: Real Estate Appraisal Tool · Created: 2026-05-20

## Philosophy

This model treats every change to the appraisal system as an immutable event appended to an event store. The current state of any entity (property, appraisal, valuation) is derived by replaying its events. Materialised views (read models) are projected from the event stream for operational queries, but the event log is the single source of truth. This is a CQRS (Command Query Responsibility Segregation) architecture where writes go to the event store and reads come from projections.

The appraisal domain is an ideal fit for event sourcing because USPAP requires a complete audit trail of every data change, the CFPB AVM Rule demands documented data lineage, and temporal queries ("what was the appraised value on date X?" or "when did this comparable adjustment change?") are natural operations on an event stream. Rather than bolting audit logging onto a mutable schema, this model makes auditability the foundational architecture.

Financial services systems (banking ledgers, securities trading) and regulated healthcare platforms (EHR audit trails) commonly use event sourcing for the same regulatory reasons. The pattern also enables powerful analytics: ML pipelines can consume the event stream to detect valuation patterns, bias trends, and appraiser behaviour over time.

**Best for:** Teams building a compliance-critical platform where full auditability, temporal queries, and event-driven analytics are core requirements — especially if AI/ML pipelines will consume historical valuation data.

**Trade-offs:**
- Pro: Complete, immutable audit trail satisfying USPAP and CFPB requirements by design
- Pro: Temporal queries are trivial — replay to any point in time
- Pro: Event stream feeds ML pipelines, bias detection, and real-time analytics directly
- Pro: Easy to add new read models (projections) without changing the write side
- Con: Higher storage requirements (every change is stored, not just current state)
- Con: Read model projections add infrastructure complexity (projection workers, eventual consistency)
- Con: Schema evolution requires versioned event schemas and upcasters
- Con: Developers unfamiliar with event sourcing face a steeper learning curve
- Con: MISMO XML export requires assembling current state from projections rather than direct table reads

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| USPAP | Every data change is an immutable event — USPAP audit trail is a natural byproduct of the architecture, not a bolt-on |
| CFPB AVM Rule | AVM valuation events include model version, data sources, confidence intervals, and bias check results as event payload — full data lineage for every automated valuation |
| UAD 3.6 | Materialised `appraisal_report_view` projection contains UAD 3.6 fields assembled from latest events — same field structure, different storage mechanism |
| MISMO v3.6 | MISMO XML generated from report projection; event history enables generation of MISMO for any historical report version |
| FIRREA | Assignment events track independence confirmations; credential events track license status over time |
| ISO 19152 (LADM) | Property events follow LADM entity model; spatial and rights changes tracked as events |

---

## Event Store Core

```sql
-- Central event store — immutable append-only log
CREATE TABLE event_store (
    event_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_id UUID NOT NULL,                  -- Entity identifier (property ID, report ID, etc.)
    stream_type VARCHAR(50) NOT NULL,         -- PROPERTY, APPRAISAL_ORDER, APPRAISAL_REPORT, COMPARABLE, AVM, LEASE, INSPECTION
    event_type VARCHAR(100) NOT NULL,         -- e.g., PropertyCreated, ComparableAdjusted, ReportSubmitted
    event_version SMALLINT NOT NULL DEFAULT 1,-- Schema version for this event type
    sequence_number BIGINT NOT NULL,          -- Per-stream ordering
    payload JSONB NOT NULL,                   -- Event-specific data
    metadata JSONB NOT NULL DEFAULT '{}',     -- Correlation IDs, causation, user agent
    caused_by UUID REFERENCES app_user(id),   -- Who/what triggered this event
    occurred_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    
    UNIQUE(stream_id, sequence_number)
);

-- Ordered retrieval by stream (entity history)
CREATE INDEX idx_event_stream ON event_store(stream_id, sequence_number);
-- Type-based queries (all property events, all AVM events)
CREATE INDEX idx_event_type ON event_store(stream_type, event_type);
-- Time-based queries (replay from date, analytics windows)
CREATE INDEX idx_event_time ON event_store(occurred_at);
-- JSONB payload queries (search within events)
CREATE INDEX idx_event_payload ON event_store USING gin(payload);

-- Snapshots for performance (avoid replaying long streams)
CREATE TABLE event_snapshot (
    stream_id UUID NOT NULL,
    stream_type VARCHAR(50) NOT NULL,
    snapshot_version BIGINT NOT NULL,         -- sequence_number at snapshot time
    state JSONB NOT NULL,                     -- Full entity state at this point
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY(stream_id, snapshot_version)
);
```

### Event Type Taxonomy

```
-- Property Events
PropertyCreated           -- Initial property record
PropertyUpdated           -- Address, characteristics changed
PropertyMediaAdded        -- Photo, sketch, aerial added
PropertyConditionAssessed -- AI image recognition result

-- Order Events
OrderPlaced               -- New appraisal order from client
OrderAssigned             -- Appraiser assigned to order
OrderAccepted             -- Appraiser accepted assignment
OrderDeclined             -- Appraiser declined assignment
IndependenceConfirmed     -- FIRREA independence verified

-- Inspection Events
InspectionScheduled
InspectionStarted
InspectionCompleted
FloorPlanSketched
PhotoCaptured

-- Report Events
ReportDraftCreated
ReportFieldUpdated        -- Individual field change (old + new value)
ComparableSelected        -- Comp added to report
ComparableRemoved
ComparableAdjusted        -- Adjustment value changed
NarrativeGenerated        -- AI-generated narrative section
NarrativeEdited           -- Appraiser edited narrative
ValuationReconciled       -- Final value determined
ReportSignedByAppraiser
ReportSubmittedToUCDP
ReportAcceptedByGSE
ReportRejectedByGSE
RevisionRequested
RevisionCompleted

-- AVM Events
AVMValuationRequested
AVMValuationCompleted     -- Value, confidence, model metadata
AVMBiasCheckPassed
AVMBiasCheckFailed
AVMRandomSampleTested     -- CFPB compliance

-- Commercial Events
LeaseAbstracted           -- OCR/NLP extracted lease terms
LeaseVerified             -- Appraiser confirmed extracted data
DCFModelCreated
DCFAssumptionChanged
DCFSensitivityRun

-- Market Events
MarketTrendUpdated
ComparableSaleRecorded
MLSDataIngested
```

## Read Model Projections

```sql
-- ============================================================
-- PROJECTION: Current property state
-- Built from: PropertyCreated, PropertyUpdated events
-- ============================================================
CREATE TABLE property_projection (
    id UUID PRIMARY KEY,                      -- Same as stream_id
    street_address VARCHAR(500) NOT NULL,
    city VARCHAR(255) NOT NULL,
    state_province VARCHAR(100),
    postal_code VARCHAR(20),
    county VARCHAR(255),
    latitude NUMERIC(10, 7),
    longitude NUMERIC(10, 7),
    parcel_number VARCHAR(100),
    property_type VARCHAR(50),
    year_built SMALLINT,
    gross_living_area NUMERIC(10, 2),
    lot_size NUMERIC(12, 2),
    bedroom_count SMALLINT,
    bathroom_count NUMERIC(4, 1),
    condition_rating VARCHAR(10),
    quality_rating VARCHAR(10),
    
    -- AI-derived fields
    ai_condition_score NUMERIC(3, 2),
    property_embedding VECTOR(768),
    
    last_event_sequence BIGINT NOT NULL,
    projection_updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_prop_proj_location ON property_projection(latitude, longitude);
CREATE INDEX idx_prop_proj_postal ON property_projection(postal_code);
CREATE INDEX idx_prop_proj_embedding ON property_projection USING ivfflat (property_embedding vector_cosine_ops);

-- ============================================================
-- PROJECTION: Current appraisal report state (UAD 3.6 aligned)
-- Built from: ReportDraftCreated, ReportFieldUpdated, ComparableSelected,
--             ComparableAdjusted, ValuationReconciled, ReportSigned events
-- ============================================================
CREATE TABLE appraisal_report_projection (
    id UUID PRIMARY KEY,
    order_id UUID NOT NULL,
    appraiser_id UUID NOT NULL,
    status VARCHAR(30) NOT NULL,
    
    -- UAD 3.6 subject property fields
    effective_date DATE,
    property_rights_appraised VARCHAR(50),
    condition_rating VARCHAR(10),
    quality_rating VARCHAR(10),
    
    -- Site
    site_area NUMERIC(12, 2),
    topography VARCHAR(100),
    
    -- Improvements
    foundation_type VARCHAR(50),
    exterior_walls VARCHAR(100),
    roof_surface VARCHAR(100),
    heating_type VARCHAR(100),
    cooling_type VARCHAR(100),
    
    -- Neighbourhood
    neighbourhood_name VARCHAR(255),
    market_conditions VARCHAR(50),
    
    -- Valuation conclusions
    cost_approach_value NUMERIC(14, 2),
    sales_comparison_value NUMERIC(14, 2),
    income_approach_value NUMERIC(14, 2),
    final_reconciled_value NUMERIC(14, 2),
    
    -- Submission
    mismo_xml_url TEXT,
    ucdp_submission_id VARCHAR(100),
    ucdp_status VARCHAR(30),
    signed_at TIMESTAMPTZ,
    
    last_event_sequence BIGINT NOT NULL,
    projection_updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_report_proj_order ON appraisal_report_projection(order_id);
CREATE INDEX idx_report_proj_status ON appraisal_report_projection(status);
CREATE INDEX idx_report_proj_appraiser ON appraisal_report_projection(appraiser_id);

-- ============================================================
-- PROJECTION: Comparable adjustments grid
-- Built from: ComparableSelected, ComparableAdjusted, ComparableRemoved events
-- ============================================================
CREATE TABLE comparable_grid_projection (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    report_id UUID NOT NULL,
    comp_number SMALLINT NOT NULL,
    comparable_sale_id UUID NOT NULL,
    
    sale_price NUMERIC(14, 2),
    sale_date DATE,
    
    -- Adjustment columns (same as normalised model)
    location_adjustment NUMERIC(10, 2) DEFAULT 0,
    site_adjustment NUMERIC(10, 2) DEFAULT 0,
    quality_adjustment NUMERIC(10, 2) DEFAULT 0,
    condition_adjustment NUMERIC(10, 2) DEFAULT 0,
    gla_adjustment NUMERIC(10, 2) DEFAULT 0,
    room_count_adjustment NUMERIC(10, 2) DEFAULT 0,
    other_adjustment NUMERIC(10, 2) DEFAULT 0,
    net_adjustment NUMERIC(10, 2) DEFAULT 0,
    adjusted_sale_price NUMERIC(14, 2),
    
    selection_method VARCHAR(50),
    similarity_score NUMERIC(5, 4),
    
    last_event_sequence BIGINT NOT NULL,
    projection_updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(report_id, comp_number)
);

CREATE INDEX idx_comp_grid_report ON comparable_grid_projection(report_id);

-- ============================================================
-- PROJECTION: AVM results with compliance status
-- Built from: AVMValuationCompleted, AVMBiasCheckPassed/Failed events
-- ============================================================
CREATE TABLE avm_projection (
    id UUID PRIMARY KEY,
    property_id UUID NOT NULL,
    report_id UUID,
    
    model_name VARCHAR(100),
    model_version VARCHAR(50),
    estimated_value NUMERIC(14, 2),
    confidence_score NUMERIC(5, 4),
    confidence_interval_low NUMERIC(14, 2),
    confidence_interval_high NUMERIC(14, 2),
    
    bias_check_status VARCHAR(20),            -- PASSED, FAILED, PENDING
    random_sample_tested BOOLEAN DEFAULT false,
    data_sources_used TEXT[],
    top_factors JSONB,
    
    valued_at TIMESTAMPTZ,
    last_event_sequence BIGINT NOT NULL,
    projection_updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_avm_proj_property ON avm_projection(property_id);
CREATE INDEX idx_avm_proj_valued ON avm_projection(valued_at);
```

## Supporting Operational Tables

```sql
-- Users (not event-sourced — operational concern)
CREATE TABLE app_user (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) NOT NULL UNIQUE,
    full_name VARCHAR(255) NOT NULL,
    role VARCHAR(50) NOT NULL,
    organisation_id UUID,
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Organisations
CREATE TABLE organisation (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    org_type VARCHAR(50) NOT NULL,
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Projection tracking (which projections are up-to-date)
CREATE TABLE projection_checkpoint (
    projection_name VARCHAR(100) PRIMARY KEY,
    last_processed_event_id UUID REFERENCES event_store(event_id),
    last_processed_at TIMESTAMPTZ NOT NULL,
    lag_events BIGINT DEFAULT 0,
    status VARCHAR(20) NOT NULL DEFAULT 'RUNNING' -- RUNNING, PAUSED, REBUILDING, ERROR
);
```

## Temporal Query Examples

```sql
-- "What was the appraised value of property X on March 1, 2026?"
-- Replay report events up to that date
SELECT payload->>'final_reconciled_value' AS value_at_date
FROM event_store
WHERE stream_id = (
    SELECT id FROM appraisal_report_projection WHERE order_id = :order_id
)
AND stream_type = 'APPRAISAL_REPORT'
AND event_type = 'ValuationReconciled'
AND occurred_at <= '2026-03-01T23:59:59Z'
ORDER BY sequence_number DESC
LIMIT 1;

-- "Show me every adjustment change for Comp 2 on report X"
SELECT 
    occurred_at,
    caused_by,
    payload->>'field_name' AS adjustment_field,
    payload->>'old_value' AS old_value,
    payload->>'new_value' AS new_value,
    payload->>'reason' AS reason
FROM event_store
WHERE stream_id = :report_id
AND event_type = 'ComparableAdjusted'
AND payload->>'comp_number' = '2'
ORDER BY sequence_number;

-- "All AVM bias check failures in the last 90 days"
SELECT 
    stream_id AS avm_id,
    payload->>'demographic_category' AS category,
    payload->>'disparity_ratio' AS disparity,
    payload->>'geographic_scope' AS scope,
    occurred_at
FROM event_store
WHERE stream_type = 'AVM'
AND event_type = 'AVMBiasCheckFailed'
AND occurred_at >= now() - INTERVAL '90 days'
ORDER BY occurred_at DESC;

-- "Rebuild the state of report X as it existed at sequence 42"
SELECT * FROM event_store
WHERE stream_id = :report_id
AND sequence_number <= 42
ORDER BY sequence_number;
-- Application code replays these events to reconstruct state
```

## Event Payload Examples

```json
// PropertyCreated
{
    "street_address": "123 Oak Lane",
    "city": "Austin",
    "state_province": "TX",
    "postal_code": "78701",
    "parcel_number": "0123456789",
    "property_type": "SFR",
    "year_built": 2005,
    "gross_living_area": 2450.00,
    "bedroom_count": 4,
    "bathroom_count": 2.5
}

// ComparableAdjusted
{
    "comp_number": 2,
    "comparable_sale_id": "a1b2c3d4-...",
    "field_name": "gla_adjustment",
    "old_value": 0,
    "new_value": -12500,
    "reason": "Subject GLA 2450 sqft vs comp GLA 2700 sqft; $50/sqft adjustment"
}

// AVMValuationCompleted (CFPB compliant)
{
    "model_name": "ResidentialAVM",
    "model_version": "3.2.1",
    "estimated_value": 485000.00,
    "confidence_score": 0.87,
    "confidence_interval_low": 460000.00,
    "confidence_interval_high": 510000.00,
    "data_sources_used": ["MLS_ACTRIS", "TRAVIS_COUNTY_TAX", "SATELLITE_IMAGERY"],
    "top_factors": [
        {"factor": "GLA", "impact": 0.32},
        {"factor": "recent_sale_proximity", "impact": 0.28},
        {"factor": "lot_size", "impact": 0.15}
    ],
    "comparable_ids": ["comp-uuid-1", "comp-uuid-2", "comp-uuid-3"],
    "conflict_of_interest_check": true,
    "nondiscrimination_check_passed": true
}
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store | 2 | event_store, event_snapshot |
| Property Projections | 1 | property_projection |
| Report Projections | 2 | appraisal_report_projection, comparable_grid_projection |
| AVM Projections | 1 | avm_projection |
| Operational | 2 | app_user, organisation |
| Infrastructure | 1 | projection_checkpoint |
| **Total** | **9** | Plus additional projections as needed |

---

## Key Design Decisions

1. **Single `event_store` table rather than per-aggregate tables** — simpler infrastructure, single ordered log, and `stream_type` partitioning handles scale. At very high volume, the table can be partitioned by `occurred_at` for time-based retention.

2. **Events are immutable** — no UPDATE or DELETE on `event_store`. Corrections are modelled as new events (e.g., `ReportFieldUpdated` with old and new values). This is the fundamental guarantee behind USPAP audit compliance.

3. **Projections are disposable and rebuildable** — any projection can be dropped and rebuilt from the event store. This means the read models can evolve independently of the event schema.

4. **Snapshot table for long-lived streams** — properties with hundreds of events would be slow to replay. Periodic snapshots store the full state at a known sequence number, allowing replay from the snapshot forward.

5. **Event payload is JSONB, not typed columns** — each event type has a different structure. The payload schema is documented and validated at the application layer, not the database layer. This provides maximum flexibility for schema evolution.

6. **`projection_checkpoint` enables exactly-once processing** — each projection worker tracks which event it last processed, enabling crash recovery and monitoring of projection lag.

7. **User and organisation tables are NOT event-sourced** — these are operational concerns that do not benefit from temporal queries. Keeping them as standard CRUD tables reduces complexity.

8. **Event type taxonomy follows domain language** — event names like `ComparableAdjusted` and `ValuationReconciled` use the same terms appraisers use, making the event stream readable by domain experts.

9. **Metadata field captures correlation and causation** — every event can link to the command that caused it and the user session that initiated it, enabling end-to-end tracing for compliance investigations.

10. **Read models are optimised for specific query patterns** — the `comparable_grid_projection` is structured exactly like the URAR comp grid, enabling direct rendering without joins. Different consumers (MISMO export, UI, analytics) can have their own tailored projections.
