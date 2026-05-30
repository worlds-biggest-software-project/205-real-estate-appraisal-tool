# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: Real Estate Appraisal Tool · Created: 2026-05-20

## Philosophy

This model uses typed relational columns for core, universally-present fields (property address, sale price, valuation conclusion) while storing variable, form-specific, and jurisdiction-dependent fields in JSONB columns. The insight is that while every appraisal has a subject property, comparables, and a reconciled value, the specific fields required vary dramatically between residential UAD 3.6 reports, commercial DCF valuations, RICS Red Book reports, and IVS-compliant international appraisals.

Rather than creating dozens of nullable columns or separate tables for each form variation, JSONB columns hold the variable fields with schema-on-read validation at the application layer. A `form_template` system defines which JSONB fields are required for each report type and jurisdiction, enabling the platform to support new form types and international standards without DDL migrations.

This pattern is widely used in modern SaaS platforms that serve multiple markets — Shopify stores product attributes in JSONB, Stripe stores payment method details in flexible JSON, and healthcare EHR systems use FHIR's JSON-based extensions for the same reason. For an appraisal platform that needs to support UAD 3.6 today, RICS Red Book tomorrow, and unknown future regulatory schemas, this flexibility is valuable.

**Best for:** Teams building a multi-market platform that must support residential, commercial, and international appraisal standards with rapid iteration and without heavy migration overhead.

**Trade-offs:**
- Pro: New form types and jurisdictions added without schema migrations
- Pro: Fewer tables (~15) — simpler to understand, deploy, and maintain
- Pro: Core relational fields enable standard SQL queries and indexing
- Pro: JSONB GIN indexes support efficient queries within variable fields
- Pro: Rapid MVP development — add fields in application code, not database DDL
- Con: JSONB fields lack database-level type enforcement and NOT NULL constraints
- Con: Application must validate JSONB structure against form templates
- Con: Complex JSONB queries (deeply nested paths) are slower than indexed columns
- Con: MISMO XML generation requires mapping from JSONB paths to XML elements
- Con: Database documentation is less self-describing — must reference form templates to understand JSONB structure

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| UAD 3.6 | UAD fields stored in `report_data` JSONB column; `form_template` for UAD 3.6 defines required fields, validation rules, and UAD standardised value enums |
| RICS Red Book | Separate `form_template` for RICS reports; different `report_data` JSONB structure with RICS-specific fields (ESG factors, valuation uncertainty) |
| IVS | IVS-compliant form template with international valuation fields |
| MISMO v3.6 | JSONB-to-MISMO mapping table defines how to serialise each `report_data` path to MISMO XML elements |
| USPAP | `revision_history` JSONB array on reports captures field-level changes; `data_sources` JSONB documents provenance |
| CFPB AVM Rule | AVM results stored with compliance fields in typed columns; explainability in JSONB |
| RESO Web API | MLS data ingested into `raw_mls_data` JSONB; RESO standard fields extracted to typed columns |
| ISO 3166 | Jurisdiction codes as typed columns on core tables |

---

## Form Template System

