# Standards & API Reference

> Project: Real Estate Appraisal Tool · Generated: 2026-05-03

## Industry Standards & Specifications

### US Regulatory & Compliance Standards

**USPAP — Uniform Standards of Professional Appraisal Practice (2024 Edition)**
- Publisher: The Appraisal Foundation (Appraisal Standards Board)
- URL: https://appraisalfoundation.org/pages/uspap
- The national mandatory ethical and performance standard for licensed appraisers in the US. The 2024 Edition is open-ended (no fixed expiry); appraisers must complete a 7-Hour USPAP Update course every two years. In April 2026, Advisory Opinion 41 (AO-41) was adopted, providing guidance on USPAP compliance when using technological tools including AVMs, regression software, and generative AI. Any appraisal platform used by licensed appraisers must support USPAP-compliant report structures and audit trails.

**FIRREA — Financial Institutions Reform, Recovery, and Enforcement Act (Title XI)**
- Publisher: US Federal Government; enforced via OCC, FDIC, Federal Reserve, NCUA, CFPB
- URL: https://www.ecfr.gov/current/title-12/chapter-III/subchapter-B/part-323
- Federal law requiring appraisals for federally related transactions to be performed in writing, by state-certified or state-licensed appraisers, in accordance with USPAP. Title XI mandates appraiser independence from lending, investment, and collection functions. Platforms used by AMCs and lenders must enforce independence rules and maintain documented appraiser qualifications.

**CFPB AVM Rule — Automated Valuation Models Rule (Effective July 1, 2025)**
- Publisher: CFPB, Federal Housing Finance Agency, FDIC, Federal Reserve, NCUA, OCC
- URL: https://www.consumerfinance.gov/about-us/blog/cfpb-approves-rule-to-ensure-accuracy-and-accountability-in-the-use-of-ai-and-algorithms-in-home-appraisals/
- Federal rule requiring businesses using algorithmic appraisal tools to ensure high confidence in estimates, protect against data manipulation, avoid conflicts of interest, conduct random sample testing, and comply with nondiscrimination laws. Directly governs any AI/AVM engine embedded in mortgage lending workflows. Non-compliance risk is significant for platforms used by lenders.

### GSE Data Standards

**UAD 3.6 — Uniform Appraisal Dataset, Version 3.6**
- Publisher: Fannie Mae and Freddie Mac (GSEs); aligned to MISMO Reference Model 3.6
- URL (Fannie Mae): https://singlefamily.fanniemae.com/delivering/uniform-mortgage-data-program/uniform-appraisal-dataset
- URL (Freddie Mac): https://sf.freddiemac.com/tools-learning/uniform-mortgage-data-program/uad
- The mandatory data schema for all residential appraisals submitted to the GSEs from November 2, 2026. UAD 3.6 replaces all legacy URAR forms (1004, 1073, 1025, 2055, etc.) with a single redesigned Uniform Residential Appraisal Report (URAR) built on structured, machine-readable data rather than free-text narratives. The transition from UAD 2.6 to UAD 3.6 runs January–November 2026. All residential appraisal platforms must implement UAD 3.6 compliance by the mandate date.

**UCDP — Uniform Collateral Data Portal**
- Publisher: Fannie Mae and Freddie Mac
- URL (Fannie Mae): https://singlefamily.fanniemae.com/applications-technology/uniform-collateral-data-portal
- URL (Freddie Mac): https://sf.freddiemac.com/tools-learning/uniform-mortgage-data-program/ucdp
- The joint GSE portal through which lenders must submit appraisal data. Supports both a web UI and Direct Integration (DI) via vendor-provided APIs. Platforms targeting the lending market must support UCDP direct integration for MISMO XML delivery. The Limited Production Period for UAD 3.6 UCDP enhancements is currently open.

### Mortgage & Data Exchange Standards

**MISMO Reference Model v3.6**
- Publisher: Mortgage Industry Standards Maintenance Organization (MISMO)
- URL: https://www.mismo.org/standards-resources/residential-specifications/reference-model
- URL (XML Schema): https://www.mismo.org/standards-resources/residential-specifications/reference-model/xml-schema
- URL (Datasets): https://www.mismo.org/standards-resources/residential-specifications/datasets
- URL (Appraisal Procurement Dataset): https://www.mismo.org/standards-resources/mismo-product/appraisal-procurement-dataset-specification
- The XML data model standard for mortgage-related appraisal data exchange between lenders, AMCs, appraisers, and GSEs. Version 3.6 (Candidate Recommendation) underpins UAD 3.6. Supports appraisal procurement, delivery, and review workflows. An updated Appraisal Procurement Dataset Specification was open for public comment through April 23, 2026. Platforms must implement MISMO v3.6 XML for GSE submission and lender integrations.

