# Data Model Suggestion 4: Graph-Relational Hybrid

> Project: Real Estate Appraisal Tool · Created: 2026-05-20

## Philosophy

This model combines relational tables for structured CRUD operations with a property graph layer for relationship-rich queries. The key insight is that real estate appraisal is fundamentally a relationship problem: the value of a property depends on its similarity to comparables, its position within market areas, ownership and transaction chains, and its proximity to amenities and hazards. Traditional SQL handles entity storage well but struggles with queries like "find the 5 most similar properties within 2 miles that sold in the last 12 months" or "trace the ownership chain of this property through 3 transfers."

The relational layer handles forms, reports, compliance, and transactional data. The graph layer — implemented using PostgreSQL `ltree` for hierarchical data and a property graph schema (`graph_node`/`graph_edge` tables) for similarity and relationship networks — handles comparable discovery, market area hierarchies, and relationship traversal. For teams that need deeper graph capabilities, the same model can be backed by Neo4j or Amazon Neptune.

Cherre, the real estate data platform used by Valcre, uses exactly this pattern: a knowledge graph connecting disparate property data sources, exposed via GraphQL. CoStar's internal data model also relies on graph-like structures for market area hierarchies and property relationship networks.

**Best for:** Teams building an AI-powered platform where comparable selection, property similarity networks, market area analysis, and ownership chain traversal are core differentiators.

**Trade-offs:**
- Pro: Comparable similarity search is a native graph operation, not a complex SQL query
- Pro: Market area hierarchies (MSA > County > ZIP > Neighbourhood) are first-class graph structures
- Pro: Ownership chains, transaction networks, and conflict-of-interest analysis are natural graph queries
- Pro: Property embeddings stored as node properties enable hybrid graph + vector similarity search
- Con: Dual storage (relational + graph) increases infrastructure complexity
- Con: Graph queries have different performance characteristics than SQL (traversal depth affects latency)
- Con: Team must learn graph query patterns in addition to SQL
- Con: More complex backup, migration, and monitoring requirements
- Con: MISMO XML export and UAD compliance still require relational structures

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| UAD 3.6 | Appraisal report table stores UAD fields in relational columns; graph connects report to comparables, property, and market area nodes |
| MISMO v3.6 | MISMO XML generated from relational report tables; graph provides comp selection input |
| USPAP | Relational audit trail tables satisfy record-keeping; graph edge timestamps enable "what was connected when" temporal queries |
| CFPB AVM Rule | AVM results in relational tables; graph-based bias analysis traverses demographic-property-valuation relationships |
| ISO 19152 (LADM) | Graph nodes model LADM Party-RRR-SpatialUnit relationships: owners, rights, and parcels as connected graph entities |
| RESO Web API | MLS data ingested into both relational comparable tables and graph property nodes |
| ISO 3166 | Jurisdiction hierarchy modelled as an ltree in the graph layer |

---

## Graph Layer