```sql
-- Form templates define the JSONB schema for each report type
CREATE TABLE form_template (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code VARCHAR(50) NOT NULL UNIQUE,         -- e.g., UAD_3_6_URAR, RICS_RED_BOOK, IVS_COMMERCIAL
    name VARCHAR(255) NOT NULL,
    version VARCHAR(20) NOT NULL,
    jurisdiction_country CHAR(2),              -- NULL = universal
    property_category VARCHAR(50),             -- RESIDENTIAL, COMMERCIAL, MIXED, ALL
    
    -- JSON Schema defining required/optional JSONB fields
    field_schema JSONB NOT NULL,
    -- Example:
    -- {
    --   "subject": {
    --     "property_rights_appraised": {"type": "string", "enum": ["FEE_SIMPLE", "LEASEHOLD", "LEASED_FEE"], "required": true},
    --     "condition_rating": {"type": "string", "enum": ["C1","C2","C3","C4","C5","C6"], "required": true},
    --     "quality_rating": {"type": "string", "enum": ["Q1","Q2","Q3","Q4","Q5","Q6"], "required": true},
    --     "site_area": {"type": "number", "required": true},
    --     "topography": {"type": "string", "required": false}
    --   },
    --   "improvements": {
    --     "foundation_type": {"type": "string", "required": true},
    --     "exterior_walls": {"type": "string", "required": true}
    --   },
    --   "neighbourhood": {
    --     "name": {"type": "string", "required": true},
    --     "market_conditions": {"type": "string", "enum": ["STABLE","DECLINING","INCREASING"], "required": true}
    --   }
    -- }
    
    -- MISMO mapping for XML export
    mismo_field_map JSONB,
    -- Example:
    -- {
    --   "subject.condition_rating": "PROPERTY/STRUCTURE/CONDITION_RATING",
    --   "subject.quality_rating": "PROPERTY/STRUCTURE/QUALITY_RATING",
    --   "subject.site_area": "PROPERTY/SITE/SITE_AREA"
    -- }
    
    is_active BOOLEAN NOT NULL DEFAULT true,
    effective_date DATE,
    sunset_date DATE,                         -- When this template version expires
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_form_template_code ON form_template(code);
CREATE INDEX idx_form_template_jurisdiction ON form_template(jurisdiction_country);
```

## Core Tables

```sql
-- Properties (core fields relational, variable fields JSONB)
CREATE TABLE property (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    
    -- Core relational fields (always present, always queried)
    street_address VARCHAR(500) NOT NULL,
    city VARCHAR(255) NOT NULL,
    state_province VARCHAR(100),
    postal_code VARCHAR(20),
    country_code CHAR(2) NOT NULL DEFAULT 'US',
    latitude NUMERIC(10, 7),
    longitude NUMERIC(10, 7),
    parcel_number VARCHAR(100),
    property_type VARCHAR(50) NOT NULL,       -- SFR, CONDO, COMMERCIAL, MULTIFAMILY
    property_category VARCHAR(20) NOT NULL,   -- RESIDENTIAL, COMMERCIAL
    
    -- Key numeric fields for filtering and comp search
    year_built SMALLINT,
    gross_living_area NUMERIC(10, 2),
    lot_size NUMERIC(12, 2),
    bedroom_count SMALLINT,
    bathroom_count NUMERIC(4, 1),
    
    -- Variable/extended property characteristics
    extended_attributes JSONB NOT NULL DEFAULT '{}',
    -- Residential example:
    -- {
    --   "stories": 2,
    --   "garage_spaces": 2,
    --   "pool": true,
    --   "basement_type": "FULL_FINISHED",
    --   "basement_sqft": 1200,
    --   "heating": "FORCED_AIR",
    --   "cooling": "CENTRAL",
    --   "fireplace_count": 1,
    --   "hoa_fee_monthly": 250
    -- }
    -- Commercial example:
    -- {
    --   "subtype": "OFFICE",
    --   "total_rentable_area": 45000,
    --   "number_of_units": 24,
    --   "number_of_floors": 4,
    --   "parking_spaces": 80,
    --   "occupancy_rate": 0.92,
    --   "noi": 850000,
    --   "cap_rate": 0.065,
    --   "elevator": true,
    --   "sprinkler_system": true
    -- }
    
    -- AI features
    property_embedding VECTOR(768),
    ai_condition_score NUMERIC(3, 2),
    
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_property_location ON property(latitude, longitude);
CREATE INDEX idx_property_postal ON property(postal_code);
CREATE INDEX idx_property_type ON property(property_type);
CREATE INDEX idx_property_category ON property(property_category);
CREATE INDEX idx_property_parcel ON property(parcel_number);
CREATE INDEX idx_property_extended ON property USING gin(extended_attributes);
CREATE INDEX idx_property_embedding ON property USING ivfflat (property_embedding vector_cosine_ops);

-- Users
CREATE TABLE app_user (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) NOT NULL UNIQUE,
    full_name VARCHAR(255) NOT NULL,
    role VARCHAR(50) NOT NULL,
    organisation_id UUID REFERENCES organisation(id),
    
    -- Appraiser-specific fields (NULL for non-appraisers)
    credentials JSONB,
    -- {
    --   "licenses": [
    --     {
    --       "jurisdiction": "US-TX",
    --       "license_number": "TX-12345",
    --       "license_type": "CERTIFIED_GENERAL",
    --       "status": "ACTIVE",
    --       "expiration_date": "2027-06-30",
    --       "designations": ["MAI", "SRA"]
    --     }
    --   ],
    --   "uspap_update_date": "2025-09-15",
    --   "specialties": ["RESIDENTIAL", "COMMERCIAL_OFFICE"]
    -- }
    
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_user_role ON app_user(role);
CREATE INDEX idx_user_org ON app_user(organisation_id);
CREATE INDEX idx_user_credentials ON app_user USING gin(credentials);

-- Organisations
CREATE TABLE organisation (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    org_type VARCHAR(50) NOT NULL,
    country_code CHAR(2) DEFAULT 'US',
    details JSONB NOT NULL DEFAULT '{}',
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Appraisal & Report Tables

```sql
-- Appraisal orders
CREATE TABLE appraisal_order (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_number VARCHAR(100) NOT NULL UNIQUE,
    property_id UUID NOT NULL REFERENCES property(id),
    client_id UUID NOT NULL REFERENCES organisation(id),
    appraiser_id UUID REFERENCES app_user(id),
    
    -- Core order fields
    intended_use VARCHAR(100) NOT NULL,
    form_template_id UUID NOT NULL REFERENCES form_template(id),
    status VARCHAR(30) NOT NULL DEFAULT 'PENDING',
    due_date DATE,
    rush BOOLEAN DEFAULT false,
    fee_amount NUMERIC(10, 2),
    
    -- Assignment details
    assignment_details JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "assigned_by": "user-uuid",
    --   "assigned_at": "2026-05-15T10:30:00Z",
    --   "accepted_at": "2026-05-15T14:00:00Z",
    --   "independence_confirmed": true,
    --   "engagement_letter_url": "https://..."
    -- }
    
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_order_property ON appraisal_order(property_id);
CREATE INDEX idx_order_status ON appraisal_order(status);
CREATE INDEX idx_order_appraiser ON appraisal_order(appraiser_id);
CREATE INDEX idx_order_due ON appraisal_order(due_date);

