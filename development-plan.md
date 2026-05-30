# Real Estate Appraisal Tool — Phased Development Plan

> Project: 205-real-estate-appraisal-tool · Created: 2026-05-29
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Language | Python 3.12+ (backend) + TypeScript (frontend) | Python for MISMO XML generation (lxml), ML-based comp selection (scikit-learn, sentence-transformers), LLM narrative generation (Anthropic SDK), and PDF report rendering. TypeScript frontend for appraiser dashboard. |
| API framework | FastAPI | Async for concurrent MLS/public-record data fetches; auto-generates OpenAPI 3.1; Pydantic v2 for strict UAD 3.6 field validation |
| Database | PostgreSQL 16 | 60+ normalised tables from data-model-suggestion-1 aligned with MISMO v3.6; PostGIS for comp proximity queries and property geocoding |
| Migrations | Alembic | Version-controlled schema; critical given high table count and UAD 3.6 mandate deadline |
| ORM | SQLAlchemy 2.0 (async) | Mapped dataclasses; complex joins across appraisal→comps→adjustments→reports |
| Task queue | Celery + Redis | Async MLS data fetches, AVM scoring, report PDF generation, MISMO XML export, bias auditing |
| Frontend | Next.js 15 (App Router) + Tailwind CSS | Appraiser dashboard, comp selection UI, report editor, inspection photo viewer |
| XML generation | lxml | MISMO v3.6 XML generation for UCDP/GSE submission; XSD validation |
| PDF reports | WeasyPrint | URAR form PDFs matching GSE layout requirements; USPAP-compliant narrative formatting |
| ML / Embeddings | sentence-transformers + pgvector | Embedding-based comparable selection; property similarity search using vector embeddings of property characteristics |
| LLM integration | Anthropic Python SDK (Claude) | USPAP narrative drafting, market analysis commentary, bias audit explanations |
| Maps | Mapbox GL JS | Comp map with radius/polygon selection; subject property pin |
| Mobile | React Native (Expo) | Field inspection app with camera, offline capability, sketch tool |
| Containerisation | Docker + docker-compose | PostgreSQL+PostGIS+pgvector, Redis, FastAPI, Celery, Next.js |
| Testing | pytest + pytest-asyncio + httpx + Vitest | Python backend tests; Vitest for frontend |
| Linting | Ruff (Python) + Biome (TS) | Fast linter/formatter for both stacks |
| Type checking | mypy (strict) + TypeScript strict | Both stacks type-checked |
| Package manager | uv (Python) + pnpm (TS) | Fast, lockfile-based |

### Project Structure

```
appraisal-tool/
├── pyproject.toml
├── docker-compose.yml
├── Dockerfile.api
├── Dockerfile.frontend
├── alembic.ini
├── alembic/versions/
├── src/
│   ├── __init__.py
│   ├── main.py                        # FastAPI app factory
│   ├── config.py
│   ├── database.py
│   ├── models/                        # SQLAlchemy ORM (from data-model-suggestion-1)
│   │   ├── property.py
│   │   ├── appraisal.py
│   │   ├── comparable.py
│   │   ├── adjustment.py
│   │   ├── report.py
│   │   ├── appraiser.py
│   │   ├── avm_result.py
│   │   ├── inspection.py
│   │   └── audit_log.py
│   ├── schemas/                       # Pydantic request/response
│   │   ├── appraisals.py
│   │   ├── comparables.py
│   │   ├── reports.py
│   │   ├── uad.py                     # UAD 3.6 field validators
│   │   └── mismo.py                   # MISMO XML schemas
│   ├── api/
│   │   ├── appraisals.py
│   │   ├── comparables.py
│   │   ├── reports.py
│   │   ├── properties.py
│   │   ├── inspections.py
│   │   ├── avm.py
│   │   └── ai.py
│   ├── services/
│   │   ├── appraisal_manager.py
│   │   ├── comp_selector.py           # AI comp selection
│   │   ├── adjustment_engine.py
│   │   ├── report_generator.py        # URAR PDF + MISMO XML
│   │   ├── narrative_drafter.py       # LLM narrative generation
│   │   ├── avm_engine.py
│   │   ├── bias_auditor.py
│   │   ├── mls_client.py
│   │   ├── public_records.py
│   │   └── data_normaliser.py
│   ├── ml/
│   │   ├── property_embeddings.py     # sentence-transformers
│   │   ├── comp_ranking.py
│   │   └── condition_assessor.py      # Image recognition
│   ├── tasks/
│   │   ├── reports.py
│   │   ├── avm.py
│   │   ├── mls_sync.py
│   │   └── bias.py
│   └── lib/
│       ├── mismo_xml.py               # MISMO v3.6 XML builder
│       ├── uad_validation.py          # UAD 3.6 field validation
│       ├── pdf.py                     # WeasyPrint URAR generation
│       └── geo.py                     # PostGIS helpers
├── frontend/
│   ├── package.json
│   └── src/
│       ├── app/
│       │   ├── appraisals/
│       │   ├── comps/
│       │   ├── reports/
│       │   ├── inspections/
│       │   └── settings/
│       └── components/
│           ├── CompSelector.tsx
│           ├── AdjustmentGrid.tsx
│           ├── CompMap.tsx
│           ├── ReportEditor.tsx
│           ├── InspectionPhotos.tsx
│           └── BiasAuditBadge.tsx
├── mobile/                            # React Native (Expo) inspection app
├── tests/
│   ├── unit/
│   ├── integration/
│   └── fixtures/
│       ├── sample_mismo_v36.xml
│       ├── sample_urar.json
│       └── sample_comps.json
└── templates/
    ├── urar_form.html                 # WeasyPrint template for URAR PDF
    └── narrative_sections.html
```