**RESO Web API — Real Estate Standards Organization Web API**
- Publisher: Real Estate Standards Organization (RESO)
- URL: https://www.reso.org/reso-web-api/
- The industry standard for MLS data integration, built on REST architecture and OData V4 for query standardisation. The RESO Data Dictionary defines standard field names, accepted values, and relationships across MLS systems. Platforms requiring access to MLS listing and comparable sales data should implement RESO Web API integration to achieve broad MLS compatibility rather than building per-MLS integrations.

### International Valuation Standards

**IVS — International Valuation Standards (Effective 31 January 2025)**
- Publisher: International Valuation Standards Council (IVSC)
- URL: https://ivsc.org/standards/
- URL (Full text): https://saicawebprstorage.blob.core.windows.net/uploads/resources/IVS-effective-31-January-2025.pdf
- The global valuation standard comprising seven General Standards and eight Asset-specific Standards. The 2025 edition (published January 2024, effective January 2025) adds new guidance on technology in valuations, ESG factors, revised definitions of fair value and market value, and updates to real estate valuation. Required for IVS-compliant valuations in international markets and adopted by RICS into its Red Book.

**RICS Valuation — Global Standards (Red Book, December 2024)**
- Publisher: Royal Institution of Chartered Surveyors (RICS)
- URL: https://www.rics.org/profession-standards/rics-standards-and-guidance/sector-standards/valuation-standards/red-book/red-book-global
- URL (PDF): https://www.rics.org/content/dam/ricsglobal/documents/standards/Red-Book-Global-Standards-incorporating-IVS.pdf
- The mandatory valuation standard for RICS members globally, incorporating the 2025 IVS. The December 2024 edition (effective 31 January 2025) mandates ESG factor consideration at every valuation stage and includes new material on valuation modelling and methods. Platforms targeting UK, European, and international markets must support RICS Red Book-compliant reporting structures.

### ISO Geographic & Property Data Standards

**ISO 19152-1:2024 — Land Administration Domain Model (LADM), Part 1: Generic Conceptual Model**
- Publisher: International Organization for Standardization (ISO/TC 211)
- URL: https://www.iso.org/standard/81263.html
- Defines the abstract, conceptual data model for land administration covering parties (people and organisations), administrative units, rights/responsibilities/restrictions, and spatial representations. Provides a standardised global vocabulary for property data that can inform the data model design of an appraisal platform's property record schema.

**ISO 19152-4:2025 — Land Administration Domain Model (LADM), Part 4: Valuation Information**
- Publisher: International Organization for Standardization (ISO/TC 211)
- URL: https://www.iso.org/standard/81266.html
- The specific LADM part covering property valuation information, published in 2025. Defines standardised data structures for representing property values, valuation methodologies, and valuation history within land administration systems. Particularly relevant for tax assessment and portfolio valuation use cases.

---

## Similar Products — Developer Documentation & APIs

### HouseCanary

- **Description:** AI-powered residential AVM platform covering 114 million US properties, providing machine-learning valuations, image recognition-based condition assessments, market analytics, and comparable data. Designed for API-first embedded use in lending and investment platforms.
- **API Documentation:** https://api-docs.housecanary.com/ (primary docs) and https://api-docs-legacy.housecanary.com/ (legacy)
- **Developer Tools & Quick Start:** https://www.housecanary.com/resources/developer-tools
- **Product Pages:** https://www.housecanary.com/products/data-explorer (Data Explorer API)
- **SDKs/Libraries:** Postman collection with examples in 20+ languages; REST endpoints at `https://api.housecanary.com/v2/property/value`
- **Standards:** REST/JSON; test credentials available via settings page
- **Authentication:** API Key (test and production keys)

### Clear Capital

- **Description:** Residential valuation technology company providing AVMs, hybrid appraisals, property analytics, and appraisal risk review tools for lenders. Offers a full API suite for embedding valuations and risk assessment into loan origination workflows.
- **API Documentation:** https://docs.api.clearcapital.com/
- **Product Valuation API:** https://www.clearcapital.com/products/property-valuation-api/
- **Property Analytics API:** https://www.clearcapital.com/products/property-analytics-api/
- **AURA Risk API:** https://www.clearcapital.com/products/aura/ (appraisal underwriting risk analyser)
- **SDKs/Libraries:** RESTful, multitenancy API; language-specific examples in developer portal
- **Standards:** REST/JSON; OpenAPI-compatible
- **Authentication:** OAuth 2.0 / API Key (documented in developer portal)

### Reggora

- **Description:** Modern appraisal order management platform for lenders and AMCs, providing automated appraisal ordering, vendor panel management, quality control, and MISMO XML delivery. Supports 100% order management within lenders' proprietary LOS systems.
- **API Documentation (Lender API):** https://api.reggora.io/
- **Developer Portal:** https://developer.reggora.io/
- **Standards:** REST/JSON; MISMO XML delivery to UCDP
- **Authentication:** API Key with secure endpoints; prebuilt API wrappers for multiple languages
- **Notes:** Supports iFrame embedding and custom point-of-sale system integration

### ARGUS Enterprise (Altus Group)