-- Appraisal reports (core + JSONB form data)
CREATE TABLE appraisal_report (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id UUID NOT NULL REFERENCES appraisal_order(id),
    appraiser_id UUID NOT NULL REFERENCES app_user(id),
    form_template_id UUID NOT NULL REFERENCES form_template(id),
    report_version SMALLINT NOT NULL DEFAULT 1,
    status VARCHAR(30) NOT NULL DEFAULT 'DRAFT',
    
    -- Core valuation fields (always present, always queried)
    effective_date DATE NOT NULL,
    final_reconciled_value NUMERIC(14, 2),
    cost_approach_value NUMERIC(14, 2),
    sales_comparison_value NUMERIC(14, 2),
    income_approach_value NUMERIC(14, 2),
    
    -- Form-specific data (structure varies by form_template)
    report_data JSONB NOT NULL DEFAULT '{}',
    -- UAD 3.6 example:
    -- {
    --   "subject": {
    --     "property_rights_appraised": "FEE_SIMPLE",
    --     "condition_rating": "C3",
    --     "quality_rating": "Q3",
    --     "site_area": 8500,
    --     "site_shape": "RECTANGULAR",
    --     "topography": "LEVEL"
    --   },
    --   "improvements": {
    --     "foundation_type": "CONCRETE_SLAB",
    --     "exterior_walls": "BRICK_VENEER",
    --     "roof_surface": "COMPOSITION_SHINGLE",
    --     "heating": "FORCED_AIR_GAS",
    --     "cooling": "CENTRAL_ELECTRIC"
    --   },
    --   "neighbourhood": {
    --     "name": "Zilker",
    --     "boundaries": "N: Barton Creek; S: Oltorf; E: S. Lamar; W: MoPac",
    --     "market_conditions": "INCREASING",
    --     "built_up": "OVER_75",
    --     "growth": "RAPID"
    --   },
    --   "narrative": {
    --     "scope_of_work": "Full interior/exterior inspection...",
    --     "market_analysis": "The Zilker neighbourhood...",
    --     "highest_best_use": "Continued residential use..."
    --   }
    -- }
    
    -- Data sources and provenance (USPAP)
    data_sources JSONB NOT NULL DEFAULT '[]',
    -- [
    --   {"type": "MLS", "name": "ACTRIS", "ref": "A12345", "retrieved_at": "2026-05-14", "reliability": "HIGH"},
    --   {"type": "PUBLIC_RECORD", "name": "Travis County", "ref": "0123456789", "retrieved_at": "2026-05-14"}
    -- ]
    
    -- Revision history (USPAP audit trail)
    revision_history JSONB NOT NULL DEFAULT '[]',
    -- [
    --   {"at": "2026-05-15T10:30:00Z", "by": "user-uuid", "field": "subject.condition_rating", "from": "C4", "to": "C3", "reason": "Updated after interior inspection"},
    --   {"at": "2026-05-15T11:00:00Z", "by": "user-uuid", "field": "final_reconciled_value", "from": 475000, "to": 485000, "reason": "Adjusted after reviewing comp 3"}
    -- ]
    
    -- Submission
    mismo_xml_url TEXT,
    ucdp_submission_id VARCHAR(100),
    ucdp_status VARCHAR(30),
    signed_at TIMESTAMPTZ,
    
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_report_order ON appraisal_report(order_id);
CREATE INDEX idx_report_appraiser ON appraisal_report(appraiser_id);
CREATE INDEX idx_report_status ON appraisal_report(status);
CREATE INDEX idx_report_effective ON appraisal_report(effective_date);
CREATE INDEX idx_report_data ON appraisal_report USING gin(report_data);
CREATE INDEX idx_report_value ON appraisal_report(final_reconciled_value);
```

## Comparables Table

```sql
-- Comparable sales (core + JSONB)
CREATE TABLE comparable_sale (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    property_id UUID REFERENCES property(id),
    data_source VARCHAR(50) NOT NULL,
    data_source_id VARCHAR(100),
    
    -- Core fields (always queried)
    sale_price NUMERIC(14, 2) NOT NULL,
    sale_date DATE NOT NULL,
    latitude NUMERIC(10, 7),
    longitude NUMERIC(10, 7),
    gross_living_area NUMERIC(10, 2),
    year_built SMALLINT,
    bedroom_count SMALLINT,
    bathroom_count NUMERIC(4, 1),
    
    -- Variable fields
    sale_details JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "sale_type": "ARM_LENGTH",
    --   "financing_type": "CONVENTIONAL",
    --   "concessions_amount": 5000,
    --   "days_on_market": 21,
    --   "condition_rating": "C3",
    --   "quality_rating": "Q3",
    --   "lot_size": 7500,
    --   "stories": 2,
    --   "garage_spaces": 2,
    --   "pool": false
    -- }
    
    -- MLS raw data (lossless)
    raw_mls_data JSONB,
    
    -- AI
    property_embedding VECTOR(768),
    verified BOOLEAN DEFAULT false,
    
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_comp_sale_date ON comparable_sale(sale_date);
CREATE INDEX idx_comp_sale_price ON comparable_sale(sale_price);
CREATE INDEX idx_comp_location ON comparable_sale(latitude, longitude);
CREATE INDEX idx_comp_details ON comparable_sale USING gin(sale_details);
CREATE INDEX idx_comp_embedding ON comparable_sale USING ivfflat (property_embedding vector_cosine_ops);

-- Comparable selections and adjustments for a report
CREATE TABLE appraisal_comparable (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    report_id UUID NOT NULL REFERENCES appraisal_report(id),
    comparable_sale_id UUID NOT NULL REFERENCES comparable_sale(id),
    comp_number SMALLINT NOT NULL,
    selection_method VARCHAR(50),
    similarity_score NUMERIC(5, 4),
    
    -- Adjustments stored as JSONB (varies by form template)
    adjustments JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "location": -5000,
    --   "site": 2500,
    --   "quality": 0,
    --   "condition": -7500,
    --   "gla": -12500,
    --   "room_count": 3000,
    --   "basement": 0,
    --   "garage": 5000,
    --   "other": 0,
    --   "net_adjustment": -14500,
    --   "adjusted_sale_price": 470500,
    --   "justification": "Subject smaller GLA, better condition..."
    -- }
    
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(report_id, comp_number)
);