---

## Phase 1: Foundation

### Purpose
Establish the project skeleton, database schema from data-model-suggestion-1 (properties, appraisals, comparables, appraisers, reports), authentication, and Docker environment. After this phase, users can create organisations, add properties, and assign appraisers.

### Tasks

#### 1.1 — Project Scaffold and Configuration

**What**: Create pyproject.toml, FastAPI app, Pydantic Settings config, Docker with PostgreSQL+PostGIS+pgvector, Redis.

**Design**:

```python
class Settings(BaseSettings):
    database_url: str = "postgresql+asyncpg://appraisal:appraisal@localhost:5432/appraisal"
    redis_url: str = "redis://localhost:6379/0"
    anthropic_api_key: str = ""
    mls_api_url: str = ""
    mls_api_key: str = ""
    mapbox_token: str = ""
    site_url: str = "http://localhost:3000"
    mismo_xsd_path: str = "schemas/mismo_v36.xsd"
    model_config = {"env_prefix": "APPRAISAL_"}
```

**Testing**:
- Unit: config parses env correctly
- Integration: `docker-compose up -d` → PostgreSQL with PostGIS and pgvector extensions enabled

#### 1.2 — Database Schema — Core Tables

**What**: Implement jurisdiction, property, property_type, appraisal, appraiser, appraisal_assignment tables from data-model-suggestion-1.

**Design**:

Tables from data-model-suggestion-1: `jurisdiction` (ISO 3166, valuation_standard), `property` (all UAD 3.6 fields: address, GLA, lot_size, bedrooms, bathrooms, year_built, parcel_number, lat/lng), `property_type` (residential/commercial/land), `appraiser` (licence_number, licence_state, licence_expiry, certifications), `appraisal` (subject_property_id, appraiser_id, purpose, effective_date, status lifecycle), `appraisal_assignment` (AMC tracking, independence certification per FIRREA).

Appraisal state machine: `draft` → `in_progress` → `review` → `completed` → `submitted` → `revision_requested` → `in_progress` (loop) → `delivered`.

**Testing**:
- Integration: migrations apply, PostGIS + pgvector extensions created
- Integration: create property → appraisal → appraiser assignment → FK chain holds
- Unit: appraisal status transitions enforced (cannot skip from draft to submitted)

#### 1.3 — Authentication and Roles

**What**: Auth for appraisers, reviewers, and admin users; API key for AMC/lender integrations.

**Design**:

Roles: `appraiser` (create/edit appraisals, select comps), `reviewer` (approve/request revision), `admin` (manage users, settings), `api_service` (AMC/lender integration). FIRREA independence: appraiser cannot be assigned to appraisal if they have a conflict_of_interest flag for that lender.

**Testing**:
- Unit: appraiser can create appraisal → allowed
- Unit: reviewer can approve → allowed
- Unit: appraiser accessing another appraiser's draft → 403

---

## Phase 2: Comparable Sales Data and Selection

