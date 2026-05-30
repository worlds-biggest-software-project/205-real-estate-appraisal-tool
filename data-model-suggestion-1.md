# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Real Estate Appraisal Tool · Created: 2026-05-20

## Philosophy

This model follows traditional third-normal-form (3NF) relational design where every concept in the appraisal domain gets its own table with explicit foreign key relationships. The schema mirrors the structure of MISMO v3.6 and UAD 3.6 data elements, with dedicated tables for each form section, comparable attribute, and valuation approach. Reference data (property types, condition ratings, quality ratings) is stored in lookup tables aligned with UAD standardised values.

This approach prioritises data integrity and regulatory compliance. Every field that USPAP, UAD 3.6, or MISMO defines has a dedicated, typed, indexed column — making it straightforward to validate submissions, generate MISMO XML exports, and prove compliance during audits. The schema is verbose but unambiguous: there are no "magic" JSON blobs to parse, and every relationship is enforceable at the database level.

Real-world systems using this pattern include legacy appraisal platforms like TOTAL and ACI, which map their internal data models closely to URAR form fields. Enterprise mortgage systems (Ellie Mae/ICE Mortgage Technology) also use heavily normalised schemas aligned to MISMO.

**Best for:** Teams building a compliance-first platform where regulatory alignment, data integrity, and auditability are paramount.

**Trade-offs:**
- Pro: Every field is typed, indexed, and validatable — no schema ambiguity
- Pro: Direct mapping to MISMO XML simplifies GSE submission
- Pro: Standard SQL queries for all reporting and analytics
- Pro: Foreign key constraints prevent orphaned or inconsistent data
- Con: High table count (~60+) increases migration complexity
- Con: Schema changes require DDL migrations for every new field
- Con: Multi-jurisdiction variations (RICS vs USPAP) require additional columns or tables
- Con: Complex joins for cross-entity queries (e.g., "all comps for all appraisals in a market")

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| UAD 3.6 | Each UAD field maps to a dedicated column in the `appraisal_report` and related tables; lookup tables enforce UAD standardised values for condition, quality, and location ratings |
| MISMO v3.6 | Table and column naming follows MISMO Reference Model entities (PROPERTY, APPRAISAL, COMPARABLE_SALE, etc.) for direct XML serialisation |
| USPAP | Audit trail tables (`appraisal_revision`, `data_source_log`) capture every data change with timestamps and appraiser identity |
| FIRREA | `appraiser` table stores licensing credentials; `appraisal_assignment` enforces independence from lending functions |
| CFPB AVM Rule | `avm_result` table stores model version, confidence interval, and bias audit results for every automated valuation |
| ISO 19152 (LADM) | `property` table structure aligns with LADM Party-RRR-SpatialUnit model for property rights and spatial representation |
| RESO Web API | `mls_listing` table mirrors RESO Data Dictionary field names for direct MLS import compatibility |
| ISO 3166 | `jurisdiction` lookup table uses ISO 3166-1 (country) and ISO 3166-2 (subdivision) codes |

---

## Core Property & Location Tables

