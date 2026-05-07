# Standards & API Reference

> Project: Expert Network Platform · Generated: 2026-05-07

## Industry Standards & Specifications

### ISO Standards

- **ISO 30401:2018 — Knowledge Management Systems — Requirements**
  https://www.iso.org/standard/68683.html
  Defines requirements for establishing, implementing, and maintaining a knowledge management system. Directly relevant to how an expert network captures, organises, distributes, and applies expert knowledge across an organisation.

- **ISO 27001 — Information Security Management Systems**
  https://www.iso.org/standard/27001
  Information security certification expected by enterprise buyers handling sensitive research conversations and expert data. A baseline requirement for expert network platforms serving financial institutions and consulting firms.

- **ISO/IEC 25012 — Data Quality Model**
  https://www.iso.org/standard/35749.html
  Defines core data quality characteristics (accuracy, completeness, consistency) applicable to expert profile data, transcript data, and engagement records that must maintain high fidelity.

- **ISO 8000 — Data Quality — Master Data**
  https://www.iso.org/standard/60805.html
  Framework for managing, verifying, and exchanging high-quality master data. Relevant to expert profile master data management, deduplication, and cross-platform interoperability.

### W3C & IETF Standards

- **RDF (Resource Description Framework) — W3C**
  https://www.w3.org/RDF/
  Open standard for describing concepts and resources with semantic meaning. Foundation for building expert knowledge graphs that model relationships between experts, skills, industries, and research topics.

- **SKOS (Simple Knowledge Organization System) — W3C**
  https://www.w3.org/2004/02/skos/
  Vocabulary for knowledge organization systems such as taxonomies and thesauri. Applicable to structuring industry and skills taxonomies used for expert classification and matching.

- **OWL (Web Ontology Language) — W3C**
  https://www.w3.org/OWL/
  Language for defining structured ontologies enabling richer integration and interoperability. Useful for modelling the expert domain ontology (expertise areas, industry verticals, seniority levels).

- **JSON-LD — W3C**
  https://www.w3.org/TR/json-ld11/
  JSON-based method for encoding Linked Data. Relevant for structuring expert profile data with schema.org vocabularies (Person, Organization, ProfilePage) for interoperability.

- **RFC 6749 / RFC 6750 — OAuth 2.0 Authorization Framework (IETF)**
  https://datatracker.ietf.org/doc/html/rfc6749
  Industry standard for authorizing third-party access to user data without sharing credentials. Essential for expert and client authentication, API access control, and third-party integrations.

- **RFC 4791 — CalDAV (Calendaring Extensions to WebDAV)**
  https://datatracker.ietf.org/doc/html/rfc4791
  Standard for accessing and managing calendar data with meeting scheduling capability. Relevant for expert call scheduling and calendar integration features.

- **RFC 6638 — Scheduling Extensions to CalDAV**
  https://datatracker.ietf.org/doc/html/rfc6638
  Extensions enabling scheduling of iCalendar-based components between users. Supports automated scheduling workflows for expert consultations.

- **WebRTC — W3C / IETF**
  https://webrtc.org/
  Open standard for real-time peer-to-peer audio and video communication in web browsers. Standardized by W3C and IETF, supported by Apple, Google, Microsoft, and Mozilla. Foundation for built-in expert video consultation without third-party plugins.

### Data Model & API Specifications

- **Schema.org — Person, Organization, ProfilePage Types**
  https://schema.org/Person
  Structured data vocabulary for describing people, organisations, and profile pages. Provides a standard data model for expert profiles including name, job title, affiliations, credentials, and social/professional links.

- **OpenAPI Specification (OAS) 3.1**
  https://spec.openapis.org/oas/latest.html
  Standard for describing RESTful APIs in a machine-readable format. The primary specification for documenting the platform's public API endpoints for expert sourcing, scheduling, and content retrieval.

- **GraphQL Specification**
  https://spec.graphql.org/
  Query language and runtime for APIs enabling clients to request exactly the data they need. Well-suited for complex expert profile queries combining skills, availability, industry coverage, and engagement history.

- **JSON Schema**
  https://json-schema.org/
  Vocabulary for annotating and validating JSON documents. Used for validating API request/response payloads, expert profile data, and configuration objects.

- **ESCO (European Skills, Competences, Qualifications and Occupations)**
  https://esco.ec.europa.eu/en/classification
  EU classification of 3,000 occupations and 13,000 skills with a public API. Provides a standardized skills taxonomy for expert profile classification and cross-platform interoperability.

- **O*NET (Occupational Information Network)**
  https://www.onetonline.org/
  US Department of Labor occupational information system. Complementary to ESCO for classifying expert occupational profiles and skills in North American markets.

- **HR-Open Standards**
  https://www.hropenstandards.org/
  XML and JSON specifications for human resource data interchange. Relevant for expert profile data portability, talent marketplace interoperability, and integration with enterprise HR systems.