### Purpose
Connect to MLS and public record data sources, normalise comp data, and build the comparable selection interface. This is the core daily workflow for appraisers.

### Tasks

#### 2.1 — MLS and Public Record Data Integration

**What**: Fetch comparable sales from MLS (via RESO Web API) and public records; normalise into unified comp schema.

**Design**:

```python
class MLSClient:
    async def search_comps(self, subject: Property, radius_miles: float = 1.0,
                           months_back: int = 12, max_results: int = 50) -> list[ComparableSale]:
        """Query RESO Web API with OData $filter:
        CloseDate gt {months_back_date} AND
        PropertyType eq '{subject.property_type}' AND
        geo.distance(latitude,longitude,{lat},{lng}) lt {radius_miles * 1609}
        """

class DataNormaliser:
    def normalise(self, mls_record: dict, public_record: dict | None) -> ComparableSale:
        """Reconcile MLS and public record data:
        - Prefer MLS for sale price, close date, listing info
        - Prefer public records for GLA, lot size, year built (more accurate)
        - Flag discrepancies for appraiser review
        """
```

**Testing**:
- Integration (mocked RESO): search within 1 mile → returns 15 comps with normalised fields
- Unit: MLS says 1,800 sqft, public record says 1,750 → discrepancy flagged, public record preferred
- Unit: no public record match → MLS data used with warning

#### 2.2 — Comparable Selection UI with Map

**What**: Map-based comp selector with filters, side-by-side comparison, and adjustment justification.

**Design**:

`/appraisals/[id]/comps` page: Mapbox map with subject property pin (blue) and comp pins (green for selected, grey for available). Sidebar filters: price range, GLA range, year built range, proximity radius. Click comp → side-by-side comparison card showing subject vs. comp with differences highlighted.

Selection: appraiser selects 3-6 comps; each selected comp creates a `comparable_sale` record linked to the appraisal.

**Testing**:
- E2E: load comp page → map shows subject and available comps
- E2E: select 3 comps → appear in "Selected Comps" panel
- E2E: filter by price → map updates to show only matching comps
- Unit: comp > 1 mile away → flagged with distance warning

#### 2.3 — Adjustment Grid

**What**: Side-by-side adjustment grid where appraiser enters adjustments for each comp relative to the subject.

**Design**:

```typescript
interface AdjustmentRow {
  field: string;              // "GLA", "Bedrooms", "Garage", "Pool", "Condition", "Location"
  subjectValue: string;
  compValue: string;
  adjustmentCents: number;    // positive = comp inferior, negative = comp superior
  justification: string;
}

// GET /api/v1/appraisals/{id}/comps/{compId}/adjustments → AdjustmentRow[]
// PUT /api/v1/appraisals/{id}/comps/{compId}/adjustments → save grid
```

Adjusted price computed: `comp.sale_price + sum(adjustments)`. Final reconciled value is appraiser's weighted conclusion from adjusted prices.

**Testing**:
- Unit: comp with 200 fewer sqft at $100/sqft → GLA adjustment = +$20,000
- E2E: enter adjustments → adjusted price updates in real time
- Unit: total adjustments > 25% of sale price → warning "Large net adjustment"

---

## Phase 3: Report Generation and MISMO XML

### Purpose
Generate USPAP-compliant appraisal reports as PDFs and MISMO v3.6 XML for GSE submission. This is the deliverable that appraisers produce.

### Tasks

#### 3.1 — URAR PDF Report Generation

**What**: Generate the Uniform Residential Appraisal Report (URAR) as a PDF matching GSE form layout.

**Design**:

WeasyPrint renders HTML template → PDF. Template matches UAD 3.6 URAR form layout: Subject Section, Contract Section, Neighbourhood Section, Site Section, Improvements Section, Sales Comparison Approach, Reconciliation, Appraiser Certification.

```python
class ReportGenerator:
    async def generate_urar_pdf(self, appraisal_id: UUID) -> bytes:
        """
        1. Load appraisal with subject, comps, adjustments, narrative sections
        2. Validate all UAD 3.6 required fields populated
        3. Render HTML template with Jinja2
        4. Convert to PDF via WeasyPrint
        5. Store PDF in S3, update report record with URL
        """
```