```sql
-- Reference: Jurisdictions (ISO 3166)
CREATE TABLE jurisdiction (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    country_code CHAR(2) NOT NULL,           -- ISO 3166-1 alpha-2
    subdivision_code VARCHAR(6),              -- ISO 3166-2 (e.g., US-CA)
    name VARCHAR(255) NOT NULL,
    currency_code CHAR(3) NOT NULL DEFAULT 'USD', -- ISO 4217
    valuation_standard VARCHAR(50) NOT NULL DEFAULT 'USPAP', -- USPAP, RICS, IVS
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_jurisdiction_country ON jurisdiction(country_code);
CREATE INDEX idx_jurisdiction_subdivision ON jurisdiction(subdivision_code);

-- Properties
CREATE TABLE property (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    jurisdiction_id UUID NOT NULL REFERENCES jurisdiction(id),
    street_address VARCHAR(500) NOT NULL,
    city VARCHAR(255) NOT NULL,
    state_province VARCHAR(100),
    postal_code VARCHAR(20),
    county VARCHAR(255),
    latitude NUMERIC(10, 7),
    longitude NUMERIC(10, 7),
    parcel_number VARCHAR(100),              -- APN / tax parcel ID
    legal_description TEXT,
    property_type_id UUID NOT NULL REFERENCES property_type(id),
    year_built SMALLINT,
    gross_living_area NUMERIC(10, 2),        -- sq ft or sq m
    lot_size NUMERIC(12, 2),
    lot_size_unit VARCHAR(10) DEFAULT 'sqft', -- sqft, acres, sqm
    bedroom_count SMALLINT,
    bathroom_count NUMERIC(4, 1),
    stories NUMERIC(3, 1),
    garage_spaces SMALLINT,
    pool BOOLEAN DEFAULT false,
    zoning VARCHAR(50),
    flood_zone VARCHAR(20),
    census_tract VARCHAR(20),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_property_location ON property(latitude, longitude);
CREATE INDEX idx_property_jurisdiction ON property(jurisdiction_id);
CREATE INDEX idx_property_parcel ON property(parcel_number);
CREATE INDEX idx_property_postal ON property(postal_code);
CREATE INDEX idx_property_type ON property(property_type_id);

-- Property type lookup (UAD standardised)
CREATE TABLE property_type (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code VARCHAR(20) NOT NULL UNIQUE,        -- e.g., SFR, CONDO, 2-4UNIT, COMMERCIAL
    name VARCHAR(100) NOT NULL,
    category VARCHAR(50) NOT NULL,           -- RESIDENTIAL, COMMERCIAL
    uad_code VARCHAR(10),                    -- UAD 3.6 property type code
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Property images and media
CREATE TABLE property_media (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    property_id UUID NOT NULL REFERENCES property(id),
    media_type VARCHAR(20) NOT NULL,         -- PHOTO, SKETCH, AERIAL, DRONE
    category VARCHAR(50) NOT NULL,           -- FRONT, REAR, STREET, INTERIOR, FLOORPLAN
    file_url TEXT NOT NULL,
    file_size_bytes BIGINT,
    mime_type VARCHAR(100),
    caption TEXT,
    captured_at TIMESTAMPTZ,
    captured_by UUID REFERENCES app_user(id),
    ai_condition_score NUMERIC(3, 2),        -- 0.00–1.00 from image recognition
    ai_condition_notes TEXT,
    sort_order SMALLINT DEFAULT 0,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_property_media_property ON property_media(property_id);
```

## User & Organisation Tables