CREATE INDEX idx_appr_comp_report ON appraisal_comparable(report_id);
```

## AVM, Inspection & Market Tables

```sql
-- AVM results
CREATE TABLE avm_result (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    property_id UUID NOT NULL REFERENCES property(id),
    report_id UUID REFERENCES appraisal_report(id),
    
    -- Core AVM fields
    model_name VARCHAR(100) NOT NULL,
    model_version VARCHAR(50) NOT NULL,
    estimated_value NUMERIC(14, 2) NOT NULL,
    confidence_score NUMERIC(5, 4) NOT NULL,
    confidence_interval_low NUMERIC(14, 2),
    confidence_interval_high NUMERIC(14, 2),
    
    -- Compliance and explainability (JSONB for flexibility)
    compliance JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "data_sources_used": ["MLS_ACTRIS", "TRAVIS_COUNTY_TAX"],
    --   "conflict_of_interest_check": true,
    --   "random_sample_tested": false,
    --   "nondiscrimination_check_passed": true,
    --   "data_quality_flags": []
    -- }
    
    explainability JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "top_factors": [{"factor": "GLA", "impact": 0.32}],
    --   "comparable_ids": ["uuid-1", "uuid-2"],
    --   "methodology": "ENSEMBLE_GRADIENT_BOOST"
    -- }
    
    valued_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_avm_property ON avm_result(property_id);