### Security & Authentication Standards

- **OpenID Connect Core 1.0**
  https://openid.net/specs/openid-connect-core-1_0.html
  Authentication layer built on OAuth 2.0 providing standardised identity tokens. Essential for SSO integration with enterprise clients and secure expert identity verification.

- **OWASP Application Security Verification Standard (ASVS)**
  https://owasp.org/www-project-application-security-verification-standard/
  Comprehensive security testing framework for web applications. Adopted by Google, Microsoft, Salesforce, and Slack as their security baseline. Required for marketplace platform security assessment.

- **OWASP Top 10**
  https://owasp.org/www-project-top-ten/
  Standard awareness document for the most critical web application security risks. Marketplace partners (e.g., Atlassian Marketplace) are advised to review and mitigate these risks.

- **SEC Regulation FD (Fair Disclosure)**
  https://www.sec.gov/rules/final/33-7881.htm
  US regulation prohibiting selective disclosure of material non-public information. Expert networks must enforce compliance guardrails ensuring MNPI is not exchanged during consultations.

- **Section 204A of the Advisers Act — MNPI Policies**
  https://www.sec.gov/about/offices/ocie/informationbarriers.pdf
  Requires investment advisers to implement written policies preventing misuse of material non-public information. Defines compliance record-keeping requirements for expert network engagements including NDAs, conflict screening, and audit trails.

- **CFA Institute Research Objectivity Standards**
  https://www.cfainstitute.org/ethics-standards
  Govern expert network usage in investment research contexts. Compliance with these standards is a buyer requirement for investment-focused expert network clients.

- **GDPR (General Data Protection Regulation)**
  https://gdpr.eu/
  EU regulation governing expert profile data, consent for contact, and data retention. Requires explicit consent mechanisms and data portability provisions for expert personal data.

- **CCPA (California Consumer Privacy Act)**
  https://oag.ca.gov/privacy/ccpa
  California privacy regulation applicable to expert data collected from California residents. Requires disclosure of data practices and right-to-delete capabilities.

- **NIST Cybersecurity Framework**
  https://www.nist.gov/cyberframework
  Framework for managing cybersecurity risk. Provides structure for security controls around sensitive expert network data and research content.

### MCP Server Specifications

- **Model Context Protocol (MCP) — Anthropic**
  https://modelcontextprotocol.io/
  Open standard for connecting AI agents to external tools, databases, and APIs. Crossed 97 million installs in March 2026 and is the industry standard for AI-native integrations. Directly relevant: Guidepoint and Third Bridge have both launched MCP servers to make expert transcript libraries queryable by AI agents (Claude, ChatGPT).

- **MCP 2026 Roadmap**
  https://blog.modelcontextprotocol.io/posts/2026-mcp-roadmap/
  Current roadmap addressing enterprise requirements: audit trails, SSO-integrated authentication, gateway behaviour, and configuration portability. These capabilities align with expert network compliance and governance requirements.

## Similar Products — Developer Documentation & APIs

### AlphaSense (incl. Tegus)
- **Description:** Market intelligence and search platform unifying expert transcripts, broker research, company filings, and financial data across 500+ million documents. Tegus Expert Transcript Library includes 200,000+ transcripts covering 25,000+ companies.
- **API Documentation:** https://developer.alpha-sense.com/api/getting-started
- **SDKs/Libraries:** Expert Transcripts API delivering Tegus and AlphaSense transcripts in JSON format
- **Developer Guide:** https://developer.alpha-sense.com/
- **Standards:** REST/JSON API
- **Authentication:** API keys, client credentials, bearer tokens (OAuth 2.0)

### Third Bridge
- **Description:** Analyst-led expert network with 100,000+ interview transcripts covering 65,000+ companies. First expert network to adopt MCP-native architecture for AI distribution.
- **API Documentation:** No traditional REST API documented
- **SDKs/Libraries:** MCP server for native AI connectivity
- **Developer Guide:** https://www.thirdbridge.com/en-us/data-solutions/mcp
- **Standards:** Model Context Protocol (MCP), compatible with leading LLMs (ChatGPT, Claude)
- **Authentication:** Enterprise SSO; MCP authentication per specification

### Guidepoint
- **Description:** Expert network with 1.75 million+ advisors and Guidepoint360 AI-powered research platform. Library of 100,000+ compliance-reviewed expert interview transcripts with 5,000+ added monthly.
- **API Documentation:** No traditional REST API documented
- **SDKs/Libraries:** MCP server for Claude integration
- **Developer Guide:** Guidepoint360 platform documentation (client access)
- **Standards:** Model Context Protocol (MCP) via Anthropic's Claude
- **Authentication:** Enterprise SSO; Guidepoint360 platform authentication