```sql
-- Users (appraisers, AMC staff, lenders, admins)
CREATE TABLE app_user (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) NOT NULL UNIQUE,
    full_name VARCHAR(255) NOT NULL,
    phone VARCHAR(50),
    role VARCHAR(50) NOT NULL,               -- APPRAISER, AMC_MANAGER, LENDER, ADMIN
    organisation_id UUID REFERENCES organisation(id),
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_user_org ON app_user(organisation_id);
CREATE INDEX idx_user_role ON app_user(role);

-- Appraiser credentials (FIRREA compliance)
CREATE TABLE appraiser_credential (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES app_user(id),
    jurisdiction_id UUID NOT NULL REFERENCES jurisdiction(id),
    license_number VARCHAR(100) NOT NULL,
    license_type VARCHAR(50) NOT NULL,       -- CERTIFIED_GENERAL, CERTIFIED_RESIDENTIAL, LICENSED, TRAINEE
    license_status VARCHAR(20) NOT NULL,     -- ACTIVE, EXPIRED, SUSPENDED, REVOKED
    issue_date DATE NOT NULL,
    expiration_date DATE NOT NULL,
    uspap_update_date DATE,                  -- Last 7-hour USPAP update course
    designations TEXT[],                     -- MAI, SRA, AI-GRS, etc.
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(user_id, jurisdiction_id, license_number)
);

CREATE INDEX idx_credential_user ON appraiser_credential(user_id);
CREATE INDEX idx_credential_expiry ON appraiser_credential(expiration_date);

-- Organisations (AMCs, appraisal firms, lenders)
CREATE TABLE organisation (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    org_type VARCHAR(50) NOT NULL,           -- AMC, APPRAISAL_FIRM, LENDER, INVESTOR
    jurisdiction_id UUID REFERENCES jurisdiction(id),
    address TEXT,
    phone VARCHAR(50),
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Appraisal Order & Assignment Tables

```sql
-- Appraisal orders (from lender/AMC)
CREATE TABLE appraisal_order (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_number VARCHAR(100) NOT NULL UNIQUE,
    property_id UUID NOT NULL REFERENCES property(id),
    client_id UUID NOT NULL REFERENCES organisation(id),   -- Lender or AMC
    intended_use VARCHAR(100) NOT NULL,      -- MORTGAGE_ORIGINATION, REFINANCE, ESTATE, TAX_APPEAL
    report_type VARCHAR(50) NOT NULL,        -- URAR, SUMMARY, RESTRICTED, DESKTOP, HYBRID
    engagement_letter_url TEXT,
    ordered_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    due_date DATE,
    rush BOOLEAN DEFAULT false,
    status VARCHAR(30) NOT NULL DEFAULT 'PENDING',  -- PENDING, ASSIGNED, IN_PROGRESS, SUBMITTED, REVISION_REQUESTED, COMPLETED, CANCELLED
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_order_property ON appraisal_order(property_id);
CREATE INDEX idx_order_client ON appraisal_order(client_id);
CREATE INDEX idx_order_status ON appraisal_order(status);
CREATE INDEX idx_order_due ON appraisal_order(due_date);

-- Appraiser assignments (FIRREA independence)
CREATE TABLE appraisal_assignment (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id UUID NOT NULL REFERENCES appraisal_order(id),
    appraiser_id UUID NOT NULL REFERENCES app_user(id),
    assigned_by UUID NOT NULL REFERENCES app_user(id),
    assigned_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    accepted_at TIMESTAMPTZ,
    declined_at TIMESTAMPTZ,
    decline_reason TEXT,
    fee_amount NUMERIC(10, 2),
    fee_currency CHAR(3) DEFAULT 'USD',
    independence_confirmed BOOLEAN NOT NULL DEFAULT false, -- FIRREA
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_assignment_order ON appraisal_assignment(order_id);
CREATE INDEX idx_assignment_appraiser ON appraisal_assignment(appraiser_id);
```

## Appraisal Report Tables (UAD 3.6 Aligned)

```sql
-- Appraisal reports (core URAR data)
CREATE TABLE appraisal_report (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id UUID NOT NULL REFERENCES appraisal_order(id),
    appraiser_id UUID NOT NULL REFERENCES app_user(id),
    report_version SMALLINT NOT NULL DEFAULT 1,
    status VARCHAR(30) NOT NULL DEFAULT 'DRAFT', -- DRAFT, REVIEW, SUBMITTED, ACCEPTED, REVISION
    
    -- Subject property (UAD 3.6 fields)
    effective_date DATE NOT NULL,
    property_rights_appraised VARCHAR(50),   -- FEE_SIMPLE, LEASEHOLD, LEASED_FEE
    
    -- Site section
    site_area NUMERIC(12, 2),
    site_area_unit VARCHAR(10),
    site_shape VARCHAR(50),
    topography VARCHAR(100),
    utilities TEXT,
    off_site_improvements TEXT,
    
    -- Improvements section
    foundation_type VARCHAR(50),
    exterior_walls VARCHAR(100),
    roof_surface VARCHAR(100),
    heating_type VARCHAR(100),
    cooling_type VARCHAR(100),
    
    -- UAD condition and quality ratings
    condition_rating VARCHAR(10) NOT NULL,    -- C1-C6 (UAD standardised)
    quality_rating VARCHAR(10) NOT NULL,      -- Q1-Q6 (UAD standardised)
    
    -- Neighbourhood section
    neighbourhood_name VARCHAR(255),
    neighbourhood_boundaries TEXT,
    market_conditions VARCHAR(50),           -- STABLE, DECLINING, INCREASING
    
    -- Valuation conclusions
    cost_approach_value NUMERIC(14, 2),
    sales_comparison_value NUMERIC(14, 2),
    income_approach_value NUMERIC(14, 2),
    final_reconciled_value NUMERIC(14, 2) NOT NULL,
    
    -- Appraiser certification
    appraiser_signature_url TEXT,
    signed_at TIMESTAMPTZ,
    supervisory_appraiser_id UUID REFERENCES app_user(id),
    
    -- MISMO XML
    mismo_xml_url TEXT,                      -- Generated MISMO v3.6 XML file
    ucdp_submission_id VARCHAR(100),         -- UCDP tracking ID
    ucdp_submitted_at TIMESTAMPTZ,
    ucdp_status VARCHAR(30),                -- SUBMITTED, ACCEPTED, REJECTED
    
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_report_order ON appraisal_report(order_id);
CREATE INDEX idx_report_appraiser ON appraisal_report(appraiser_id);
CREATE INDEX idx_report_status ON appraisal_report(status);
CREATE INDEX idx_report_effective_date ON appraisal_report(effective_date);

-- Appraisal report revisions (USPAP audit trail)
CREATE TABLE appraisal_revision (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    report_id UUID NOT NULL REFERENCES appraisal_report(id),
    revision_number SMALLINT NOT NULL,
    changed_by UUID NOT NULL REFERENCES app_user(id),
    field_name VARCHAR(100) NOT NULL,
    old_value TEXT,
    new_value TEXT,
    reason TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_revision_report ON appraisal_revision(report_id);
CREATE INDEX idx_revision_field ON appraisal_revision(field_name);
```

## Comparable Sales Tables

```sql
-- Comparable sales (used across appraisals)
CREATE TABLE comparable_sale (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    property_id UUID REFERENCES property(id), -- Link to property if in our system
    data_source VARCHAR(50) NOT NULL,         -- MLS, PUBLIC_RECORD, APPRAISER, COSTAR
    data_source_id VARCHAR(100),              -- MLS number, deed book/page, etc.
    
    -- Sale details
    sale_price NUMERIC(14, 2) NOT NULL,
    sale_date DATE NOT NULL,
    sale_type VARCHAR(50),                    -- ARM_LENGTH, REO, SHORT_SALE, FORECLOSURE
    financing_type VARCHAR(50),               -- CONVENTIONAL, FHA, VA, CASH, SELLER_FINANCED
    concessions_amount NUMERIC(10, 2),
    days_on_market SMALLINT,
    
    -- Property characteristics at time of sale
    gross_living_area NUMERIC(10, 2),
    lot_size NUMERIC(12, 2),
    year_built SMALLINT,
    bedroom_count SMALLINT,
    bathroom_count NUMERIC(4, 1),
    condition_rating VARCHAR(10),             -- C1-C6
    quality_rating VARCHAR(10),               -- Q1-Q6
    
    -- Location
    latitude NUMERIC(10, 7),
    longitude NUMERIC(10, 7),
    
    -- Embedding for AI similarity search
    property_embedding VECTOR(768),           -- pgvector for similarity search
    
    verified BOOLEAN DEFAULT false,
    verified_by UUID REFERENCES app_user(id),
    verified_at TIMESTAMPTZ,
    
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_comp_sale_date ON comparable_sale(sale_date);
CREATE INDEX idx_comp_sale_price ON comparable_sale(sale_price);
CREATE INDEX idx_comp_location ON comparable_sale(latitude, longitude);
CREATE INDEX idx_comp_data_source ON comparable_sale(data_source, data_source_id);
CREATE INDEX idx_comp_embedding ON comparable_sale USING ivfflat (property_embedding vector_cosine_ops);

-- Comparable selection for a specific appraisal
CREATE TABLE appraisal_comparable (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    report_id UUID NOT NULL REFERENCES appraisal_report(id),
    comparable_sale_id UUID NOT NULL REFERENCES comparable_sale(id),
    comp_number SMALLINT NOT NULL,            -- Comp 1, 2, 3, etc.
    selection_method VARCHAR(50),             -- MANUAL, AI_SUGGESTED, AI_CONFIRMED
    similarity_score NUMERIC(5, 4),           -- AI embedding similarity
    
    -- Adjustments (UAD standardised)
    location_adjustment NUMERIC(10, 2) DEFAULT 0,
    site_adjustment NUMERIC(10, 2) DEFAULT 0,
    design_adjustment NUMERIC(10, 2) DEFAULT 0,
    quality_adjustment NUMERIC(10, 2) DEFAULT 0,
    age_adjustment NUMERIC(10, 2) DEFAULT 0,
    condition_adjustment NUMERIC(10, 2) DEFAULT 0,
    gla_adjustment NUMERIC(10, 2) DEFAULT 0,
    room_count_adjustment NUMERIC(10, 2) DEFAULT 0,
    basement_adjustment NUMERIC(10, 2) DEFAULT 0,
    functional_utility_adjustment NUMERIC(10, 2) DEFAULT 0,
    heating_cooling_adjustment NUMERIC(10, 2) DEFAULT 0,
    garage_adjustment NUMERIC(10, 2) DEFAULT 0,
    porch_patio_adjustment NUMERIC(10, 2) DEFAULT 0,
    other_adjustment NUMERIC(10, 2) DEFAULT 0,
    net_adjustment NUMERIC(10, 2) GENERATED ALWAYS AS (
        location_adjustment + site_adjustment + design_adjustment +
        quality_adjustment + age_adjustment + condition_adjustment +
        gla_adjustment + room_count_adjustment + basement_adjustment +
        functional_utility_adjustment + heating_cooling_adjustment +
        garage_adjustment + porch_patio_adjustment + other_adjustment
    ) STORED,
    adjusted_sale_price NUMERIC(14, 2),
    
    adjustment_justification TEXT,
    
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(report_id, comp_number)
);

CREATE INDEX idx_appraisal_comp_report ON appraisal_comparable(report_id);
CREATE INDEX idx_appraisal_comp_sale ON appraisal_comparable(comparable_sale_id);
```

## AVM & Bias Detection Tables

```sql
-- AVM results (CFPB AVM Rule compliance)
CREATE TABLE avm_result (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    property_id UUID NOT NULL REFERENCES property(id),
    report_id UUID REFERENCES appraisal_report(id),  -- Optional link to appraisal
    
    model_name VARCHAR(100) NOT NULL,
    model_version VARCHAR(50) NOT NULL,
    
    estimated_value NUMERIC(14, 2) NOT NULL,
    confidence_score NUMERIC(5, 4) NOT NULL, -- 0.0000–1.0000
    confidence_interval_low NUMERIC(14, 2),
    confidence_interval_high NUMERIC(14, 2),
    forecast_standard_deviation NUMERIC(14, 2),
    
    -- CFPB compliance fields
    data_sources_used TEXT[] NOT NULL,
    data_quality_flags TEXT[],
    conflict_of_interest_check BOOLEAN NOT NULL DEFAULT true,
    random_sample_tested BOOLEAN DEFAULT false,
    nondiscrimination_check_passed BOOLEAN,
    
    -- Explainability
    top_factors JSONB,  -- e.g., [{"factor": "GLA", "impact": 0.35}, ...]
    comparable_ids UUID[],                    -- Comps used by the model
    
    valued_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_avm_property ON avm_result(property_id);
CREATE INDEX idx_avm_report ON avm_result(report_id);
CREATE INDEX idx_avm_valued_at ON avm_result(valued_at);

-- Bias audit results
CREATE TABLE bias_audit (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    audit_type VARCHAR(50) NOT NULL,          -- MODEL_WIDE, SINGLE_VALUATION, MARKET_SEGMENT
    avm_result_id UUID REFERENCES avm_result(id),
    
    demographic_category VARCHAR(100),        -- RACE, ETHNICITY, INCOME_LEVEL
    geographic_scope TEXT,
    
    disparity_ratio NUMERIC(6, 4),
    statistical_significance NUMERIC(5, 4),
    pass_fail VARCHAR(10) NOT NULL,           -- PASS, FAIL, REVIEW
    findings TEXT,
    remediation_notes TEXT,
    
    audited_by UUID REFERENCES app_user(id),
    audited_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_bias_audit_avm ON bias_audit(avm_result_id);
CREATE INDEX idx_bias_audit_type ON bias_audit(audit_type);
```

## Commercial Appraisal Tables

```sql
-- Commercial properties (extends base property)
CREATE TABLE commercial_property (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    property_id UUID NOT NULL UNIQUE REFERENCES property(id),
    property_subtype VARCHAR(50),             -- OFFICE, RETAIL, INDUSTRIAL, MULTIFAMILY, MIXED_USE
    total_rentable_area NUMERIC(12, 2),
    number_of_units SMALLINT,
    number_of_floors SMALLINT,
    parking_spaces SMALLINT,
    occupancy_rate NUMERIC(5, 2),
    noi NUMERIC(14, 2),                       -- Net operating income
    cap_rate NUMERIC(5, 4),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Lease records for income approach
CREATE TABLE lease (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    property_id UUID NOT NULL REFERENCES property(id),
    tenant_name VARCHAR(255) NOT NULL,
    unit_identifier VARCHAR(100),
    lease_type VARCHAR(50) NOT NULL,          -- GROSS, NET, NNN, MODIFIED_GROSS, PERCENTAGE
    lease_start DATE NOT NULL,
    lease_end DATE NOT NULL,
    base_rent_annual NUMERIC(12, 2) NOT NULL,
    rent_per_sqft NUMERIC(8, 2),
    rentable_sqft NUMERIC(10, 2),
    escalation_type VARCHAR(50),              -- FIXED, CPI, STEP, MARKET
    escalation_rate NUMERIC(5, 4),
    cam_charges NUMERIC(10, 2),
    insurance_charges NUMERIC(10, 2),
    tax_charges NUMERIC(10, 2),
    tenant_improvement_allowance NUMERIC(10, 2),
    free_rent_months SMALLINT DEFAULT 0,
    renewal_options TEXT,
    source VARCHAR(50),                       -- MANUAL, OCR_EXTRACTED, API
    source_document_url TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_lease_property ON lease(property_id);
CREATE INDEX idx_lease_dates ON lease(lease_start, lease_end);

-- DCF valuation model
CREATE TABLE dcf_valuation (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    report_id UUID NOT NULL REFERENCES appraisal_report(id),
    projection_years SMALLINT NOT NULL DEFAULT 10,
    discount_rate NUMERIC(6, 4) NOT NULL,
    terminal_cap_rate NUMERIC(6, 4) NOT NULL,
    vacancy_rate NUMERIC(5, 4),
    expense_growth_rate NUMERIC(5, 4),
    rent_growth_rate NUMERIC(5, 4),
    
    projected_noi_year1 NUMERIC(14, 2),
    present_value_cash_flows NUMERIC(14, 2),
    present_value_reversion NUMERIC(14, 2),
    total_dcf_value NUMERIC(14, 2) NOT NULL,
    
    sensitivity_discount_low NUMERIC(14, 2),
    sensitivity_discount_high NUMERIC(14, 2),
    sensitivity_cap_low NUMERIC(14, 2),
    sensitivity_cap_high NUMERIC(14, 2),
    
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_dcf_report ON dcf_valuation(report_id);
```

## Inspection & Field Data Tables

```sql
-- Field inspections
CREATE TABLE inspection (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id UUID NOT NULL REFERENCES appraisal_order(id),
    appraiser_id UUID NOT NULL REFERENCES app_user(id),
    inspection_type VARCHAR(50) NOT NULL,     -- FULL_INTERIOR, EXTERIOR_ONLY, DESKTOP, DRIVE_BY
    scheduled_at TIMESTAMPTZ,
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    notes TEXT,
    weather_conditions VARCHAR(100),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_inspection_order ON inspection(order_id);
CREATE INDEX idx_inspection_appraiser ON inspection(appraiser_id);

-- Floor plan sketches
CREATE TABLE floor_plan (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    property_id UUID NOT NULL REFERENCES property(id),
    inspection_id UUID REFERENCES inspection(id),
    floor_number SMALLINT NOT NULL DEFAULT 1,
    sketch_data JSONB NOT NULL,               -- Vector drawing data
    calculated_area NUMERIC(10, 2),
    area_unit VARCHAR(10) DEFAULT 'sqft',
    image_url TEXT,                            -- Rendered image
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_floor_plan_property ON floor_plan(property_id);
```

## MLS & Data Integration Tables

```sql
-- MLS listing data (RESO Data Dictionary aligned)
CREATE TABLE mls_listing (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    mls_id VARCHAR(50) NOT NULL,
    mls_source VARCHAR(100) NOT NULL,         -- Which MLS system
    property_id UUID REFERENCES property(id),
    
    -- RESO standard fields
    listing_key VARCHAR(100),
    list_price NUMERIC(14, 2),
    close_price NUMERIC(14, 2),
    listing_contract_date DATE,
    close_date DATE,
    days_on_market SMALLINT,
    standard_status VARCHAR(30),              -- Active, Pending, Closed, Withdrawn
    property_type VARCHAR(50),
    
    raw_data JSONB,                           -- Full RESO response for additional fields
    
    fetched_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(mls_id, mls_source)
);

CREATE INDEX idx_mls_property ON mls_listing(property_id);
CREATE INDEX idx_mls_source ON mls_listing(mls_source, mls_id);
CREATE INDEX idx_mls_close_date ON mls_listing(close_date);

-- Data source provenance log (USPAP compliance)
CREATE TABLE data_source_log (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    report_id UUID REFERENCES appraisal_report(id),
    source_type VARCHAR(50) NOT NULL,         -- MLS, PUBLIC_RECORD, APPRAISER_OBSERVATION, AVM, API
    source_name VARCHAR(255) NOT NULL,
    source_reference VARCHAR(255),
    data_retrieved_at TIMESTAMPTZ NOT NULL,
    fields_used TEXT[],
    reliability_rating VARCHAR(20),           -- HIGH, MEDIUM, LOW
    notes TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_data_source_report ON data_source_log(report_id);
```

## Market Data Tables

```sql
-- Market area definitions
CREATE TABLE market_area (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    area_type VARCHAR(50) NOT NULL,           -- MSA, COUNTY, ZIP, NEIGHBOURHOOD, SUBMARKET
    parent_area_id UUID REFERENCES market_area(id),
    boundary_geojson JSONB,
    jurisdiction_id UUID REFERENCES jurisdiction(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_market_area_type ON market_area(area_type);
CREATE INDEX idx_market_area_parent ON market_area(parent_area_id);

-- Market trend snapshots
CREATE TABLE market_trend (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    market_area_id UUID NOT NULL REFERENCES market_area(id),
    property_type_id UUID REFERENCES property_type(id),
    period_start DATE NOT NULL,
    period_end DATE NOT NULL,
    
    median_sale_price NUMERIC(14, 2),
    average_sale_price NUMERIC(14, 2),
    sale_count INTEGER,
    median_days_on_market SMALLINT,
    price_per_sqft NUMERIC(8, 2),
    inventory_months NUMERIC(4, 1),
    
    -- Commercial metrics
    avg_cap_rate NUMERIC(5, 4),
    avg_vacancy_rate NUMERIC(5, 4),
    avg_rent_per_sqft NUMERIC(8, 2),
    
    source VARCHAR(100),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(market_area_id, property_type_id, period_start)
);

CREATE INDEX idx_trend_area ON market_trend(market_area_id);
CREATE INDEX idx_trend_period ON market_trend(period_start, period_end);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Property & Location | 4 | property, property_type, property_media, jurisdiction |
| Users & Organisations | 3 | app_user, appraiser_credential, organisation |
| Order Management | 2 | appraisal_order, appraisal_assignment |
| Appraisal Reports | 2 | appraisal_report, appraisal_revision |
| Comparables | 2 | comparable_sale, appraisal_comparable |
| AVM & Bias | 2 | avm_result, bias_audit |
| Commercial | 3 | commercial_property, lease, dcf_valuation |
| Field Inspection | 2 | inspection, floor_plan |
| MLS & Data | 2 | mls_listing, data_source_log |
| Market Data | 2 | market_area, market_trend |
| **Total** | **24** | |

---

## Key Design Decisions

1. **Separate tables for every UAD adjustment column** in `appraisal_comparable` rather than a generic key-value store — this ensures type safety and enables direct MISMO XML generation without parsing.

2. **`comparable_sale` is decoupled from `property`** with an optional foreign key — comps may come from external data sources (MLS, public records) where we do not have a full property record.

3. **pgvector column on `comparable_sale`** enables AI-powered similarity search for automated comp selection while keeping the relational structure for traditional queries.

4. **`appraisal_revision` table provides a field-level audit trail** — every change to a report is logged with old/new values, satisfying USPAP record-keeping requirements.

5. **`avm_result` stores model metadata and CFPB compliance fields** as first-class columns — confidence intervals, data quality flags, and nondiscrimination checks are not optional metadata but regulatory requirements.

6. **Commercial properties use table inheritance** (`commercial_property` extends `property` via 1:1 FK) rather than polymorphism — keeps the base property table clean while adding commercial-specific fields.

7. **MLS data stored with `raw_data` JSONB** alongside RESO-standard typed columns — normalised fields for querying, raw JSON for lossless data preservation.

8. **Market areas form a self-referencing hierarchy** — supports MSA > County > ZIP > Neighbourhood drill-downs for market analysis.

9. **`data_source_log` provides provenance tracking** — every data point used in an appraisal links back to its source with retrieval timestamp and reliability rating, as required by USPAP.

10. **ISO 3166 jurisdiction codes** used throughout rather than free-text country/state — enables multi-jurisdiction support for RICS/IVS international valuations.
