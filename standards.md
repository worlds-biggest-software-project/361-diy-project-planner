# Standards & API Reference

> Project: DIY Project Planner · Generated: 2026-05-04

## Industry Standards & Specifications

### ISO Standards

**ISO 19650-1:2018 — Information management using building information modelling (BIM): Concepts and principles**
- URL: https://www.iso.org/standard/68078.html
- Defines concepts and principles for information management across the life cycle of a built asset using BIM. Directly relevant to how project data (tasks, materials, phases) should be structured for interoperability with professional tools.

**ISO 23387:2020 — BIM data templates for construction objects**
- URL: https://www.iso.org/standard/75403.html
- Specifies data templates for construction objects (e.g., doors, windows, fixtures, materials) used across the life cycle of built assets. Informs how a bill-of-materials data model should be structured for DIY projects so that outputs are compatible with professional BIM workflows.

**ISO 16739-1:2024 — Industry Foundation Classes (IFC) for data sharing in the construction and facility management industries**
- URL: https://www.iso.org/standard/84123.html
- IFC is the open neutral data format for construction objects, published by buildingSMART International and ratified as an ISO standard. Adopting IFC-compatible data models for rooms, elements, and materials would allow export to professional design tools. The data schema can be serialised as STEP (.ifc), XML, or JSON.

**ISO/IEC 40500:2025 — Information technology: W3C Web Content Accessibility Guidelines (WCAG) 2.2**
- URL: https://www.iso.org/standard/58625.html (WCAG 2.2 ISO ratification)
- WCAG 2.2 is now an ISO standard. Applies to both web and mobile apps. Nine new success criteria for mobile (touch targets, gestures, responsive design) are relevant for a DIY planner app targeting non-technical homeowners across diverse accessibility needs.

---

### W3C & IETF Standards

**WCAG 2.2 — Web Content Accessibility Guidelines 2.2 (W3C Recommendation)**
- URL: https://www.w3.org/TR/WCAG22/
- Core accessibility standard for web and (via WCAG2Mobile guidance) mobile applications. Success criteria for touch targets, contrast, keyboard accessibility, and cognitive load are directly applicable to a consumer-facing mobile-first DIY tool.

**WCAG2Mobile — Guidance on Applying WCAG 2.2 to Mobile Applications**
- URL: https://www.w3.org/TR/wcag2mobile-22/
- W3C Group Note providing specific guidance on applying WCAG 2.2 success criteria to native iOS, Android, and hybrid mobile apps. Covers gestures, touch target sizing, and responsive layout — all critical for an on-site DIY app used in variable conditions.

**RFC 8252 — OAuth 2.0 for Native Apps (IETF Best Current Practice)**
- URL: https://datatracker.ietf.org/doc/rfc8252/
- Defines security requirements for native and mobile application OAuth 2.0 flows, including PKCE and the prohibition on embedded web views. Mandatory guidance for any authentication implementation in the iOS/Android DIY planner app.

**RFC 7636 — Proof Key for Code Exchange (PKCE) by OAuth Public Clients**
- URL: https://datatracker.ietf.org/doc/html/rfc7636
- PKCE prevents authorisation code interception attacks in mobile apps. Now considered essential for all public OAuth 2.0 clients (mobile and SPA). Required alongside RFC 8252 for any user authentication flow.

**RFC 7617 — The 'Basic' HTTP Authentication Scheme**
- URL: https://datatracker.ietf.org/doc/html/rfc7617
- Referenced for completeness; Basic auth is not recommended for mobile consumer apps — OAuth 2.0 + PKCE (above) is the correct standard.

**RFC 9110 — HTTP Semantics (successor to RFC 7231)**
- URL: https://datatracker.ietf.org/doc/html/rfc9110
- The current IETF standard for HTTP request/response semantics. Underpins any REST API layer the DIY planner exposes or consumes. Relevant to correct use of status codes, headers, and method semantics.

---

### Data Model & API Specifications