```sql
-- Graph nodes — entities that participate in relationships
CREATE TABLE graph_node (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    node_type VARCHAR(50) NOT NULL,
    -- Node types:
    --   PROPERTY, COMPARABLE, MARKET_AREA, OWNER, APPRAISER,
    --   APPRAISAL_REPORT, AVM_RESULT, AMENITY, HAZARD, SCHOOL_DISTRICT
    
    entity_id UUID,                           -- FK to relational table (property.id, app_user.id, etc.)
    label VARCHAR(500),                       -- Human-readable label
    
    -- Common properties stored as typed columns for indexing
    latitude NUMERIC(10, 7),
    longitude NUMERIC(10, 7),
    
    -- Variable properties
    properties JSONB NOT NULL DEFAULT '{}',
    
    -- Vector embedding for similarity search
    embedding VECTOR(768),
    
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_node_type ON graph_node(node_type);
CREATE INDEX idx_node_entity ON graph_node(entity_id);
CREATE INDEX idx_node_location ON graph_node(latitude, longitude);
CREATE INDEX idx_node_properties ON graph_node USING gin(properties);
CREATE INDEX idx_node_embedding ON graph_node USING ivfflat (embedding vector_cosine_ops);

-- Graph edges — relationships between nodes
CREATE TABLE graph_edge (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    source_node_id UUID NOT NULL REFERENCES graph_node(id),
    target_node_id UUID NOT NULL REFERENCES graph_node(id),
    edge_type VARCHAR(50) NOT NULL,
    -- Edge types:
    --   SIMILAR_TO, COMPARABLE_FOR, LOCATED_IN, OWNED_BY, SOLD_TO,
    --   APPRAISED_BY, IN_MARKET_AREA, NEAR_AMENITY, NEAR_HAZARD,
    --   PARENT_AREA, LEASED_BY, TRANSFERRED_TO
    
    -- Edge weight / strength
    weight NUMERIC(8, 4),                     -- Similarity score, distance, etc.
    
    -- Edge properties
    properties JSONB NOT NULL DEFAULT '{}',
    -- SIMILAR_TO example: {"similarity_score": 0.92, "distance_miles": 0.8, "gla_diff": -250}
    -- SOLD_TO example: {"sale_price": 485000, "sale_date": "2026-01-15", "sale_type": "ARM_LENGTH"}
    -- IN_MARKET_AREA example: {"area_type": "ZIP", "code": "78701"}
    
    -- Temporal validity
    valid_from TIMESTAMPTZ NOT NULL DEFAULT now(),
    valid_to TIMESTAMPTZ,                     -- NULL = currently valid
    
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_edge_source ON graph_edge(source_node_id);
CREATE INDEX idx_edge_target ON graph_edge(target_node_id);
CREATE INDEX idx_edge_type ON graph_edge(edge_type);
CREATE INDEX idx_edge_weight ON graph_edge(weight);
CREATE INDEX idx_edge_validity ON graph_edge(valid_from, valid_to);
CREATE INDEX idx_edge_properties ON graph_edge USING gin(properties);

-- Market area hierarchy using ltree
CREATE TABLE market_area (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    node_id UUID NOT NULL REFERENCES graph_node(id),
    path LTREE NOT NULL,                      -- e.g., 'US.TX.AUSTIN_MSA.TRAVIS.78701.ZILKER'
    name VARCHAR(255) NOT NULL,
    area_type VARCHAR(50) NOT NULL,           -- COUNTRY, STATE, MSA, COUNTY, ZIP, NEIGHBOURHOOD, SUBMARKET
    boundary_geojson JSONB,
    
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_market_path ON market_area USING gist(path);
CREATE INDEX idx_market_type ON market_area(area_type);
```

## Relational Layer — Core Tables