**Testing**:
- Unit: appraisal with all required fields → valid PDF generated
- Unit: missing required UAD field → validation error listing missing fields
- Integration: generate PDF → readable, contains subject address, comps, adjustments, narrative
- Fixture: `tests/fixtures/sample_urar.json` → expected PDF structure

#### 3.2 — MISMO v3.6 XML Export

**What**: Generate MISMO v3.6 XML for submission to UCDP (GSE portal).

**Design**:

```python
class MISMOXMLBuilder:
    def build(self, appraisal: Appraisal) -> etree.Element:
        """Build MISMO v3.6 XML document:
        - MESSAGE > DEAL_SETS > DEAL_SET > DEALS > DEAL
        - DEAL > COLLATERALS > COLLATERAL > SUBJECT_PROPERTY
        - DEAL > SERVICES > SERVICE > APPRAISAL
        - APPRAISAL > APPRAISED_VALUE, COMPARABLE_SALES, ADJUSTMENTS
        Validates against MISMO v3.6 XSD before returning.
        """

    def validate(self, xml: etree.Element) -> list[str]:
        """Validate against MISMO XSD. Return list of errors (empty if valid)."""
```

**Testing**:
- Unit: complete appraisal → valid MISMO XML (passes XSD validation)
- Unit: XML contains correct UAD 3.6 field values for all comps and adjustments
- Unit: missing required MISMO element → validation error
- Fixture: `tests/fixtures/sample_mismo_v36.xml` → golden reference

---

## Phase 4: Mobile Field Inspection App

### Purpose
Mobile app for on-site property inspection with photo capture, property measurement, and offline capability.

### Tasks

#### 4.1 — Inspection Photo Capture and Upload

**What**: React Native app for capturing inspection photos tagged by room/area; syncs to appraisal.

**Design**:

Screens: Inspection Checklist (exterior front, sides, rear, street, interior rooms), Camera View (capture with tag), Photo Review (crop, annotate), Offline Queue.

```typescript
interface InspectionPhoto {
  appraisalId: string;
  category: "exterior_front" | "exterior_rear" | "exterior_left" | "exterior_right"
           | "street_scene" | "interior_kitchen" | "interior_living" | "interior_bedroom"
           | "interior_bathroom" | "interior_other" | "deficiency" | "improvement";
  photoUri: string;          // local file URI
  notes: string;
  capturedAt: string;
  gpsLatitude: number;
  gpsLongitude: number;
  synced: boolean;
}
```

Offline: photos stored locally; background upload when connectivity returns.

**Testing**:
- E2E: take 3 photos → tagged and visible in local gallery
- E2E: offline → photos queue locally → reconnect → uploaded to server
- Unit: GPS coordinates captured with each photo

#### 4.2 — Property Sketch Tool

**What**: Simple drawing tool for creating property floor plan sketches with room dimensions.

**Design**:

Canvas-based sketch tool: draw rooms as rectangles, enter width/length, label rooms. Auto-computes GLA from room dimensions. Export as image attached to appraisal. SVG-based for vector quality.

**Testing**:
- E2E: draw 3 rooms with dimensions → GLA computed = sum of room areas
- E2E: export sketch → attached to appraisal as inspection photo
- Unit: room 15×20 → area = 300 sqft

---

## Phase 5: AI-Powered Comparable Selection

### Purpose
Use embedding-based similarity search to automatically rank and suggest the best comparable sales. This is the key AI differentiator over incumbent manual comp browsing.

### Tasks

#### 5.1 — Property Embedding Generation

**What**: Generate vector embeddings from property characteristics for similarity search.

**Design**:

```python
class PropertyEmbedder:
    def embed(self, property: Property) -> list[float]:
        """Create embedding from structured property features:
        - Normalised: GLA, lot_size, bedrooms, bathrooms, year_built, stories
        - Categorical encoded: property_type, condition, quality, location_rating
        - Geographic: lat/lng
        Uses sentence-transformers to encode a text representation of the property
        into a 384-dimensional vector, stored in pgvector column.
        """

    PROPERTY_TEXT_TEMPLATE = """
    {property_type} property at {address}, {city}, {state}.
    {bedrooms} bedrooms, {bathrooms} bathrooms, {gla} sqft on {lot_size} sqft lot.
    Built {year_built}. Condition: {condition}. Quality: {quality}.
    """
```

pgvector column on `comparable_sale` table; indexed with IVFFlat.