### GLG (Gerson Lehrman Group)
- **Description:** Pioneer expert network with 900,000+ vetted professionals across 150+ countries. Reimagined myGLG platform with AI-powered Strategic Research Agent, Flexible Insight Gathering, and Precision Synthesis Tooling.
- **API Documentation:** No publicly documented API
- **SDKs/Libraries:** None publicly available
- **Developer Guide:** None publicly available
- **Standards:** Proprietary platform
- **Authentication:** Enterprise SSO; proprietary platform authentication

### AlphaSights
- **Description:** Expert network with proprietary AlphaGraph knowledge graph mapping 25+ million expert-to-company relationships. Features AlphaGPT generative AI and AlphaNow research library.
- **API Documentation:** No publicly documented API
- **SDKs/Libraries:** None publicly available
- **Developer Guide:** None publicly available
- **Standards:** Proprietary platform
- **Authentication:** Proprietary platform authentication

### Catalant
- **Description:** Enterprise consulting marketplace connecting companies with 100,000+ independent consultants. Used by 30%+ of Fortune 100 for agile workforce augmentation.
- **API Documentation:** No publicly documented API
- **SDKs/Libraries:** None publicly available
- **Developer Guide:** None publicly available
- **Standards:** Proprietary platform with enterprise workflow integration
- **Authentication:** Enterprise SSO; proprietary platform authentication

### NewtonX
- **Description:** AI-powered expert network using a proprietary Knowledge Graph to source from 1.1 billion professionals across 140 industries. Hub platform centralizes recruitment, fieldwork, and analysis.
- **API Documentation:** No publicly documented API
- **SDKs/Libraries:** None publicly available
- **Developer Guide:** None publicly available
- **Standards:** Proprietary AI/ML platform
- **Authentication:** Proprietary platform authentication

### Inex One
- **Description:** Expert network aggregator combining 25+ networks and survey firms in one platform. Transparent pay-as-you-go pricing with AI-powered transcription and insight aggregation.
- **API Documentation:** API integration demonstrated (Enquire AI integrated into Uzabase SPEEDA) but no public developer portal
- **SDKs/Libraries:** None publicly available
- **Developer Guide:** None publicly available
- **Standards:** Aggregation layer connecting multiple expert network backends
- **Authentication:** Platform authentication; details not publicly documented

### ESCO API (Skills Taxonomy)
- **Description:** European Commission's classification of 3,000 occupations and 13,000 skills. Free, public API for accessing the full ESCO classification for skills taxonomy integration.
- **API Documentation:** https://esco.ec.europa.eu/en/use-esco/use-esco-services-api
- **SDKs/Libraries:** RESTful web service; no official SDK but open for integration
- **Developer Guide:** https://esco.ec.europa.eu/en/about-esco/escopedia/escopedia/esco-api
- **Standards:** REST/JSON, SKOS vocabulary, linked data
- **Authentication:** Open access (no authentication required)

### Stripe Connect (Marketplace Payments)
- **Description:** Payment infrastructure for platforms and marketplaces managing payments between multiple parties. Used by Shopify, DoorDash, and similar two-sided marketplaces for expert payment processing.
- **API Documentation:** https://docs.stripe.com/connect
- **SDKs/Libraries:** Official SDKs for JavaScript, Python, Ruby, Go, Java, .NET, PHP
- **Developer Guide:** https://docs.stripe.com/connect/how-connect-works
- **Standards:** REST/JSON API, PCI DSS compliant, OpenAPI specification
- **Authentication:** API keys, OAuth 2.0 for connected accounts

## Notes

- **API gap in the expert network industry**: The majority of expert network platforms (GLG, AlphaSights, Guidepoint, Catalant, NewtonX, proSapient, CleverX) do not offer publicly documented developer APIs. AlphaSense is the notable exception with a comprehensive developer portal. This represents a significant opportunity for an open-source platform to differentiate through API-first design.

- **MCP as emerging integration standard**: Both Guidepoint and Third Bridge launched MCP servers in May 2026, signalling that the Model Context Protocol is becoming the preferred integration pathway for expert network content in AI-native research workflows. An open-source platform should adopt MCP as a first-class integration from the outset.

- **Skills taxonomy convergence**: ESCO and O*NET are the two dominant open skills frameworks. ESCO provides a free public API while O*NET provides downloadable datasets. Cross-mapping between these frameworks is supported, enabling a platform to serve both European and North American markets with standardised skills classification.

- **Compliance standards are evolving**: SEC examination staff now explicitly require firms to log and monitor expert network engagements. The three compliance pillars (real-time documentation, MNPI surveillance, cooling-off period) are crystallising into concrete regulatory expectations. A platform built in 2026 should treat compliance as a core architectural concern, not an afterthought.

- **No open-source expert network platform exists**: The entire category is proprietary. Building an open-source, API-first platform with standardised data models (schema.org, ESCO, JSON-LD) and open integration protocols (MCP, OpenAPI, OAuth 2.0) would represent a novel entrant in this space.