- **Description:** Industry-standard commercial real estate DCF valuation and cash flow modelling platform for institutional investors, funds, and commercial appraisers. Supports multi-method valuation, lease-by-lease modelling, sensitivity analysis, and 40+ report types.
- **API Documentation:** https://www.altusgroup.com/argus/products/integration-solutions
- **Downloads & Guides:** https://www.altusgroup.com/argus/downloads/argus-enterprise/
- **Standards:** Cloud-based REST API; Excel integration for data exchange
- **Authentication:** Enterprise OAuth / SSO (contact Altus Group for API access)
- **Notes:** ARGUS API allows extraction and ingestion of data into ARGUS models and databases without opening the application; supports portfolio consolidation APIs and automated recalculation workflows.

### Valcre

- **Description:** Cloud-based commercial appraisal platform with template-based valuation, integrated comps database, report generation, and team collaboration tools. Uses Cherre's GraphQL API for property data enrichment and integrates with Rockport VAL for DCF modelling.
- **Product URL:** https://valcre.com/
- **Enterprise & API:** https://www.valcre.com/enterprise/ (open API for CRM/database connections)
- **Standards:** REST API; GraphQL (via Cherre integration); DCF via Rockport VAL integration
- **Authentication:** API Key / OAuth (details available to enterprise clients)
- **Notes:** Integrates with 15+ nationwide data providers; Rockport VAL integration enables seamless DCF calculation and appraisal report generation.

### Rockport VAL

- **Description:** Cloud-based commercial real estate DCF valuation and appraisal report generation software for lenders, investors, and commercial appraisers. API-first architecture enables seamless data flow into and out of the platform from third-party appraisal platforms.
- **Product URL:** https://www.therockportgroup.com/products/rockport-val
- **Standards:** REST API; integrates with Valcre and Realquantum via API
- **Authentication:** Contact Rockport VAL for developer access
- **Notes:** API enables flow of lease and property data into DCF models and extraction of valuation results into appraisal reports. Known integrations with Valcre and Realquantum.

### Cherre

- **Description:** Real estate data management and analytics platform that ingests disparate property data sources into a centralised knowledge graph, exposing unified access via a single GraphQL API. Used by institutional investors and commercial appraisal platforms for data enrichment.
- **Product URL:** https://cherre.com/products/platform/
- **Standards:** GraphQL API (powered by Hasura/PostGIS on Google Cloud SQL + BigQuery)
- **Authentication:** Enterprise access (contact Cherre for API credentials)
- **Notes:** Cherre chose GraphQL over REST for its ability to handle complex geospatial queries and cross-source joins. Integrated into Valcre for property and building-level data enrichment.

### ACI (First American)

- **Description:** Residential appraisal forms platform with 500+ USPAP/UAD-compliant forms, cloud-based ACI Sky Workbench, mobile inspection (SureStep), and access to First American's 100% US housing stock data. Provides MISMO XML delivery for UCDP submission.
- **Product URL:** https://www.aciweb.com/
- **Downloads / Integration:** https://www.aciweb.com/downloads/
- **Standards:** MISMO XML (UCDP delivery); UAD 2.6 and 3.6 compliant forms; REST integration API for AMC/lender portals
- **Authentication:** API Key / portal credentials (contact ACI support at support@aciweb.com)
- **Notes:** Express MISMO XML delivery is a core product feature; integration API for direct connection to appraisal management systems documented internally. Transitioning from legacy desktop to ACI Sky cloud platform.

---

## Notes

**UAD 3.6 Transition (2026):** The November 2, 2026 mandate for UAD 3.6 compliance is the most significant near-term standards event in this space. Both legacy UAD 2.6 and the new UAD 3.6 will be accepted from January 26 to November 2, 2026. Any new platform must be UAD 3.6 native from launch to avoid costly retrofitting.

**CFPB AVM Rule (July 2025):** The AVM Rule effective July 1, 2025 creates material compliance obligations for any AI-native valuation product used in mortgage lending workflows. Platforms must implement policies for data quality, conflict of interest controls, random sample testing, and nondiscrimination monitoring before entering the lender market.

**USPAP AO-41 (April 2026):** The newly adopted Advisory Opinion 41 on technology use (including generative AI) is not yet widely reflected in existing software. This is an emerging area where an AI-native platform can differentiate by building compliance tooling for AO-41 documentation requirements.

**RESO Web API vs Legacy RETS:** The Real Estate Transaction Standards (RETS) protocol is being phased out in favour of the RESO Web API (OData v4). New platforms should implement RESO Web API rather than RETS for MLS data access to ensure forward compatibility.

**CoStar Data Access:** CoStar does not provide a publicly accessible developer API. Commercial data access requires a commercial licensing agreement and is not available on a per-query or self-service basis. An AI-native platform targeting the commercial appraisal segment should plan to integrate with more accessible data providers (Cherre, Reonomy, CompStak) for comparable sales and market data.