**Testing**:
- Unit: two similar properties → cosine similarity > 0.8
- Unit: dissimilar properties (condo vs. ranch on acreage) → similarity < 0.5
- Integration: embed 100 properties → vector search returns ranked results

#### 5.2 — AI Comp Ranking and Suggestion

**What**: Rank available comps by similarity to subject and suggest top 5 with explanations.

**Design**:

```python
class CompSelector:
    async def suggest_comps(self, appraisal_id: UUID, max_results: int = 5) -> list[CompSuggestion]:
        """
        1. Embed subject property
        2. Vector similarity search in pgvector for sales within time/distance window
        3. Re-rank by weighted factors: similarity score (40%), proximity (20%),
           sale recency (20%), GLA similarity (10%), price per sqft (10%)
        4. Return top max_results with explanation of why each was selected
        """

@dataclass
class CompSuggestion:
    comparable_sale_id: UUID
    similarity_score: float
    rank_explanation: str       # "Selected because: similar GLA (1,820 vs 1,800), same neighbourhood, sold 45 days ago"
    distance_miles: float
    days_since_sale: int
```

**Testing**:
- Unit: subject 1,800 sqft ranch → top suggestion is closest 1,750-1,850 sqft ranch
- Unit: 0 comps within 1 mile → expand to 3 miles automatically
- Integration: suggest comps → top 5 returned with explanations
- E2E: click "AI Suggest Comps" → comps appear on map with rank badges

---

## Phase 6: Narrative Report Drafting

### Purpose
Use Claude to draft USPAP-compliant narrative sections from structured appraisal data, saving appraisers hours of writing time.

### Tasks

#### 6.1 — AI Narrative Generation

**What**: Draft market analysis, neighbourhood description, highest-and-best-use analysis, and reconciliation sections.

**Design**:

```python
class NarrativeDrafter:
    SYSTEM_PROMPT = """You are a licensed real estate appraiser drafting USPAP-compliant
    narrative sections for a residential appraisal report. Write in third person, factual,
    professional tone. Reference specific data points (comparable sales, market trends,
    property characteristics). Each section must support the final value conclusion.
    Do not make value judgments not supported by the data provided."""

    async def draft_section(self, appraisal: Appraisal, section: str) -> str:
        """Draft a narrative section: 'market_analysis', 'neighbourhood_description',
        'highest_and_best_use', 'reconciliation', 'scope_of_work'.
        Provides: subject property data, selected comps with adjustments,
        market trend data, neighbourhood demographics."""
```

Appraiser reviews, edits, and approves each drafted section before finalisation.

**Testing**:
- Unit (mocked): market analysis → references specific comp sales and price trends
- Unit (mocked): reconciliation → explains weight given to each comp
- Unit (mocked): USPAP compliance → sections do not contain speculative language
- Integration: draft all 5 sections → appraiser reviews in report editor

---

## Phase 7: AVM Integration and Bias Auditing

### Purpose
Integrate automated valuation model alongside appraiser estimate, and audit valuations for bias. Addresses CFPB AVM Rule and regulatory fairness requirements.

### Tasks

#### 7.1 — AVM Engine with Confidence Scoring

**What**: Automated valuation using comp-based regression with confidence intervals.

**Design**:

```python
class AVMEngine:
    def estimate(self, property: Property, comps: list[ComparableSale]) -> AVMResult:
        """
        1. Weighted average of adjusted comp prices (distance and recency weighted)
        2. Hedonic regression on property characteristics for cross-validation
        3. Confidence interval: spread of adjusted comp prices
        4. Store result with model_version and methodology for CFPB compliance
        """

@dataclass
class AVMResult:
    estimated_value_cents: int
    confidence_lower_cents: int
    confidence_upper_cents: int
    confidence_score: float         # 0-1
    methodology: str                # "weighted_comp_average" or "hedonic_regression"
    model_version: str
    comp_ids_used: list[UUID]
```

AVM displayed alongside appraiser's value conclusion for comparison; not a replacement.

**Testing**:
- Unit: 5 comps with adjusted prices $300k-$320k → AVM ≈ $310k, CI = $298k-$322k
- Unit: high comp spread → lower confidence score
- Unit: methodology and model_version stored for CFPB audit

#### 7.2 — Bias Detection Audit

**What**: Audit valuations for demographic and geographic bias patterns per HUD 2024 AI guidance.