**OpenAPI Specification 3.1.1**
- URL: https://spec.openapis.org/oas/v3.1.1.html
- The industry standard for describing REST APIs. OpenAPI 3.1 is a superset of JSON Schema Draft 2020-12, giving full schema validation compatibility. Any public or partner-facing API for the DIY planner (e.g., contractor portal, retailer integration) should be described using OAS 3.1.

**JSON Schema (Draft 2020-12)**
- URL: https://json-schema.org/draft/2020-12/
- The canonical standard for describing and validating JSON data structures. Use for defining and validating the DIY planner's core data objects: Project, Phase, Task, MaterialItem, ContractorQuote, Expense, Photo.

**Industry Foundation Classes (IFC) — buildingSMART**
- URL: https://technical.buildingsmart.org/standards/ifc/
- The open BIM data standard (ISO 16739-1) serialisable as JSON, XML, or STEP. For the subset of users doing room-level planning, adopting IFC-compatible room and element object schemas enables interoperability with Magicplan, Revit, and other professional tools.

**National BIM Standard-United States (NBIMS-US v4)**
- URL: https://nibs.org/nbims/v4/
- The US domestic adaptation of ISO 19650. Defines level-of-detail requirements and data exchange protocols for construction information. Relevant if the DIY planner targets the US market and wishes to enable data hand-off to professional contractors.

---

### Security & Authentication Standards

**OpenID Connect Core 1.0 (OIDC)**
- URL: https://openid.net/specs/openid-connect-core-1_0.html
- Authentication layer built on OAuth 2.0. Provides a standard identity token (ID Token as JWT) for user authentication. Recommended for the DIY planner's sign-in flow to support social login (Google, Apple) and shared household accounts.