CREATE INDEX idx_avm_report ON avm_result(report_id);
CREATE INDEX idx_avm_valued ON avm_result(valued_at);

-- Inspections
CREATE TABLE inspection (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id UUID NOT NULL REFERENCES appraisal_order(id),
    appraiser_id UUID NOT NULL REFERENCES app_user(id),
    inspection_type VARCHAR(50) NOT NULL,
    status VARCHAR(30) NOT NULL DEFAULT 'SCHEDULED',
    scheduled_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    
    -- Inspection data (varies by type)
    inspection_data JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "weather": "Clear, 85F",
    --   "access_notes": "Met homeowner at front door",
    --   "observations": [
    --     {"area": "ROOF", "condition": "GOOD", "notes": "No visible damage"},
    --     {"area": "FOUNDATION", "condition": "FAIR", "notes": "Minor settling crack on east wall"}
    --   ]
    -- }
    
    -- Photos stored as JSONB array
    photos JSONB NOT NULL DEFAULT '[]',
    -- [
    --   {"url": "https://...", "category": "FRONT", "caption": "Front elevation", "ai_score": 0.85},
    --   {"url": "https://...", "category": "KITCHEN", "caption": "Kitchen — updated 2020"}
    -- ]
    
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_inspection_order ON inspection(order_id);

-- Market trends
CREATE TABLE market_trend (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    market_area VARCHAR(255) NOT NULL,
    area_type VARCHAR(50) NOT NULL,           -- MSA, COUNTY, ZIP, SUBMARKET
    property_type VARCHAR(50),
    period_start DATE NOT NULL,
    period_end DATE NOT NULL,
    
    metrics JSONB NOT NULL,
    -- {
    --   "median_sale_price": 425000,
    --   "avg_sale_price": 448000,
    --   "sale_count": 342,
    --   "median_dom": 18,
    --   "price_per_sqft": 195.50,
    --   "inventory_months": 2.3,
    --   "avg_cap_rate": 0.058,    -- commercial only
    --   "avg_vacancy_rate": 0.07  -- commercial only
    -- }
    
    source VARCHAR(100),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(market_area, property_type, period_start)
);