**Design**:

```python
class BiasAuditor:
    async def audit(self, appraisal: Appraisal) -> BiasAuditResult:
        """
        1. Compare appraised value to AVM estimate — flag if appraiser value
           is significantly below AVM in majority-minority census tracts
        2. Compare adjustment patterns: are negative adjustments disproportionately
           applied to comps in certain demographic areas?
        3. Check comp selection geographic diversity — did appraiser skip
           closer comps from a different demographic area?
        4. Return audit_score (0-100, higher = more concern), findings list
        """

@dataclass
class BiasAuditResult:
    audit_score: int                # 0-100
    risk_level: str                 # low, medium, high
    findings: list[BiasAuditFinding]
    recommendations: list[str]
```

Audit result stored on appraisal; visible to reviewer. Not a blocking gate — informational for reviewer decision.

**Testing**:
- Unit: value matches AVM, diverse comp selection → audit_score < 20
- Unit: value 15% below AVM in minority tract → finding generated
- Unit: closer comp skipped in favour of farther comp from different area → finding generated

---

## Phase 8: Commercial Appraisal Support (v1.1)

### Purpose
Extend to commercial appraisals with DCF valuation, rent roll analysis, and cap-rate derivation.

### Tasks

#### 8.1 — Income Approach and DCF Valuation

**What**: DCF worksheet with rent roll ingestion, expense modelling, cap-rate selection, and scenario analysis.

**Design**:

```python
@dataclass
class DCFInput:
    rent_roll: list[LeaseEntry]     # tenant, sqft, rate_per_sqft, start, end, escalations
    operating_expenses_cents: int
    vacancy_rate: float
    cap_rate: float
    discount_rate: float
    holding_period_years: int

@dataclass
class DCFResult:
    net_operating_income_cents: int
    value_by_direct_cap_cents: int
    value_by_dcf_cents: int
    irr: float
    cash_flow_schedule: list[AnnualCashFlow]
```

**Testing**:
- Unit: 10-unit building with known rents → NOI computed correctly
- Unit: cap rate 6% on $500k NOI → value = $8.33M
- Unit: DCF with 10-year hold → present value matches hand calculation

#### 8.2 — Lease Abstraction via OCR + NLP

**What**: Extract lease terms from PDF lease documents using OCR and Claude.

**Design**:

Upload PDF → OCR (pdf2image + Tesseract) → Claude extracts: tenant name, suite/unit, sqft, base rent, escalation schedule, lease start/end, options, CAM charges. Outputs structured `LeaseEntry` for rent roll.

**Testing**:
- Unit (mocked): sample lease PDF → correct tenant, rent, dates extracted
- Integration: upload lease → structured data populates rent roll
- Unit: unclear terms → Claude flags for manual review

---

## Phase Summary & Dependencies

```
Phase 1: Foundation                    ─── required by everything
    │
Phase 2: Comparable Sales & Selection  ─── requires Phase 1
    │
Phase 3: Report Generation & MISMO     ─── requires Phase 2
    │
Phase 4: Mobile Inspection App         ─── requires Phase 1 (parallel with Phases 2-3)
    │
Phase 5: AI Comp Selection             ─── requires Phase 2
    │
Phase 6: Narrative Drafting            ─── requires Phase 3
    │
Phase 7: AVM & Bias Auditing          ─── requires Phase 2
    │
Phase 8: Commercial Support            ─── requires Phase 3
```

Parallelism opportunities:
- Phase 4 (mobile) can be developed concurrently with Phases 2-3
- Phases 5, 6, 7 can be developed concurrently after their dependencies
- Phase 8 can be developed concurrently with Phases 5-7

---

## Definition of Done (per phase)

1. All tasks implemented with code matching the design specification.
2. All unit and integration tests pass (`pytest` + `vitest`).
3. Ruff linting passes (Python); Biome passes (TypeScript).
4. mypy strict passes (Python); tsc --noEmit passes (TypeScript).
5. Docker build succeeds.
6. `docker-compose up` brings all services to healthy state.
7. Feature works end-to-end.
8. UAD 3.6 field validation passes for all appraisal data (where applicable).
9. MISMO XML validates against XSD schema (Phase 3+).
10. New API endpoints appear in auto-generated OpenAPI spec.
11. Database migrations created and tested.
12. New environment variables documented in config.py.