**OAuth 2.0 (RFC 6749 + RFC 6750)**
- URL: https://oauth.net/2/
- The authorisation framework underlying all modern consumer authentication. Required for integrations with home improvement retailers (Home Depot/Lowe's if APIs become available) and contractor directories.

**AppAuth SDK (iOS, Android, JS)**
- URL: https://appauth.io/
- Open-source SDK implementing OAuth 2.0 + OIDC best practices for native mobile apps per RFC 8252. The recommended implementation library for authentication in the iOS and Android DIY planner clients.

**GDPR (EU General Data Protection Regulation)**
- URL: https://gdpr-info.eu/
- Applies to any EU users. Requires opt-in consent, data access/deletion mechanisms, and privacy-by-design in data collection. For a DIY planner storing home address, renovation details, and photos: data minimisation and right-to-erasure must be designed in from the start.

**CCPA/CPRA (California Consumer Privacy Act / California Privacy Rights Act)**
- URL: https://oag.ca.gov/privacy/ccpa
- Applies to California users where annual gross revenue exceeds $25M or data on 100,000+ consumers is processed. Requires opt-out of data sharing/sale and accessible "Do Not Sell or Share" controls within the app.

**OWASP Mobile Application Security Verification Standard (MASVS) v2.0**
- URL: https://mas.owasp.org/MASVS/
- The definitive security standard for mobile apps. Defines security controls for authentication, network communication, data storage, code quality, and tamper resistance. Compliance with MASVS-L1 (basic) is recommended; L2 for any feature involving payment or contractor financial data.

---

### Offline / Local-First Data Standards

**CRDT (Conflict-free Replicated Data Types)**
- Reference implementation: https://github.com/sqliteai/sqlite-sync
- Mathematical data structures that allow distributed replicas (multiple devices, multiple users) to be edited independently and merged without conflicts. Directly relevant to the offline-first design requirement: a DIY planner used on-site (phone) and at home (tablet/web) needs CRDT-based sync to avoid data loss or manual conflict resolution.

**SQLite (embedded relational database)**
- URL: https://www.sqlite.org/
- The most widely deployed database engine in the world; the de-facto standard for local-first mobile app storage on iOS (via GRDB/SQLite.swift) and Android (via Room). Pairs with CRDT-based sync layers (ElectricSQL, SQLite Cloud) for cloud sync.

---

## Similar Products — Developer Documentation & APIs

### Procore

- **Description:** Leading professional construction project management platform used by general contractors and developers. Covers budgets, scheduling, RFIs, submittals, documents, and client communication.
- **API Documentation:** https://developers.procore.com/reference/rest/docs/rest-api-overview
- **Developer Hub:** https://developers.procore.com/documentation/introduction
- **Authentication:** OAuth 2.0 (https://developers.procore.com/documentation/oauth-endpoints)
- **Standards:** REST/JSON; versioned resource-level API; Postman collection available
- **SDKs/Libraries:** No official SDK listed; REST-only with Postman collection
- **Notes:** Procore's API covers budgets, cost codes, schedule items, RFIs, and documents — all resource types relevant to a DIY planner's professional-grade feature set. Good reference for data modelling.

---

### Buildertrend

- **Description:** All-in-one construction management SaaS for remodelers, home builders, and specialty trades; covers scheduling, estimating, job costing, client portal, and subcontractor management.
- **API Documentation:** Available via Buildertrend Marketplace integrations; no public API portal URL confirmed
- **Integrations:** Zapier (https://zapier.com/apps/buildertrend/integrations), QuickBooks, Xero, Salesforce, HubSpot, Gusto
- **Standards:** REST/JSON; Zapier trigger/action model for no-code integration
- **Authentication:** OAuth 2.0 (inferred from Zapier/marketplace integration pattern)
- **Notes:** Buildertrend's integration ecosystem (QuickBooks for accounting, Salesforce for CRM) illustrates the integration targets a mature DIY planner would eventually need: accounting export, CRM for contractor relationships, and payroll for hired labour tracking.

---

### HomeZada

- **Description:** Consumer-facing home management platform covering home improvement project planning, maintenance scheduling, home inventory, and financial tracking.
- **API Documentation:** No public API documented; proprietary SaaS
- **SDKs/Libraries:** None public
- **Standards:** Not applicable (no public API)
- **Authentication:** Email/password; social login
- **Notes:** The closest direct competitor in the consumer DIY planner space. Its feature set (AI material lists, budget tracking, photo storage, ROI estimation) is the primary benchmark for MVP scope. No API means no integration risk.

---

### Remodelum

- **Description:** Consumer renovation budget tracker and cost estimator with zip-code-accurate local construction cost data, expense tracking, contractor quote comparison, and collaborative sharing.
- **API Documentation:** No public API documented
- **SDKs/Libraries:** None public
- **Standards:** Not applicable (no public API)
- **Authentication:** Email/password (implied)
- **Notes:** Zip-code-accurate cost estimation is a differentiating data feature. The underlying cost benchmark data is likely sourced from RSMeans or similar construction cost databases.

---

### Magicplan

- **Description:** Professional mobile floor-plan scanning and estimating tool using LiDAR, AR, and Bluetooth laser integration; exports to Xactimate and Cotality for insurance/restoration workflows.
- **API Documentation:** https://help.magicplan.app/ (help centre; no public REST API documented)
- **Export Formats:** PDF, DXF (CAD), PNG, ESX (Xactimate), FML (Cotality)
- **Standards:** Xactimate ESX format; Cotality FML; IFC-compatible geometry concepts
- **Authentication:** Account-based (no public OAuth integration documented)
- **Notes:** Magicplan's LiDAR-to-floor-plan pipeline is a compelling integration target for the DIY planner: if room dimensions are known, material quantities (paint area, tile count, flooring area) can be auto-calculated. No official integration API, but export to standard DXF is possible.

---

### Houzz Pro

- **Description:** Professional construction management companion to consumer Houzz; covers client communication, project timelines, invoicing, and selection management for interior designers and contractors.
- **API Documentation:** Integration help: https://pro.houzz.com/pro-help/f/integrations; Seller API: sellerapi@houzz.com
- **Integrations:** Zapier (https://zapier.com/apps/houzz-pro/integrations), Google Drive, Gmail
- **Standards:** REST/JSON (implied via Zapier); no public OpenAPI spec
- **Authentication:** OAuth 2.0 (via Zapier)
- **Notes:** Houzz's dual homeowner/professional model (consumer Houzz + Houzz Pro) is an architectural pattern worth noting: homeowner creates project inspiration, contractor picks up in Pro. A DIY planner could adopt a similar handover model — homeowner plans in app, shares structured data with a hired contractor.

---

### Home Depot Product Data (Third-Party APIs)

- **Description:** Home Depot does not offer a first-party public product API. Third-party data providers offer structured access to Home Depot product listings, pricing, availability, and reviews.
- **SerpApi Home Depot Product API:** https://serpapi.com/home-depot-product — returns title, brand, price, description, UPC, model number, specs, rating, reviews
- **Traject Data BigBox API:** https://trajectdata.com/ecommerce/big-box-api/ — unified Home Depot + Lowe's product, pricing, and availability data
- **Unwrangle Home Depot API:** https://docs.unwrangle.com/homedepot-product-data-api/ — bulk pricing, feature lists, structured product data
- **Standards:** REST/JSON; API key authentication
- **Notes:** Live retailer price lookup for material items in the DIY planner's bill of materials would require a third-party data intermediary unless Home Depot introduces a first-party API. Traject's Backyard API (Home Depot + Lowe's unified) is the most practical integration path.

---

### Lowe's Developer Portal

- **Description:** Lowe's offers a developer portal for business partners, including a Product Catalog API component (Nextra platform).
- **API Documentation:** https://developer.lowes.com/portal/business-components/product-catalog/
- **Standards:** Not publicly detailed; likely REST/JSON for business partners
- **Authentication:** Partner programme access required (not open public API)
- **Notes:** Lowe's developer API appears to be gated behind a business partner programme rather than a fully open public API. Access to live pricing and product availability data would require a formal partnership agreement.

---

### OpenProject (Open-Source Reference)

- **Description:** Open-source project management software with a fully documented REST API (OpenAPI spec), covering work packages, projects, budgets, time tracking, and user management.
- **API Documentation:** https://www.openproject.org/docs/api/
- **OpenAPI Spec:** https://github.com/opf/openproject/blob/dev/docs/api/apiv3/openapi-spec.yml
- **Standards:** REST/JSON; OpenAPI 3.x; hypermedia (HAL) links
- **Authentication:** OAuth 2.0, API Key
- **SDKs/Libraries:** Community-maintained Ruby, Python, and JavaScript clients
- **Notes:** OpenProject's openly published OpenAPI spec is an excellent reference for how to model work packages, projects, cost entries, and time logs in a REST API. Its data model (WorkPackage, Project, Budget, TimeEntry) maps directly to a DIY planner's core domain objects. Apache 2.0 licensed.

---

## Notes

**Emerging standards to watch:**
- **Model Context Protocol (MCP):** Anthropic's open protocol for connecting AI models to tools and data sources. An AI-native DIY planner could expose an MCP server allowing AI assistants to read project state, generate task lists, and update material quantities programmatically. Spec at: https://modelcontextprotocol.io/
- **ElectricSQL / local-first sync:** No formal standard yet, but CRDT-based local-first sync is rapidly converging on SQLite + Postgres replication patterns. The Local-First Software community (https://localfirstweb.dev/) is the best reference for emerging conventions.
- **IFC JSON (ifcJSON):** buildingSMART's initiative to serialise IFC data as JSON (https://github.com/buildingSMART/ifcJSON) rather than STEP files, making BIM data accessible to web/mobile developers without specialised parsers. Worth tracking for future material and room data model alignment.
- **RSMeans cost data:** The de-facto industry standard for construction cost estimation in the US (published by Gordian). Not an open standard; licensed data. Any cost benchmark feature in the DIY planner would either need an RSMeans licence, a partnership with a provider like Remodelum, or an independently crowd-sourced cost dataset.