CREATE INDEX idx_trend_area ON market_trend(market_area, area_type);
CREATE INDEX idx_trend_period ON market_trend(period_start);
```

## Query Examples

```sql
-- Find all residential properties in Austin with pools (JSONB query)
SELECT id, street_address, gross_living_area, extended_attributes->>'pool' AS has_pool
FROM property
WHERE city = 'Austin'
  AND property_category = 'RESIDENTIAL'
  AND extended_attributes @> '{"pool": true}';

-- Find comps with specific sale type (JSONB containment)
SELECT id, sale_price, sale_date
FROM comparable_sale
WHERE sale_details @> '{"sale_type": "ARM_LENGTH"}'
  AND sale_date >= '2025-11-01'
  AND sale_price BETWEEN 400000 AND 550000;

-- Get all report revisions for audit (JSONB array expansion)
SELECT
    r.id AS report_id,
    rev->>'at' AS changed_at,
    rev->>'by' AS changed_by,
    rev->>'field' AS field_changed,
    rev->>'from' AS old_value,
    rev->>'to' AS new_value,
    rev->>'reason' AS reason
FROM appraisal_report r,
     jsonb_array_elements(r.revision_history) AS rev
WHERE r.id = :report_id
ORDER BY rev->>'at';

-- Validate report against form template (application layer)
-- SELECT field_schema FROM form_template WHERE id = :template_id
-- Application validates report_data JSONB against field_schema
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Form Templates | 1 | form_template |
| Properties | 1 | property (with extended_attributes JSONB) |
| Users & Organisations | 2 | app_user (with credentials JSONB), organisation |
| Orders & Reports | 3 | appraisal_order, appraisal_report, appraisal_comparable |
| Comparables | 1 | comparable_sale (with sale_details JSONB) |
| AVM | 1 | avm_result (with compliance/explainability JSONB) |
| Field Work | 1 | inspection (with inspection_data/photos JSONB) |
| Market Data | 1 | market_trend (with metrics JSONB) |
| **Total** | **11** | |

---

## Key Design Decisions

1. **Core fields are relational, variable fields are JSONB** — fields that appear in WHERE clauses, ORDER BY, or JOIN conditions are typed columns. Fields that vary by form type, jurisdiction, or property category go in JSONB. This boundary is drawn based on query patterns, not aesthetics.

2. **`form_template` system enables multi-standard support** — adding RICS Red Book or IVS support means creating a new template row with its `field_schema`, not adding 50 nullable columns or new tables. The same `appraisal_report` table serves all standards.

3. **`mismo_field_map` on form templates enables JSONB-to-XML serialisation** — the mapping from JSONB paths to MISMO XML element paths is data, not code. New MISMO versions or fields are handled by updating the mapping, not rewriting the export module.

4. **Revision history embedded as JSONB array on the report** — for most reports, the revision history is small (10-50 entries). Storing it inline eliminates a join and keeps the audit trail co-located with the report. For heavy-write scenarios, this can be moved to a separate table.

5. **Appraiser credentials stored as JSONB on `app_user`** — appraisers may hold licenses in multiple states with different types and expiration dates. JSONB handles this variable-length, structured data naturally without a junction table.

6. **Comparable adjustments stored as JSONB** — the set of adjustment fields varies between UAD residential comps and commercial comps. JSONB accommodates both without separate tables or nullable column sprawl.

7. **GIN indexes on all JSONB columns** — enables efficient containment queries (`@>`) for filtering by JSONB field values. Critical for queries like "find all properties with pool" or "find all arm's length sales."

8. **11 tables total vs 24+ in the normalised model** — significantly simpler schema to deploy, back up, and reason about. The trade-off is that field-level database constraints are weaker.

9. **Raw MLS data preserved as JSONB** — the full RESO API response is stored losslessly alongside extracted typed columns, enabling re-extraction if the normalisation logic changes.

10. **Photos stored as JSONB array on inspection** rather than a separate table — inspection photos are always accessed together with the inspection. The JSONB array eliminates a join for the most common access pattern (loading an inspection with its photos).