```sql
-- Properties (relational CRUD + linked to graph node)
CREATE TABLE property (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    graph_node_id UUID NOT NULL REFERENCES graph_node(id),
    
    street_address VARCHAR(500) NOT NULL,
    city VARCHAR(255) NOT NULL,
    state_province VARCHAR(100),
    postal_code VARCHAR(20),
    country_code CHAR(2) NOT NULL DEFAULT 'US',
    latitude NUMERIC(10, 7),
    longitude NUMERIC(10, 7),
    parcel_number VARCHAR(100),
    
    property_type VARCHAR(50) NOT NULL,
    property_category VARCHAR(20) NOT NULL,
    year_built SMALLINT,
    gross_living_area NUMERIC(10, 2),
    lot_size NUMERIC(12, 2),
    bedroom_count SMALLINT,
    bathroom_count NUMERIC(4, 1),
    
    condition_rating VARCHAR(10),
    quality_rating VARCHAR(10),
    
    extended_attributes JSONB NOT NULL DEFAULT '{}',
    
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_property_node ON property(graph_node_id);
CREATE INDEX idx_property_location ON property(latitude, longitude);
CREATE INDEX idx_property_postal ON property(postal_code);
CREATE INDEX idx_property_type ON property(property_type);

-- Users
CREATE TABLE app_user (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    graph_node_id UUID REFERENCES graph_node(id), -- Appraisers are graph nodes
    email VARCHAR(255) NOT NULL UNIQUE,
    full_name VARCHAR(255) NOT NULL,
    role VARCHAR(50) NOT NULL,
    organisation_id UUID REFERENCES organisation(id),
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Appraiser credentials
CREATE TABLE appraiser_credential (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES app_user(id),
    jurisdiction VARCHAR(10) NOT NULL,        -- ISO 3166-2 (e.g., US-TX)
    license_number VARCHAR(100) NOT NULL,
    license_type VARCHAR(50) NOT NULL,
    license_status VARCHAR(20) NOT NULL,
    expiration_date DATE NOT NULL,
    designations TEXT[],
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_credential_user ON appraiser_credential(user_id);

-- Organisations
CREATE TABLE organisation (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    org_type VARCHAR(50) NOT NULL,
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Relational Layer — Appraisal Tables

```sql
-- Appraisal orders
CREATE TABLE appraisal_order (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_number VARCHAR(100) NOT NULL UNIQUE,
    property_id UUID NOT NULL REFERENCES property(id),
    client_id UUID NOT NULL REFERENCES organisation(id),
    appraiser_id UUID REFERENCES app_user(id),
    intended_use VARCHAR(100) NOT NULL,
    report_type VARCHAR(50) NOT NULL,
    status VARCHAR(30) NOT NULL DEFAULT 'PENDING',
    due_date DATE,
    fee_amount NUMERIC(10, 2),
    independence_confirmed BOOLEAN DEFAULT false,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_order_property ON appraisal_order(property_id);
CREATE INDEX idx_order_status ON appraisal_order(status);

-- Appraisal reports (UAD 3.6 aligned)
CREATE TABLE appraisal_report (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id UUID NOT NULL REFERENCES appraisal_order(id),
    appraiser_id UUID NOT NULL REFERENCES app_user(id),
    report_version SMALLINT NOT NULL DEFAULT 1,
    status VARCHAR(30) NOT NULL DEFAULT 'DRAFT',
    
    effective_date DATE NOT NULL,
    property_rights_appraised VARCHAR(50),
    condition_rating VARCHAR(10),
    quality_rating VARCHAR(10),
    
    -- Valuation
    cost_approach_value NUMERIC(14, 2),
    sales_comparison_value NUMERIC(14, 2),
    income_approach_value NUMERIC(14, 2),
    final_reconciled_value NUMERIC(14, 2),
    
    -- Form data
    report_data JSONB NOT NULL DEFAULT '{}',
    
    -- Submission
    mismo_xml_url TEXT,
    ucdp_submission_id VARCHAR(100),
    ucdp_status VARCHAR(30),
    signed_at TIMESTAMPTZ,
    
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_report_order ON appraisal_report(order_id);
CREATE INDEX idx_report_status ON appraisal_report(status);

-- Audit trail
CREATE TABLE report_audit_log (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    report_id UUID NOT NULL REFERENCES appraisal_report(id),
    changed_by UUID NOT NULL REFERENCES app_user(id),
    field_name VARCHAR(100) NOT NULL,
    old_value TEXT,
    new_value TEXT,
    reason TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_report ON report_audit_log(report_id);

-- Comparable sales
CREATE TABLE comparable_sale (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    graph_node_id UUID NOT NULL REFERENCES graph_node(id),
    property_id UUID REFERENCES property(id),
    data_source VARCHAR(50) NOT NULL,
    data_source_id VARCHAR(100),
    
    sale_price NUMERIC(14, 2) NOT NULL,
    sale_date DATE NOT NULL,
    sale_type VARCHAR(50),
    
    gross_living_area NUMERIC(10, 2),
    year_built SMALLINT,
    bedroom_count SMALLINT,
    bathroom_count NUMERIC(4, 1),
    condition_rating VARCHAR(10),
    quality_rating VARCHAR(10),
    
    sale_details JSONB NOT NULL DEFAULT '{}',
    
    verified BOOLEAN DEFAULT false,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_comp_node ON comparable_sale(graph_node_id);
CREATE INDEX idx_comp_sale_date ON comparable_sale(sale_date);
CREATE INDEX idx_comp_sale_price ON comparable_sale(sale_price);

-- Comparable selections (links report to comps via graph edges)
CREATE TABLE appraisal_comparable (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    report_id UUID NOT NULL REFERENCES appraisal_report(id),
    comparable_sale_id UUID NOT NULL REFERENCES comparable_sale(id),
    graph_edge_id UUID REFERENCES graph_edge(id), -- Link to COMPARABLE_FOR edge
    comp_number SMALLINT NOT NULL,
    selection_method VARCHAR(50),
    similarity_score NUMERIC(5, 4),
    
    adjustments JSONB NOT NULL DEFAULT '{}',
    net_adjustment NUMERIC(10, 2),
    adjusted_sale_price NUMERIC(14, 2),
    
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(report_id, comp_number)
);

CREATE INDEX idx_appr_comp_report ON appraisal_comparable(report_id);
```

## AVM & Commercial Tables

```sql
-- AVM results
CREATE TABLE avm_result (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    graph_node_id UUID REFERENCES graph_node(id), -- AVM as graph node for bias analysis
    property_id UUID NOT NULL REFERENCES property(id),
    report_id UUID REFERENCES appraisal_report(id),
    
    model_name VARCHAR(100) NOT NULL,
    model_version VARCHAR(50) NOT NULL,
    estimated_value NUMERIC(14, 2) NOT NULL,
    confidence_score NUMERIC(5, 4) NOT NULL,
    confidence_interval_low NUMERIC(14, 2),
    confidence_interval_high NUMERIC(14, 2),
    
    compliance JSONB NOT NULL DEFAULT '{}',
    explainability JSONB NOT NULL DEFAULT '{}',
    
    valued_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_avm_property ON avm_result(property_id);
CREATE INDEX idx_avm_node ON avm_result(graph_node_id);

-- Leases (commercial)
CREATE TABLE lease (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    property_id UUID NOT NULL REFERENCES property(id),
    tenant_name VARCHAR(255) NOT NULL,
    unit_identifier VARCHAR(100),
    lease_type VARCHAR(50) NOT NULL,
    lease_start DATE NOT NULL,
    lease_end DATE NOT NULL,
    base_rent_annual NUMERIC(12, 2) NOT NULL,
    lease_details JSONB NOT NULL DEFAULT '{}',
    source VARCHAR(50),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_lease_property ON lease(property_id);

-- Inspections
CREATE TABLE inspection (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id UUID NOT NULL REFERENCES appraisal_order(id),
    appraiser_id UUID NOT NULL REFERENCES app_user(id),
    inspection_type VARCHAR(50) NOT NULL,
    completed_at TIMESTAMPTZ,
    inspection_data JSONB NOT NULL DEFAULT '{}',
    photos JSONB NOT NULL DEFAULT '[]',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Market trends (populated from market_area graph + data feeds)
CREATE TABLE market_trend (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    market_area_id UUID NOT NULL REFERENCES market_area(id),
    property_type VARCHAR(50),
    period_start DATE NOT NULL,
    period_end DATE NOT NULL,
    metrics JSONB NOT NULL,
    source VARCHAR(100),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(market_area_id, property_type, period_start)
);

CREATE INDEX idx_trend_area ON market_trend(market_area_id);
```

## Graph Query Examples

```sql
-- ============================================================
-- Find top 10 most similar properties within 2 miles using graph
-- (Combines vector similarity with graph relationship data)
-- ============================================================
SELECT 
    p.street_address,
    p.gross_living_area,
    p.year_built,
    cs.sale_price,
    cs.sale_date,
    e.weight AS similarity_score,
    e.properties->>'distance_miles' AS distance
FROM graph_node subject_node
JOIN graph_edge e ON e.source_node_id = subject_node.id
    AND e.edge_type = 'SIMILAR_TO'
    AND (e.valid_to IS NULL OR e.valid_to > now())
JOIN graph_node comp_node ON comp_node.id = e.target_node_id
JOIN comparable_sale cs ON cs.graph_node_id = comp_node.id
    AND cs.sale_date >= now() - INTERVAL '12 months'
JOIN property p ON p.id = cs.property_id
WHERE subject_node.entity_id = :subject_property_id
    AND (e.properties->>'distance_miles')::NUMERIC <= 2.0
ORDER BY e.weight DESC
LIMIT 10;

-- ============================================================
-- Traverse ownership chain for a property
-- ============================================================
WITH RECURSIVE ownership_chain AS (
    -- Start with current property node
    SELECT 
        gn.id AS node_id,
        gn.label AS entity_name,
        e.edge_type,
        e.properties->>'sale_price' AS sale_price,
        e.properties->>'sale_date' AS sale_date,
        1 AS depth
    FROM graph_node gn
    JOIN graph_edge e ON e.target_node_id = gn.id AND e.edge_type = 'OWNED_BY'
    WHERE gn.entity_id = :property_id AND gn.node_type = 'PROPERTY'
    
    UNION ALL
    
    -- Follow TRANSFERRED_TO / SOLD_TO edges
    SELECT 
        gn2.id,
        gn2.label,
        e2.edge_type,
        e2.properties->>'sale_price',
        e2.properties->>'sale_date',
        oc.depth + 1
    FROM ownership_chain oc
    JOIN graph_edge e2 ON e2.source_node_id = oc.node_id
        AND e2.edge_type IN ('TRANSFERRED_TO', 'SOLD_TO')
    JOIN graph_node gn2 ON gn2.id = e2.target_node_id
    WHERE oc.depth < 10
)
SELECT * FROM ownership_chain ORDER BY depth;

-- ============================================================
-- Find all properties in a market area hierarchy using ltree
-- ============================================================
SELECT p.*
FROM property p
JOIN graph_node gn ON gn.entity_id = p.id AND gn.node_type = 'PROPERTY'
JOIN graph_edge e ON e.source_node_id = gn.id AND e.edge_type = 'IN_MARKET_AREA'
JOIN market_area ma ON ma.node_id = e.target_node_id
WHERE ma.path <@ 'US.TX.AUSTIN_MSA'  -- All properties in Austin MSA (any county, zip, neighbourhood)
ORDER BY p.postal_code;

-- ============================================================
-- Bias analysis: compare AVM values across demographic areas
-- (Graph traversal through property -> market_area -> demographics)
-- ============================================================
SELECT 
    ma.name AS neighbourhood,
    ma.path::TEXT AS area_path,
    AVG(avm.estimated_value) AS avg_avm_value,
    AVG(avm.confidence_score) AS avg_confidence,
    COUNT(*) AS property_count,
    AVG(p.gross_living_area) AS avg_gla,
    -- Demographic properties stored on market area graph nodes
    area_node.properties->>'median_household_income' AS median_income,
    area_node.properties->>'pct_minority' AS pct_minority
FROM avm_result avm
JOIN property p ON p.id = avm.property_id
JOIN graph_node prop_node ON prop_node.entity_id = p.id AND prop_node.node_type = 'PROPERTY'
JOIN graph_edge e ON e.source_node_id = prop_node.id AND e.edge_type = 'IN_MARKET_AREA'
JOIN market_area ma ON ma.node_id = e.target_node_id AND ma.area_type = 'NEIGHBOURHOOD'
JOIN graph_node area_node ON area_node.id = ma.node_id
WHERE avm.valued_at >= now() - INTERVAL '6 months'
GROUP BY ma.name, ma.path, area_node.properties->>'median_household_income', area_node.properties->>'pct_minority'
ORDER BY avg_avm_value;

-- ============================================================
-- AI comparable selection: combine vector similarity + graph proximity
-- ============================================================
SELECT 
    cs.id,
    cs.sale_price,
    cs.sale_date,
    p.street_address,
    -- Vector similarity from pgvector
    1 - (subject.embedding <=> comp.embedding) AS vector_similarity,
    -- Graph-based similarity (if edge exists)
    COALESCE(e.weight, 0) AS graph_similarity,
    -- Combined score
    (0.6 * (1 - (subject.embedding <=> comp.embedding)) + 0.4 * COALESCE(e.weight, 0)) AS combined_score
FROM graph_node subject
CROSS JOIN LATERAL (
    SELECT gn.*, cs2.sale_price, cs2.sale_date, cs2.id AS comp_sale_id
    FROM graph_node gn
    JOIN comparable_sale cs2 ON cs2.graph_node_id = gn.id
    WHERE gn.node_type IN ('PROPERTY', 'COMPARABLE')
        AND gn.id != subject.id
    ORDER BY subject.embedding <=> gn.embedding
    LIMIT 50
) comp
JOIN comparable_sale cs ON cs.graph_node_id = comp.id
JOIN property p ON p.id = cs.property_id
LEFT JOIN graph_edge e ON e.source_node_id = subject.id 
    AND e.target_node_id = comp.id 
    AND e.edge_type = 'SIMILAR_TO'
WHERE subject.entity_id = :subject_property_id
    AND cs.sale_date >= now() - INTERVAL '12 months'
ORDER BY combined_score DESC
LIMIT 10;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Graph Layer | 3 | graph_node, graph_edge, market_area (ltree) |
| Property | 1 | property |
| Users & Organisations | 3 | app_user, appraiser_credential, organisation |
| Orders & Reports | 4 | appraisal_order, appraisal_report, report_audit_log, appraisal_comparable |
| Comparables | 1 | comparable_sale |
| AVM | 1 | avm_result |
| Commercial | 1 | lease |
| Field Work | 1 | inspection |
| Market Data | 1 | market_trend |
| **Total** | **16** | |

---

## Key Design Decisions

1. **Dual-layer architecture (relational + graph)** — relational tables handle forms, compliance, and CRUD. The graph layer handles relationship-intensive queries (similarity, hierarchies, ownership chains). Each layer does what it does best.

2. **`graph_node.entity_id` links graph nodes to relational entities** — a property exists as both a row in `property` and a node in `graph_node`. This avoids data duplication while enabling graph traversal from any relational entity.

3. **`graph_edge.valid_from/valid_to` enables temporal graph queries** — "who owned this property in 2024?" or "what was the similarity network on the appraisal date?" are answerable by filtering edges by validity period.

4. **`ltree` for market area hierarchies** — PostgreSQL's `ltree` extension provides efficient ancestor/descendant queries. "Find all properties in the Austin MSA" is a single ltree containment query, regardless of how many levels deep the hierarchy goes.

5. **SIMILAR_TO edges pre-computed from embeddings** — rather than computing vector similarity at query time every time, the system periodically builds SIMILAR_TO edges between properties with high embedding similarity. This makes comp discovery a simple edge traversal instead of a full vector scan.

6. **Appraisers are graph nodes** — enables "which appraiser has appraised properties similar to this one?" and conflict-of-interest queries by traversing APPRAISED_BY edges.

7. **AVM results are graph nodes for bias analysis** — connecting AVM valuations to properties, market areas, and demographic data through the graph enables the bias traversal queries required by the CFPB AVM Rule.

8. **Combined vector + graph similarity scoring** — the AI comparable selection query uses both pgvector distance (property characteristic similarity) and graph edge weight (relationship-based similarity) to produce a blended score. This outperforms either approach alone.

9. **Graph properties stored as JSONB** — node and edge properties vary by type (a SIMILAR_TO edge has different properties than a SOLD_TO edge). JSONB with GIN indexes provides type-flexible property storage.

10. **Relational tables remain the system of record for compliance** — the graph is an acceleration layer, not a replacement for the relational audit trail. MISMO XML, USPAP audit logs, and UAD form data all come from relational tables.
